# DRUID

**Domain Resource Unified Intermediate Descriptor**

A specification for AI-native, domain-driven API development.

DRUID defines a language-agnostic format for describing domain resources (IR) and a methodology for turning plain-language domain descriptions into working APIs through behavioral specifications.

## Why DRUID exists

Existing tools describe parts of a system: OpenAPI describes HTTP endpoints, Prisma describes database schema, JSON Schema describes data shapes. None of them describe the **domain** — the resources, their lifecycle, their relations, their constraints, and who can do what with them.

When an AI agent builds an API, it needs all of this in one place. Today it scrapes information from scattered files: ORM models, route configs, policy classes, migration files. If any are missing or stale, the agent hallucinates.

DRUID gives the domain a single, machine-readable representation:

| What | Where today | Where in DRUID |
|------|------------|----------------|
| Resource attributes & types | ORM model files | IR `attributes` |
| API routes & methods | Router config | IR `operations` |
| DB relations | ORM associations | IR `relations` |
| State machines | Scattered across models/services | IR `transitions` |
| Unique constraints | Migration files | IR `unique` |
| Custom endpoints | Hand-wired routes | IR `actions` |
| Authorization rules | Policy files / middleware | Behavioral spec (decision table) |
| Business rules | Service / model code | Behavioral spec (contracts) |
| Lifecycle guards | Callbacks / before_actions | Behavioral spec (pre/postconditions) |

One JSON file for structure. One markdown file per resource for behavior. Round-trip between code and spec.

## Why LLMs and DRUID are a natural fit

LLMs are good at transforming structured descriptions into code. They are bad at inventing structure from vague requirements and at keeping scattered files consistent.

DRUID gives the LLM exactly what it needs at each step:

**Step 1 → 2 (human words → behavioral spec).** The LLM's strongest skill: taking fuzzy natural language and formalizing it. The five-section format (contracts, auth table, lifecycle, scenarios, side effects) channels the output into structured conventions the LLM already understands — Design by Contract, decision tables, Given-When-Then. It doesn't need to guess the format.

**Step 2 → 3 (behavioral spec → IR).** Mechanical extraction. Nouns → resources, attributes → typed fields, "belongs to" → relations, "draft → published" → transitions. No creativity needed — the LLM follows rules.

**Step 3 → 4 (IR → code).** The generator does this. Zero LLM involvement. Deterministic.

**Step 4 → 5 (generated skeleton → business logic).** The LLM reads a behavioral spec and writes code into a known structure. It doesn't decide WHERE to put the code (the layer architecture dictates that). It doesn't decide WHAT to write (the spec dictates that). It only translates spec → code within a clear boundary:

| Spec says | LLM writes in |
|-----------|--------------|
| `invariant: rating in 1..5` | Entity validation |
| `pre(publish): has lessons` | Service guard |
| `admin: ✓, student: ✗` | Endpoint auth config |
| `@on(enroll): notify student` | Service side effect |

Every decision is local. The LLM never needs to hold the whole system in context — one resource, one spec, one file at a time.

**Why this beats "generate everything from a prompt":**

A single prompt like "build me a course platform" forces the LLM to invent structure AND behavior simultaneously, with no checkpoint between them. It hallucinates fields, forgets constraints, creates inconsistent relations.

DRUID splits the problem into verifiable steps. After step 2 you can review specs. After step 3 you can validate IR (it's JSON — schema-checkable). After step 4 you have working CRUD to test. After step 5 you have business logic that traces back to a spec line. Every artifact is auditable.

## REST architectural constraints

Roy Fielding's dissertation (2000) defines REST through six constraints. DRUID IR maps to each of them:

**1. Client-Server.** IR defines the server-side domain. The client consumes it through generated endpoints. Separation is structural — IR never describes UI.

**2. Stateless.** Each IR operation is self-contained: method + path + permitted params. No session state in IR. Every request carries all needed context (auth headers, resource params).

**3. Cacheable.** IR `collection` config defines per-resource caching policy. Implementations generate `Cache-Control` headers from it. The client knows what's cacheable from the response — DRUID makes the server-side policy explicit.

**4. Uniform Interface.** This is where DRUID is most precise. Fielding's uniform interface has four sub-constraints:

- **Resource identification.** Every IR resource has `name`, `plural`, `path`. A resource is identified by `/<plural>/{id}`. This is the URI.

- **Resource manipulation through representations.** IR `attributes` define the representation. `permit` defines which fields are writable. `readonly` marks server-managed fields. The client manipulates the resource by sending a representation (JSON body with permitted fields).

- **Self-descriptive messages.** IR `operations` map each action to an HTTP method + path. `validators` declare constraints. The response carries enough information to understand itself — JSON with typed fields matching IR attributes.

- **Hypermedia as the engine of application state (HATEOAS).** IR `transitions` declare which state changes are possible. IR `relations` declare links to related resources. Implementations generate `_links` in every response: `self`, relation hrefs, and available transition actions (conditional on current state). The client discovers what it can do next from the response, not from out-of-band knowledge.

**5. Layered System.** DRUID's four DDD layers (endpoint → service → entity → repository) are a server-side layered system. Each layer only knows about the layer below it. IR doesn't prescribe client-side layers.

**6. Code on Demand (optional).** Not applicable to JSON APIs. DRUID doesn't address this constraint.

DRUID IR is a formal encoding of REST's uniform interface. `attributes` = representation, `operations` = methods, `relations` = links, `transitions` = hypermedia controls, `collection` = cache policy. The five applicable Fielding constraints are structurally guaranteed by a valid IR file.

## What DRUID is

- **IR Specification** — a JSON format for describing domain resources: attributes, relations, state machines, constraints, custom actions
- **Behavioral Spec format** — a 5-section markdown convention for capturing business rules
- **Flow** — a 5-step methodology: understand → specify → describe → generate → implement

## What DRUID is not

- Not a framework — it's a specification. Frameworks implement DRUID.
- Not a runtime — IR is a design-time artifact (a JSON file), not a protocol.
- Not Rails-specific — the first implementation is a Rails fork, but IR and specs are language-agnostic.

## Documentation

### Specification
- [Overview](spec/overview.md) — philosophy, applicability, system requirements
- [IR Format](spec/ir.md) — the JSON schema for domain resources (`druid/1.2`)
- [Behavioral Specs](spec/behavioral.md) — 5-section format for business rules
- [Flow](spec/flow.md) — the 5-step methodology

### For implementors
- [Implementation Guide](implementors/guide.md) — how to build a DRUID-compatible generator
- [Conformance Checklist](implementors/conformance.md) — what makes an implementation DRUID-compliant
- [Layer Mapping](implementors/layer-mapping.md) — IR concepts → DDD architecture layers
- [Type Mapping](implementors/type-mapping.md) — IR types → language types

### Implementations
- [Rails](implementations/rails/) — DDD fork of Ruby on Rails ([sld0Ant/rails](https://github.com/sld0Ant/rails))

## Quick example

**IR (structure):**
```json
{
  "$schema": "druid/1.2",
  "resources": [{
    "name": "Course",
    "attributes": {
      "title":  { "type": "string", "nullable": false },
      "status": { "type": "string", "default": "draft" }
    },
    "transitions": {
      "field": "status",
      "events": {
        "publish": { "from": "draft", "to": "published" },
        "archive": { "from": ["draft", "published"], "to": "archived" }
      }
    }
  }]
}
```

**Behavioral spec (business rules):**
```markdown
## Contracts
  pre(publish): course has at least 1 lesson
  pre(publish): description is not blank

## Authorization
| Action  | admin | author | guest     |
|---------|-------|--------|-----------|
| create  | ✓     | ✓      | ✗         |
| publish | ✓     | own    | ✗         |
| show    | ✓     | ✓      | published |
```

**Result:** a working API with CRUD, state transitions, authorization, and HATEOAS links — generated from IR, business logic written from spec.

## Round-trip

DRUID IR is bidirectional:

```
ir.json  →  code      (generate)
code     →  ir.json   (extract)
```

For any DRUID-compliant implementation:

```bash
generate(ir.json)     # IR → working application
extract()             # application → ir.json
generate(ir.json)     # IR → application again
```

The two generated applications are structurally identical. This is the **round-trip guarantee**.

### What is preserved in round-trip

| IR field | Generate (IR → code) | Extract (code → IR) |
|----------|---------------------|---------------------|
| `attributes` | Entity typed fields | Read from entity class attribute definitions |
| `default` | Entity `attribute :x, :string, default: "draft"` | Read from entity `_default_attributes` |
| `relations` | Record associations + Entity FK attrs | Read from ORM `reflect_on_all_associations` |
| `transitions` | Entity `transitions` DSL | Read from entity `transitions_config` |
| `operations` | HTTP routes | Reconstructed from plural name (standard CRUD) |
| `validators` | Entity `validates` declarations | Read from entity `validators` |
| `collection` | Endpoint sort/filter/search/per_page config | Read from endpoint class attributes |
| `permit` | Endpoint permitted_params | Read from endpoint class attribute |

Fields NOT round-tripped (input-only, used during generation):
- `unique` — DB indexes are infrastructure, not readable from domain layer
- `actions` — custom actions require service implementation, not introspectable

This means: `extract → generate → extract` always produces the same IR. Manual changes to the code (adding a field to entity, adding a relation to record) are captured on the next extract.

### What round-trip enables

**Living documentation.** IR is never stale. Extract it from the running code at any point — it reflects the current state of the domain. If someone added a field by hand, the next extract picks it up.

**Design-first or code-first — both work:**

```
Design-first:  write ir.json → generate code → add logic → extract ir.json (verify)
Code-first:    write code manually → extract ir.json → use IR for everything else
```

**Cross-stack portability.** Extract IR from one implementation, generate another. The domain structure transfers exactly. Business logic (behavioral specs) transfers as-is — it's markdown, not code.

```
Rails app  → extract → ir.json → generate → NestJS app
                         ↓
                    same ir.json → generate → FastAPI app
                         ↓
                    same ir.json → generate → Go app
```

The behavioral specs work across all implementations — they describe WHAT should happen, not HOW in a specific language.

**Schema evolution with auditable diff.**

```
v1/ir.json  →  v2/ir.json  →  JSON diff = exactly what changed in the domain
```

Adding a field, renaming a relation, introducing a new transition — visible as structured diff before any code changes. Review the IR diff, approve, regenerate.

## Emitting specifications from IR

IR is a superset of what most API specification formats need. A DRUID implementation can emit multiple output formats from a single `ir.json`:

```
                    ┌──────────────────┐
                    │     ir.json      │
                    │   (DRUID IR)     │
                    └────────┬─────────┘
                             │
          ┌──────────────────┼──────────────────────┐
          │                  │                      │
          ▼                  ▼                      ▼
     Emit: code         Emit: docs            Emit: clients
          │                  │                      │
          ▼                  ▼                      ▼
   Entity, Service,    OpenAPI 3.x            TypeScript types
   Repository,         JSON Schema            API client (fetch)
   Endpoint,           AsyncAPI               Python SDK
   Migration           Markdown docs          Swift models
```

### What each emitter reads from IR

**OpenAPI emitter:**

| IR field | OpenAPI output |
|----------|---------------|
| `name`, `plural`, `path` | Path objects (`/courses`, `/courses/{id}`) |
| `attributes` + `type` | Schema component with typed properties |
| `nullable` | `nullable: true` on property |
| `readonly` | Separate request/response schemas (readonly excluded from request) |
| `permit` | Request body properties |
| `operations` | Path items with HTTP methods |
| `validators` | `required` array, `minLength`, `pattern` etc. |
| `relations` (belongs_to) | `$ref` in schema, FK in properties |
| `transitions` | Additional POST paths (`/courses/{id}/publish`) |
| `actions` | Additional paths with custom methods |
| `collection` | Query parameters (`sort`, `filter[status]`, `search`, `page`, `per_page`) |
| `default` | `default` on schema property |

One IR → complete OpenAPI 3.x spec, no manual writing.

**TypeScript emitter:**

| IR field | TypeScript output |
|----------|------------------|
| `name` | `interface Course { ... }` |
| `attributes` + `type` | Typed properties (`title: string`, `price: number`) |
| `nullable` | `title: string \| null` |
| `readonly` | `readonly id: number` |
| `permit` | `type CourseCreateParams = Pick<Course, 'title' \| 'status'>` |
| `operations` | Typed fetch functions (`getCourse(id): Promise<Course>`) |
| `transitions` | `publishCourse(id): Promise<Course>` |
| `actions` | `markReadNotification(id): Promise<Notification>` |
| `collection` | `listCourses(params: CourseListParams): Promise<PaginatedResponse<Course>>` |
| `relations` | Nested type references |

One IR → full TypeScript SDK with types, API client, and request/response types.

**JSON Schema emitter:**

| IR field | JSON Schema output |
|----------|-------------------|
| `attributes` | `properties` with `type` mapping |
| `nullable` | Combined type: `{"type": ["string", "null"]}` |
| `default` | `default` keyword |
| `validators` (presence) | `required` array |
| `unique` | Not expressible in JSON Schema (DB-level concern) |

**Other possible emitters:**
- **Markdown docs** — human-readable API reference from IR
- **Postman collection** — importable request set for every operation + transition
- **Database schema (SQL DDL)** — `CREATE TABLE` from attributes + relations + unique
- **GraphQL schema** — types + queries + mutations from IR (experimental)
- **AsyncAPI** — if events are added to IR in a future version
- **Test scaffolds** — one integration test per operation, per resource
- **Seed data** — fixture generators from attribute types and constraints

### The key insight

IR is richer than any single output format. OpenAPI can't express state machines. JSON Schema can't express operations. TypeScript types can't express authorization. DRUID IR holds all of these — and each emitter takes what it needs.

```
ir.json (everything) → OpenAPI (HTTP surface)
                     → TypeScript (type surface)
                     → JSON Schema (data surface)
                     → SQL DDL (storage surface)
                     → Test suite (behavior surface)
```

This is why IR is not "yet another OpenAPI alternative" — it's a layer above, from which OpenAPI (and others) are derived.

## Scaling and workflow

### Single developer

```
describe domain → write ir.json → generate → add logic
```

One person, one file, full API in minutes.

### Team

```
Domain expert    → behavioral specs (contracts, auth tables, scenarios)
Backend engineer → ir.json from specs → generate → implement logic
Frontend engineer → reads ir.json → generates TypeScript types + API client
QA               → reads behavioral specs → writes test scenarios
```

IR is the contract between all roles. Behavioral specs are readable by non-engineers.

### Multi-service

Each service has its own `ir.json`. Cross-service contracts are explicit:

```
service-a/ir.json  — defines User, Organization
service-b/ir.json  — defines Order, Payment (references User by ID)
```

Services share the IR format. No shared code, no shared runtime. Just a shared schema language.

### Migration from existing systems

1. Build a DRUID extractor for your current stack (read ORM models → produce ir.json)
2. Extract IR from existing code
3. Write behavioral specs for existing business rules
4. Generate a new implementation from IR + specs
5. Compare behavior — IR guarantees structural equivalence

## License

MIT
