# DRUID — Conformance Checklist

Three levels of DRUID compliance. Each level includes all requirements from previous levels.

## Level 1 — Minimal

A basic DRUID implementation that can generate a working CRUD API from IR.

### Parser
- [ ] Reads `ir.json` and validates `$schema` field (major version check)
- [ ] Validates required fields: `name`, `plural`, `attributes`, `permit`, `operations`
- [ ] Validates relation targets exist in the resources array
- [ ] Performs topological sort (belongs_to dependencies generated first)

### Generator (per resource)
- [ ] Generates Entity with typed attributes from IR `attributes`
- [ ] Generates Record/Model with ORM column definitions
- [ ] Generates Repository with `find`, `create`, `update`, `destroy`, `all`
- [ ] Generates Service with CRUD method stubs delegating to repository
- [ ] Generates Endpoint/Controller with HTTP route dispatch
- [ ] Generates Migration/Schema with table definition and foreign keys

### Architecture
- [ ] Entity has no database dependency (plain typed object)
- [ ] Record is not used outside Repository
- [ ] Service delegates to Repository, Endpoint delegates to Service

### HTTP
- [ ] `GET /<plural>` → 200 + JSON array
- [ ] `GET /<plural>/:id` → 200 + JSON object
- [ ] `POST /<plural>` → 201 + JSON object (or 422 + errors)
- [ ] `PATCH /<plural>/:id` → 200 + JSON object (or 422 + errors)
- [ ] `DELETE /<plural>/:id` → 204
- [ ] `GET /<plural>/:id` for missing ID → 404 + JSON error

---

## Level 2 — Standard

Adds relations, state machines, collection features, defaults, and HATEOAS.

### IR features
- [ ] `relations` (belongs_to) → ORM associations + FK columns
- [ ] `relations` (has_many) → inverse associations on parent
- [ ] `relations` with `required: false` → optional/nullable FK
- [ ] Self-referential relations (e.g., Category.parent → Category)
- [ ] `attributes.default` → Entity field defaults
- [ ] Entity defaults flow through to persistence (Repository.create uses entity attributes)
- [ ] `transitions` → Entity state machine DSL
- [ ] `transitions.from` accepts string or array
- [ ] Auto-generated routes: `POST /<plural>/:id/<event>` for each transition
- [ ] Transition pipeline: find → guard → transition → persist
- [ ] `collection.sort` → sortable fields whitelist
- [ ] `collection.filter` → filterable fields whitelist
- [ ] `collection.search` → full-text search on specified fields
- [ ] `collection.per_page` → default pagination size
- [ ] Paginated response: `{data: [...], meta: {page, per, total}}`
- [ ] `unique` → unique database indexes in migration
- [ ] `validators` → Entity validation rules (at least `presence`)

### Repository
- [ ] `update` merges partial attributes with existing record before validation
- [ ] `paginate` with page, per, sort, filter, search parameters

### HATEOAS
- [ ] Every response includes `_links.self`
- [ ] belongs_to relations → parent resource link
- [ ] has_many relations → child collection link
- [ ] Transition links CONDITIONAL on current state (only show reachable transitions)
- [ ] Transition links include `href` and `method`

### Error handling
- [ ] 404 → `{"error": "Not found", "status": 404}`
- [ ] 422 → `{"errors": {"field": ["message"]}}`
- [ ] 500 → `{"error": "<message>", "status": 500}`

---

## Level 3 — Full

Complete DRUID implementation with round-trip, custom actions, authorization, and observability.

### Round-trip
- [ ] Extractor: running application → ir.json
- [ ] Extracted IR is structurally identical to the input IR (for generated resources)
- [ ] Attributes, defaults, relations, transitions, validators, collection config all round-trip
- [ ] `unique` and `actions` are NOT required to round-trip (input-only — unique indexes are DB-level infrastructure not detectable from entity code; actions require service implementation that can't be introspected)

### Custom actions
- [ ] `actions` in IR → auto-generated routes
- [ ] Member actions: `<METHOD> /<plural>/:id/<name>` → `service.<name>(id, params)`
- [ ] Collection actions: `<METHOD> /<plural>/<name>` → `service.<name>(params)`
- [ ] Action names validated against CRUD and transition names (no collisions)
- [ ] Authorization gate applies to custom actions by name

### Authorization gate
- [ ] Endpoint-level auth config (role-based, checked before loading resource)
- [ ] Accepts flat array format `[:admin, :instructor]`
- [ ] Accepts hash format `{roles: [:admin, :instructor]}`
- [ ] Role read from request header or auth mechanism
- [ ] Missing/unauthorized role → 403 + JSON error
- [ ] Transition and custom action auth checked by event/action name

### User context
- [ ] Middleware/interceptor extracts user role + ID from request
- [ ] Context accessible in Service layer (fiber-safe mechanism)
- [ ] Context auto-resets after each request

### Observability
- [ ] Endpoint dispatch instrumented with resource name + action
- [ ] Transitions instrumented as `transition.<event_name>`
- [ ] Custom actions instrumented as `custom_action.<action_name>`

### Reserved names
- [ ] Generator rejects IR resource names that conflict with language/framework built-ins

---

## How to test conformance

Create a test IR file with the resources below.

Note: Foreign key attributes (e.g., `author_id`) MUST appear explicitly in `attributes` AND in `relations`. The `attributes` entry defines the column type and nullability. The `relations` entry drives ORM association generation. Both are required for a complete IR. `id`, `created_at`, `updated_at` MUST also appear explicitly in `attributes`.

```json
{
  "$schema": "druid/1.2",
  "resources": [
    {
      "name": "Author",
      "plural": "authors",
      "path": "/authors",
      "attributes": {
        "id": {"type": "integer", "nullable": false, "readonly": true},
        "name": {"type": "string", "nullable": false, "readonly": false},
        "email": {"type": "string", "nullable": false, "readonly": false},
        "created_at": {"type": "datetime", "nullable": false, "readonly": true},
        "updated_at": {"type": "datetime", "nullable": false, "readonly": true}
      },
      "permit": ["name", "email"],
      "operations": [
        {"action": "index", "method": "GET", "path": "/authors"},
        {"action": "show", "method": "GET", "path": "/authors/{id}"},
        {"action": "create", "method": "POST", "path": "/authors"},
        {"action": "update", "method": "PATCH", "path": "/authors/{id}"},
        {"action": "destroy", "method": "DELETE", "path": "/authors/{id}"}
      ],
      "validators": [{"type": "presence", "fields": ["name", "email"]}]
    },
    {
      "name": "Post",
      "plural": "posts",
      "path": "/posts",
      "attributes": {
        "id": {"type": "integer", "nullable": false, "readonly": true},
        "author_id": {"type": "integer", "nullable": false, "readonly": false},
        "title": {"type": "string", "nullable": false, "readonly": false},
        "body": {"type": "text", "nullable": true, "readonly": false},
        "slug": {"type": "string", "nullable": false, "readonly": false},
        "status": {"type": "string", "nullable": false, "readonly": false, "default": "draft"},
        "views_count": {"type": "integer", "nullable": false, "readonly": true, "default": 0},
        "created_at": {"type": "datetime", "nullable": false, "readonly": true},
        "updated_at": {"type": "datetime", "nullable": false, "readonly": true}
      },
      "permit": ["author_id", "title", "body", "slug"],
      "operations": [
        {"action": "index", "method": "GET", "path": "/posts"},
        {"action": "show", "method": "GET", "path": "/posts/{id}"},
        {"action": "create", "method": "POST", "path": "/posts"},
        {"action": "update", "method": "PATCH", "path": "/posts/{id}"},
        {"action": "destroy", "method": "DELETE", "path": "/posts/{id}"}
      ],
      "validators": [{"type": "presence", "fields": ["title"]}],
      "relations": {
        "author": {"kind": "belongs_to", "resource": "Author", "required": true}
      },
      "transitions": {
        "field": "status",
        "events": {
          "publish": {"from": "draft", "to": "published"},
          "archive": {"from": ["draft", "published"], "to": "archived"}
        }
      },
      "collection": {
        "sort": ["title", "created_at"],
        "filter": ["status", "author_id"],
        "search": ["title", "body"],
        "per_page": 20
      },
      "unique": ["slug"],
      "actions": [
        {"name": "feature", "method": "PATCH", "on": "member"}
      ]
    }
  ]
}
```

**Level 1:** Generate, run migrations, verify all CRUD endpoints return correct status codes.

**Level 2:** Verify: Post.status defaults to "draft", publish/archive routes exist and work, `_links` include conditional transitions, `GET /posts?sort=title&filter[status]=draft&search=hello&page=1` works, slug has unique index.

**Level 3:** Extract IR, compare with input. Test auth gate with role headers. Test custom action `PATCH /posts/:id/feature`. Verify instrumentation events fire.
