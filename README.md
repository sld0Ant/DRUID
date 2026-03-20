# DRUID

**Domain Resource Unified Intermediate Descriptor**

A specification for AI-native, domain-driven API development.

DRUID defines a language-agnostic format for describing domain resources (IR) and a methodology for turning plain-language domain descriptions into working APIs through behavioral specifications.

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
ir.json  →  code (generate)
code     →  ir.json (extract)
```

For any DRUID-compliant implementation:

```bash
generate(ir.json)     # IR → working application
extract()             # application → ir.json
generate(ir.json)     # IR → application again
```

The two generated applications are structurally identical. This is the **round-trip guarantee**.

### Why round-trip matters

**Living documentation.** IR is never stale. Extract it from the running code at any point — it always reflects the current state.

**Cross-stack portability.** Extract IR from a Rails app, generate a NestJS app from the same IR. The domain structure transfers. Business logic (behavioral specs) transfers as-is — it's markdown, not code.

```
Rails app → extract → ir.json → generate → NestJS app
                        ↓
                   same ir.json → generate → FastAPI app
```

**External system integration.** Any system that reads JSON can consume IR:
- API gateways read IR to configure routes and rate limits
- Frontend generators read IR to produce TypeScript types and API clients
- Documentation tools read IR to emit OpenAPI, AsyncAPI, or custom docs
- Testing tools read IR to generate integration test suites
- Monitoring systems read IR to set up per-resource dashboards

No adapter code. No manual sync. Read `ir.json` — you know the full domain structure.

**Schema evolution.** Change the IR, re-generate, diff. The IR is the single source of truth for structure. Adding a field, renaming a relation, introducing a new transition — all visible as a JSON diff before any code changes.

```
v1/ir.json  →  v2/ir.json  →  diff = exactly what changed in the domain
```

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
