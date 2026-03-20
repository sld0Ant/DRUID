# DRUID — Implementation Guide

How to build a DRUID-compatible implementation for your framework/language.

## What an implementation consists of

Three components:

| Component | Input | Output | Required |
|-----------|-------|--------|----------|
| **Parser** | ir.json | Validated in-memory structure | Yes |
| **Generator** | Parsed IR | Application code (6 artifacts per resource) | Yes |
| **Extractor** | Running application | ir.json | For round-trip (Level 3) |

## Parser

Read `ir.json`, validate against the [IR specification](../spec/ir.md).

**Must validate:**
- `$schema` starts with `"druid/"`, major version matches
- `resources` is a non-empty array
- Each resource has required fields: `name`, `plural`, `attributes`, `permit`, `operations`
- Relation targets exist in the resources array
- Transition `field` exists in resource attributes
- Action names don't collide with CRUD operations or transition event names

**Must handle:**
- Topological sort: if B belongs_to A, generate A before B
- Optional fields: `relations`, `collection`, `transitions`, `unique`, `actions` — absent = empty
- `default` detection: `key?("default")` not `default == nil`

## Generator

For each IR resource, generate 6 artifacts across 4 DDD layers:

```
IR Resource
    │
    ├── Domain Layer
    │   └── Entity — typed attributes, validations, state machine, defaults. No DB access.
    │
    ├── Infrastructure Layer
    │   ├── Record / Model — ORM mapping, associations. DB access only.
    │   ├── Repository — wraps Record, returns Entity. CRUD + pagination.
    │   └── Migration / Schema — CREATE TABLE, indexes, foreign keys.
    │
    ├── Application Layer
    │   └── Service — orchestration, transition guards, domain operations.
    │
    └── UI Layer
        └── Endpoint / Controller — HTTP routes, param filtering, response formatting.
```

### Entity

From IR:
- `attributes` → typed fields with `type` mapping (see [type-mapping.md](type-mapping.md))
- `attributes[x].default` → field default value
- `attributes[x].readonly` → field is not settable by user
- `transitions` → state machine DSL declaration
- `validators` → validation rules

**Rules:**
- Entity MUST NOT import, reference, or depend on any database library
- Entity is a plain object with typed attributes, validations, and state machine
- Entity has `persisted?` (true when id present)
- Entity has `can_transition?(event)` and `transition!(event)` methods
- `transition!` changes the state field in memory, does not persist
- `from:` in transitions accepts a single value or an array

### Record / Model

From IR:
- `attributes` → database columns
- `relations` (belongs_to) → ORM associations with foreign keys
- `relations` (has_many) → inverse associations on parent records

**Rules:**
- Record MUST NOT be used outside Repository
- Record class name uses a suffix (e.g., `CourseRecord`) to distinguish from Entity
- belongs_to with `required: false` → optional association (nullable FK)
- Self-referential relations (e.g., Category.parent → Category) must use explicit FK and class name

### Repository

Wraps Record, returns Entity.

**Must implement:**
- `all` → array of entities
- `find(id)` → entity (raise not-found if missing)
- `create(attributes)` → entity (with validation)
- `update(id, attributes)` → entity (merge with existing record before validation)
- `destroy(id)` → void
- `paginate(page:, per:, sort:, filter:, search:, search_fields:)` → `{data: [...], meta: {page, per, total}}`

**Critical: merge-update.** `update` MUST load the existing record, merge the provided attributes on top, build a full entity, validate the full entity, then persist only the changed fields. This prevents partial-entity validation failures during transitions and PATCH operations.

**Critical: defaults flow.** `create` MUST build the entity first (applying attribute defaults), then persist entity's attributes to the database. This ensures IR defaults flow through to persistence.

### Migration / Schema

From IR:
- `attributes` → columns with appropriate types
- `relations` (belongs_to) → foreign key columns + references
- `unique` → unique indexes

### Service

From IR: minimal — just CRUD method stubs and the repository reference.

**Must implement:**
- `list_all` → delegates to repository (paginate or all)
- `find(id)` → delegates to repository
- `create(attributes)` → delegates to repository
- `update(id, attributes)` → delegates to repository
- `destroy(id)` → delegates to repository
- `perform_transition(id, transition_name)` → public method, called by endpoint

**Transition pipeline (built into base class):**

```
perform_transition(id, name)
  1. entity = repository.find(id)
  2. guard_transition(entity, name)      ← hook for preconditions
  3. return entity if errors
  4. entity.transition!(name)            ← state machine check + state change
  5. return entity if errors
  6. repository.update(id, {state_field => new_value})
```

`guard_transition(entity, transition_name)` — private hook. Override in subclass to add preconditions from behavioral spec. Add errors to `entity.errors` to block the transition.

**Custom actions:** Service must have a public method matching each IR action name. Member actions receive `(id, params)`. Collection actions receive `(params)`.

### Endpoint / Controller

From IR:
- `operations` → HTTP routes (method + path)
- `permit` → allowed request body fields
- `collection` → sort/filter/search/per_page config
- `relations` → HATEOAS link generation
- `transitions` → auto-routed POST endpoints
- `actions` → auto-routed custom endpoints
- authorization config (from behavioral spec, added in Step 5)

**Must implement:**
- Declarative resource config (one-line setup per resource)
- CRUD dispatch to service methods
- Transition dispatch: `POST /<plural>/:id/<event>` → `service.perform_transition(id, event)`
- Custom action dispatch: `<METHOD> /<plural>/:id/<name>` (member) or `<METHOD> /<plural>/<name>` (collection) → `service.<name>(...)`. The HTTP method comes from the action's `method` field in IR.
- JSON error responses: 404 (not found), 422 (validation errors), 403 (forbidden), 500 (unexpected)

## HATEOAS

Every JSON response for a resource MUST include `_links`:

```json
{
  "id": 1,
  "title": "Ruby 101",
  "status": "draft",
  "_links": {
    "self": "/courses/1",
    "instructor": "/instructors/3",
    "publish": { "href": "/courses/1/publish", "method": "POST" },
    "archive": { "href": "/courses/1/archive", "method": "POST" }
  }
}
```

- `self` — always present, plain string (URI)
- Relation links — from `relations` config (belongs_to → link to parent), plain string (URI)
- Transition links — object with `href` + `method`. CONDITIONAL on current state: only include transitions whose `from` matches the current state value. This is HATEOAS: the client discovers available actions from the response.

Link format convention: navigation links (self, relations) are plain URI strings. Action links (transitions, custom actions) are objects with `href` and `method` — because the client needs to know the HTTP method to invoke them.

## Authorization gate

Two-level authorization:

**Level 1 — Gate (endpoint).** Checked BEFORE loading the resource. Based on user role only.

```
authorize config: { create: [:admin, :instructor], destroy: [:admin] }
```

Role comes from request header or auth middleware. If role not in allowed list → 403.
Accept both flat array format `[:admin]` and hash format `{roles: [:admin]}`.

**Level 2 — Domain (service).** Checked AFTER loading the resource. Based on ownership, state, relations.

Implemented manually in service methods using user context (role + user_id from request).

## Error handling

| HTTP Status | When | Response body |
|------------|------|--------------|
| 200 | Successful read/update/transition | Resource JSON with `_links` |
| 201 | Successful create | Resource JSON with `_links` |
| 204 | Successful destroy | Empty |
| 403 | Authorization gate rejection | `{"error": "Forbidden", "status": 403}` |
| 404 | Resource not found | `{"error": "Not found", "status": 404}` |
| 422 | Validation failure / guard rejection | `{"errors": {"field": ["message"]}}` |
| 500 | Unexpected error | `{"error": "<message>", "status": 500}` |

## Instrumentation

Implementations SHOULD instrument endpoint dispatch for observability:

```
endpoint.process {endpoint: "CoursesEndpoint", action: "create"}
endpoint.process {endpoint: "CoursesEndpoint", action: "transition.publish"}
endpoint.process {endpoint: "CoursesEndpoint", action: "custom_action.feature"}
```

## User context

Implementations must provide a mechanism for services to access the current user's role and ID. Options:

| Mechanism | Fiber-safe | Framework example |
|-----------|-----------|-------------------|
| CurrentAttributes | Yes | Rails (ActiveSupport::CurrentAttributes) |
| Request-scoped DI | Yes | NestJS (REQUEST scope) |
| Context object passed through | Yes | Go (context.Context), Python (contextvars) |
| Thread-local | No (breaks with fibers) | Avoid in modern runtimes |

The mechanism is set by middleware that reads auth headers / tokens and populates the context before each request.
