# DRUID — Behavioral Specification Format

Behavioral specs capture business rules for DRUID resources. IR describes **structure** (what the system is). Behavioral specs describe **behavior** (how it acts).

One markdown file per resource: `docs/behavior/<resource>.md`

## When to write a behavioral spec

A resource needs a spec when it has any of:
- Validation rules beyond NOT NULL (contracts)
- Role-based access control (authorization)
- A status/state field with transitions (lifecycle)
- Non-obvious edge cases (scenarios)
- Consequences beyond the direct write (side effects)

A resource is **pure CRUD** and needs no spec when it has no transitions, no custom actions, no authorization beyond default, and no business rules beyond field-level presence/type validation. Example: a Tag or Category resource with just name and slug.

## The five sections

Each section uses a formal method from computer science. Use only the sections that apply. Order is fixed.

| # | Section | Based on | Captures |
|---|---------|----------|----------|
| 1 | Contracts | Design by Contract (Meyer, 1986) | Invariants, preconditions, postconditions |
| 2 | Authorization | Decision Tables (GE, 1957) | Role × action permission matrix |
| 3 | Lifecycle | Statecharts (Harel, 1987) | State transitions and guards |
| 4 | Scenarios | BDD / Gherkin (North, 2003) | Concrete examples, edge cases, error paths |
| 5 | Side Effects | Tagged natural language | Consequences beyond the direct response |

---

## 1. Contracts

Entity invariants and operation pre/postconditions.

### Syntax

```
invariant: <boolean expression>
```

```
# <OperationName>
pre:  <boolean expression>
post: <boolean expression>
```

**Rules:**
- `invariant` — must be true at ALL times for every instance of this resource
- `pre` — must be true BEFORE the operation executes. Violation → 422 error.
- `post` — guaranteed true AFTER the operation succeeds
- Grouped under operation name with `#` prefix
- Write only rules NOT obvious from the data type (e.g., don't write `invariant: id is integer`)

### Example

```
invariant: rating in 1..5
invariant: email matches URI::MailTo::EMAIL_REGEXP
invariant: price_cents >= 0
invariant: max_students > 0 (when present)

# PublishCourse
pre:  course.status == "draft"
pre:  course.description is not blank
pre:  course has at least 1 module
pre:  each module has at least 1 lesson
post: course.status == "published"
post: course.published_at is set

# EnrollStudent
pre:  course.status == "published"
pre:  student not already enrolled (no active/completed enrollment)
pre:  course is not full (enrollments_count < max_students)
post: enrollment.status == "active"
post: course.enrollments_count incremented by 1
```

### Skip when

Resource has no invariants beyond NOT NULL / type constraints, and no operations with preconditions.

---

## 2. Authorization

Role × action permission matrix.

### Syntax

Markdown table. Columns = roles. Rows = actions.

Cell values:

| Value | Meaning |
|-------|---------|
| `✓` | Unconditionally allowed |
| `✗` | Denied |
| `own` | Allowed only on resources owned by this user |
| `own AND <condition>` | Allowed on own resources matching condition |
| `own OR <condition>` | Allowed on own OR matching condition |
| `<condition>` | Allowed only when condition is met (e.g., `published`) |

**Rules:**
- Every action the resource supports must have a row
- Include transition actions (publish, archive) and custom actions (mark_read)
- Conditions refer to the resource's current state or the user's relationship to it
- `own` must be defined in context below the table. Syntax: `own for <role>: <resource_field> == current_user.id`. Example: `own for instructor: course.instructor_id == current_user.id`

### Example

```
| Action   | admin | instructor      | student         | guest      |
|----------|-------|-----------------|-----------------|------------|
| index    | ✓     | ✓               | published       | published  |
| show     | ✓     | own OR published| published       | published  |
| create   | ✓     | ✓               | ✗               | ✗          |
| update   | ✓     | own             | ✗               | ✗          |
| destroy  | ✓     | own AND draft   | ✗               | ✗          |
| publish  | ✓     | own             | ✗               | ✗          |
| archive  | ✓     | own             | ✗               | ✗          |
```

### Skip when

All actions are public, or authorization is handled entirely by global middleware outside the resource.

---

## 3. Lifecycle

State transitions for resources with a status/state field.

### Syntax

```
[<state>] --<event>--> [<state>]
```

Followed by event details:

```
<event>: <from> → <to>, guard: <condition>
<event>: <from> → <to>, no guard
```

**Rules:**
- `from` can be a single state or a list separated by `|`
- `guard` is a precondition that must be true for the transition to fire (beyond the `from` state check)
- Guards are shorthand — the full precondition is in Contracts section
- Mark terminal states (no outgoing transitions)

### Example

```
[draft] --publish--> [published] --archive--> [archived]
   \                                            /
    `---archive--> [archived]

publish: draft → published, guard: has description + modules + lessons
archive: draft|published → archived, no guard

Terminal: archived (no outgoing transitions)
```

### Skip when

Resource has no status/state field, or the field is free-form (no constrained transitions).

---

## 4. Scenarios

Concrete examples of behavior, focusing on non-obvious cases.

### Syntax

```
Given <initial state>
When <action>
Then <expected result>
```

**Rules:**
- Don't write scenarios for standard CRUD success paths — the framework handles those
- Focus on: error cases, guard rejections, edge cases, state-dependent behavior
- Each scenario should test ONE rule

### Example

```
Given course with 0 modules
When instructor calls POST /courses/:id/publish
Then 422, errors: ["At least one module required"]

Given course in status "archived"
When instructor calls PATCH /courses/:id with {title: "New"}
Then 422 (archived courses are immutable)

Given published course with max_students: 2 and 2 active enrollments
When student calls POST /enrollments
Then 422, errors: ["Course is full"]

Given student with cancelled enrollment on course
When student calls POST /enrollments for same course
Then 201 (cancelled enrollment does not block re-enrollment)
```

### Skip when

Resource is pure CRUD with no edge cases or error paths beyond standard validation.

---

## 5. Side Effects

What happens beyond the direct response of an operation.

### Syntax

```
@on(<event>): <consequence>
```

**Rules:**
- `<event>` is an operation name or transition name: `create`, `publish`, `confirm`, etc.
- `<consequence>` is a natural language description of the side effect
- Side effects are informational — they tell the implementor what additional work to do
- They do NOT define the response body (that's determined by the entity)

### Example

```
@on(publish):  set published_at to current time
@on(publish):  notify instructor "Course published"
@on(enroll):   increment course.enrollments_count (atomic SQL)
@on(enroll):   notify student "Enrolled in {course.title}"
@on(complete):  create Certificate with UUID number
@on(complete):  notify student "Certificate issued"
@on(refund):   decrement coupon.used_count (atomic SQL)
```

### Skip when

Operations have no consequences beyond the direct database write.

---

## Complete example

---

# Course — Behavioral Spec

## Contracts

```
invariant: title is present
invariant: price_cents >= 0
invariant: status in ["draft", "published", "archived"]

# PublishCourse
pre:  description is not blank
pre:  has at least 1 module with at least 1 lesson each
pre:  organization has not exceeded course limit
post: status == "published"
post: published_at is set

# ArchiveCourse
post: status == "archived"
```

## Authorization

| Action   | admin | instructor      | student   | guest     |
|----------|-------|-----------------|-----------|-----------|
| index    | ✓     | ✓               | published | published |
| show     | ✓     | own OR published| published | published |
| create   | ✓     | ✓               | ✗         | ✗         |
| update   | ✓     | own             | ✗         | ✗         |
| destroy  | ✓     | own AND draft   | ✗         | ✗         |
| publish  | ✓     | own             | ✗         | ✗         |
| archive  | ✓     | own             | ✗         | ✗         |

`own` for instructor: `course.instructor_id == current_user.id`

## Lifecycle

```
[draft] --publish--> [published] --archive--> [archived]
   \                                            /
    `---archive--> [archived]

publish: draft → published, guard: description + modules + lessons + org limit
archive: draft|published → archived, no guard
```

## Scenarios

```
Given course with empty description
When instructor calls POST /courses/:id/publish
Then 422, errors: ["Description can't be blank"]

Given published course
When instructor calls DELETE /courses/:id
Then 403 (only draft courses deletable by instructor)
```

## Side Effects

```
@on(publish): set published_at = now
@on(publish): notify instructor "Course {title} published"
```
