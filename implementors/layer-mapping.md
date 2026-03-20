# DRUID — Layer Mapping

How IR fields and behavioral spec sections map to DDD architecture layers.

## Sources

A DRUID resource is built from two sources:

| Source | Contains | Machine-readable |
|--------|----------|-----------------|
| **IR** (ir.json) | Structure: attributes, types, relations, transitions, operations, constraints | Yes (JSON) |
| **Behavioral spec** (docs/behavior/*.md) | Behavior: invariants, preconditions, authorization, scenarios, side effects | No (markdown for human + AI) |

IR drives code **generation** (Step 4). Behavioral specs drive **manual implementation** (Step 5).

## IR fields → layers

| IR field | UI (Endpoint) | Application (Service) | Domain (Entity) | Infrastructure (Repo/Record/Migration) |
|----------|--------------|----------------------|-----------------|---------------------------------------|
| `name` | class naming | class naming | class naming | table naming |
| `plural` | route path | — | — | table name |
| `attributes` | response shape | — | typed fields | columns |
| `attributes.default` | — | — | field default value | — |
| `attributes.readonly` | excluded from permit | — | not settable by user | — |
| `attributes.nullable` | — | — | validation | NOT NULL constraint |
| `permit` | writable fields filter | — | — | — |
| `operations` | HTTP routes (method + path) | method stubs | — | — |
| `validators` | — | — | validation rules | — |
| `relations` (belongs_to) | HATEOAS parent links | — | FK attributes | FK column + association |
| `relations` (has_many) | HATEOAS child links | — | — | inverse association |
| `collection.sort` | sort param whitelist | passed to repo | — | ORDER BY |
| `collection.filter` | filter param whitelist | passed to repo | — | WHERE |
| `collection.search` | search param | passed to repo | — | LIKE query |
| `collection.per_page` | default page size | passed to repo | — | LIMIT |
| `transitions` | auto POST routes + HATEOAS links | perform_transition method | state machine DSL | — |
| `unique` | — | — | — | unique indexes |
| `actions` | custom routes | method stubs | — | — |

## Behavioral spec sections → layers

| Spec section | UI (Endpoint) | Application (Service) | Domain (Entity) | Infrastructure |
|-------------|--------------|----------------------|-----------------|---------------|
| Contracts: invariants | — | — | validates declarations | — |
| Contracts: pre/post | — | guard methods (preconditions) | — | — |
| Authorization: ✓/✗ | authorize config (gate) | — | — | — |
| Authorization: own/condition | — | ownership/state checks | — | — |
| Lifecycle: transitions | (already from IR) | guard_transition hook | (already from IR) | — |
| Scenarios | — | — | — | — (test reference only) |
| Side Effects: @on | — | post-operation logic | — | — |

## Visual: one resource through 4 layers

```
┌─────────────────────────────────────────────────────────────┐
│ IR Resource: Course                                          │
│   attributes: title, slug, status, price_cents, instructor_id│
│   transitions: publish (draft→published)                     │
│   relations: instructor (belongs_to)                         │
│   collection: sort [title], filter [status], per_page 20    │
│   actions: [feature (POST, member)]                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
    ┌──────────────────┼──────────────────────────┐
    │                  │                          │
    ▼                  ▼                          ▼
┌─────────┐    ┌──────────────┐          ┌──────────────┐
│ Entity  │    │ Behavioral   │          │ IR only      │
│ + Spec  │    │ Spec only    │          │              │
├─────────┤    ├──────────────┤          ├──────────────┤
│ Domain  │    │ Application  │          │ UI + Infra   │
│ Layer   │    │ Layer        │          │ Layers       │
└────┬────┘    └──────┬───────┘          └──────┬───────┘
     │                │                         │
     ▼                ▼                         ▼

  Entity:           Service:                Endpoint:
  - attribute :title  - guard_transition     - resource :course
  - attribute :status   (publish guard)      - permit: [:title]
  - validates :title  - create with guards   - sort/filter/search
  - transitions DSL   - side effects         - authorize config
  - default: "draft"  - ownership checks     - auto routes
                                             - HATEOAS _links
                                            
                                            Record:
                                            - belongs_to :instructor
                                            - table: courses
                                            
                                            Repository:
                                            - find, create, update
                                            - merge-update
                                            - paginate
                                            
                                            Migration:
                                            - create_table :courses
                                            - add_index :slug, unique
```

## Layer rules

| Rule | Why |
|------|-----|
| Entity NEVER imports database libraries | Domain objects are pure — testable without DB |
| Record NEVER appears outside Repository | Infrastructure is hidden from domain and application layers |
| Service is the ONLY place with business logic | Single responsibility — orchestration lives here |
| Endpoint NEVER contains logic | HTTP adapter only — routes params to service, formats response |
| Repository.update MERGES with existing record | Prevents partial-entity validation failures on PATCH/transitions |
| Repository.create uses Entity attributes (not raw input) | Entity defaults must flow through to persistence |
