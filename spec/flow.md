# DRUID — Flow (LLM Agent Instruction)

This document is both a specification and a prompt. Paste it as system context for an AI agent that builds APIs using DRUID. For full context, also provide [behavioral.md](behavioral.md) (spec format) and [ir.md](ir.md) (IR schema). When pasting as a prompt, inline the relevant sections or provide all three files.

## The Flow

```
User's words → Behavioral Spec → IR (ir.json) → Generated skeleton → Business logic
     1               2               3                 4                    5
```

Always go in this order. Never skip to code. Never generate IR without a spec first.

```
  ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌───────────┐    ┌──────────┐
  │ 1.Under-│    │ 2.Behav- │    │ 3.Derive│    │ 4.Generate│    │ 5.Add    │
  │ stand   │───▶│ ioral    │───▶│ IR      │───▶│ skeleton  │───▶│ business │
  │ domain  │    │ specs    │    │ (JSON)  │    │ (tooling) │    │ logic    │
  └─────────┘    └──────────┘    └─────────┘    └───────────┘    └──────────┘
   human+AI        AI+human        AI            generator         AI+human
                   review          verifiable    deterministic     review
```

If any step reveals gaps, go back to the previous step. Not forward.

---

## Step 1 — Understand the Domain

**Who:** Human describes, AI asks questions.
**Input:** Natural language description of what the system does.
**Output:** List of resources, operations, rules, roles.

Extract:
- What are the main things (nouns)? → resources
- What happens to them (verbs)? → operations, transitions
- What rules exist? → constraints, policies
- Who does what? → roles, authorization

Don't ask all questions at once. Start with "What are you building?" and drill down.

**Failure mode:** If the domain is too vague, ask for concrete examples. "Give me a real scenario of a user going through the system."

---

## Step 2 — Write Behavioral Specs

**Who:** AI writes, human reviews.
**Input:** Domain understanding from Step 1.
**Output:** `docs/behavior/<resource>.md` — one file per resource.

Use the 5-section format defined in [behavioral.md](behavioral.md):

1. **Contracts** — invariants, pre/postconditions
2. **Authorization** — role × action decision table
3. **Lifecycle** — state transitions with guards
4. **Scenarios** — concrete edge cases (Given-When-Then)
5. **Side Effects** — consequences beyond the direct response

Skip sections that don't apply. Don't write specs for pure CRUD resources (no transitions, no custom rules, no authorization beyond default).

**Verification:** Human reads specs and confirms they match intent. Every business rule the human described should appear in exactly one spec.

**Failure mode:** If a rule is ambiguous, ask the human. Don't guess — ambiguous specs produce ambiguous code.

---

## Step 3 — Derive IR from Specs

**Who:** AI.
**Input:** Behavioral specs from Step 2.
**Output:** `docs/ir.json` — a single JSON file following the [IR specification](ir.md).

### How specs map to IR

| Spec section | IR field |
|-------------|----------|
| Contracts: invariants | `validators`, `nullable`, attribute `default` |
| Contracts: pre/post (operation names) | informs `operations` (which CRUD actions exist) |
| Authorization: decision table | informs `operations` (if an action has ✗ for all roles, omit it) |
| Lifecycle: transitions | `transitions` block (field, events, from/to) |
| Lifecycle: state field | `attributes` with `default` for initial state |
| Side Effects: custom actions | `actions` (custom endpoints like mark_read) |
| Side Effects: counter fields | `attributes` (e.g., enrollments_count as readonly integer) |
| Scenarios | nothing in IR — drives Step 5 |

### Rules

- `id`, `created_at`, `updated_at` — always present in `attributes`, always `readonly: true`. List them explicitly in IR.
- Foreign keys (`*_id`) come from `relations`. They MUST also appear in `attributes` (as `readonly: false` integer) for completeness, but the `relations` block is what drives ORM association generation
- `permit` = all non-readonly, non-id attributes
- `nullable: false` when spec has `invariant: X is present`
- Type vocabulary: `integer string text float decimal boolean date datetime time`
- If B belongs_to A, A should appear before B in the resources array
- `default` on attributes: use `key?("default")` to distinguish "no default" from "default to null"

### Verification

IR is JSON — validate against the schema. Check:
- Every resource from specs has an IR entry
- Relations point to existing resources
- Transition `field` exists in attributes
- Action names don't collide with CRUD or transition names

**Failure mode:** If a spec rule can't be expressed in IR (e.g., "invariant: X > Y" — cross-field validation), that's fine. IR captures structure only. The rule will be implemented in Step 5.

---

## Step 4 — Generate Skeleton

**Who:** Generator tool (deterministic, no AI).
**Input:** `docs/ir.json`
**Output:** Working CRUD API with all resources.

The generator reads IR and produces per resource:

| Artifact | Layer | Responsibility |
|----------|-------|---------------|
| Entity | Domain | Typed attributes, validations, state machine. No DB. |
| Record / Model | Infrastructure | ORM mapping. DB only. Never used in domain code. |
| Repository | Infrastructure | Wraps Record. Returns Entity. CRUD + pagination. |
| Service | Application | Orchestration. Calls repository. Transition guards hook. |
| Endpoint / Controller | UI | HTTP adapter. Declarative. Routes params to service. No logic. |
| Migration / Schema | Infrastructure | Database table definition. |

After generation:
- Run migrations / create tables
- Wire routes (one line per resource)
- Test: `GET /<plural>` returns `[]` (empty list, 200 OK)

At this point: working CRUD API with collection (pagination, sort, filter, search), HATEOAS `_links`, error handling (404/422 JSON), and transition routes. No business logic yet.

**Failure mode:** If generation fails (e.g., reserved name collision), fix the IR and re-run. Don't hand-edit generated files.

---

## Step 5 — Add Business Logic from Specs

**Who:** AI writes, human reviews.
**Input:** Generated skeleton + behavioral specs.
**Output:** Complete application with business rules.

Read each behavioral spec and modify the generated files. The spec tells you WHAT to implement. The layer architecture tells you WHERE.

### 5a. Contracts → Entity

Invariants become validations in the Entity:

```
invariant: rating in 1..5       →  validates :rating, inclusion: { in: 1..5 }
invariant: email matches <regexp> → validates :email, format: { with: <regexp> }
invariant: price_cents >= 0     →  validates :price_cents, numericality: { >= 0 }
```

If IR had `default` and `transitions`, the generator already produced them in the Entity. You add only what the generator can't know: validation rules from the spec.

### 5b. Lifecycle → Entity + Service

If transitions were in IR, the Entity already has the state machine DSL.
Guards (preconditions) go in the Service via the `guard_transition` hook:

```
Spec says:
  pre(publish): has at least 1 module with lessons

Service implements:
  guard_transition(entity, :publish) →
    check module count, check lessons per module, add error if violated
```

The transition pipeline (find → guard → transition → persist) is provided by the framework. You only write the guard.

Post-transition side effects: override `perform_transition`, call `super`, then execute side effects.

### 5c. Domain Operations → Service

Non-CRUD operations with preconditions (enrollment, review creation):

```
Spec says:
  pre(enroll): course is published
  pre(enroll): student not already enrolled
  pre(enroll): course is not full

Service implements:
  create(attributes) →
    check each precondition, add error if violated, then call repository.create
```

### 5d. Authorization → Endpoint + Service

Two levels from the spec's decision table:

| Table entry | Level | Where |
|------------|-------|-------|
| `✓` / `✗` | Gate | Endpoint auth config (flat role array) |
| `own` | Domain | Service method (check ownership after loading resource) |
| `own AND draft` | Domain | Service method (check ownership + state) |
| `published` | Domain | Service method (filter by state) |

Gate = checked before loading the resource (role only).
Domain = checked after loading the resource (ownership, state).

User context (role, user_id) comes from request headers or auth middleware. The framework provides access via a current-user mechanism (thread-local, CurrentAttributes, or request context — implementation-specific).

### 5e. Side Effects → Service

```
Spec says:
  @on(enroll): increment course.enrollments_count
  @on(enroll): notify student

Service implements:
  after successful create →
    atomic SQL increment on counter field
    create Notification record
```

Side effects execute AFTER the primary operation succeeds. If the primary operation fails (validation error), no side effects.

### Verification

For each line in the behavioral spec, there should be a corresponding line of code. Trace from spec → code. If a spec rule has no implementation, it's a bug.

**Failure mode:** If implementing a rule reveals that the spec is incomplete or contradictory, go back to Step 2. Fix the spec first, then update IR if needed, then re-implement.

---

## Summary

```
1. Listen to the user          → understand the domain
2. Write behavioral specs      → formalize rules, roles, states, effects
3. Derive ir.json              → structure for code generation
4. Generate skeleton           → working CRUD (deterministic)
5. Read specs, write logic     → entity validations, service guards,
                                  transitions, authorization, side effects
```

Behavioral specs are the source of truth for business logic.
IR is the source of truth for structure.
Neither replaces the other.
