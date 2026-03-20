# DRUID IR — Intermediate Representation Format

**Version:** 1.2

## Purpose

DRUID IR is a language-agnostic JSON format that describes domain resources: their attributes, types, relations, state machines, constraints, and custom actions. It serves as the structural contract for code generation.

IR describes WHAT the system IS. Business logic (HOW it behaves) is in behavioral specs, not IR.

## Document structure

```json
{
  "$schema": "druid/1.2",
  "resources": [ ... ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `$schema` | string | yes | Format version. Currently `"druid/1.2"` |
| `resources` | array | yes | Array of resource objects |

## Resource object

```json
{
  "name": "Course",
  "plural": "courses",
  "path": "/courses",
  "attributes": { ... },
  "permit": ["title", "status"],
  "operations": [ ... ],
  "validators": [ ... ],
  "relations": { ... },
  "collection": { ... },
  "transitions": { ... },
  "unique": [ ... ],
  "actions": [ ... ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Resource class name (PascalCase) |
| `plural` | string | yes | Pluralized name. Used for DB table, routes. |
| `path` | string | yes | Base URL path. Always `"/<plural>"`. |
| `attributes` | object | yes | Map of attribute name → attribute descriptor |
| `permit` | array | yes | Writable attribute names |
| `operations` | array | yes | CRUD operations with HTTP method and path |
| `validators` | array | yes | Validation rules (may be empty) |
| `relations` | object | no | Association declarations |
| `collection` | object | no | Pagination, sort, filter, search config |
| `transitions` | object | no | State machine declaration |
| `unique` | array | no | Unique constraints |
| `actions` | array | no | Custom (non-CRUD) actions |

## Attribute descriptor

```json
{
  "title": { "type": "string", "nullable": false, "readonly": false },
  "status": { "type": "string", "nullable": false, "readonly": false, "default": "draft" },
  "created_at": { "type": "datetime", "nullable": false, "readonly": true }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | Type name. See type vocabulary. |
| `nullable` | boolean | yes | Whether the field can be null. |
| `readonly` | boolean | yes | If true, not writable via API. |
| `default` | any | no | Default value. Absent = no default. Allowed: string, integer, float, boolean, null. |

`default` detection: use `key?("default")`, not `default == nil`. This distinguishes "no default" from "default to null".

Datetime defaults are NOT supported in IR — set them in service code (e.g., `Time.current`).

## Type vocabulary

| IR type | Description | JSON value |
|---------|------------|------------|
| `string` | Short text (varchar) | string |
| `text` | Long text (unlimited) | string |
| `integer` | Whole number | number |
| `float` | Floating point | number |
| `decimal` | Precise decimal | number |
| `boolean` | True/false | boolean |
| `date` | Calendar date | string (ISO 8601) |
| `datetime` | Date + time | string (ISO 8601) |
| `time` | Time of day | string |

See [Type Mapping](../implementors/type-mapping.md) for language-specific mappings.

## Operation object

```json
{ "action": "create", "method": "POST", "path": "/courses" }
```

Standard operations:

| Action | Method | Path |
|--------|--------|------|
| `index` | `GET` | `/<plural>` |
| `show` | `GET` | `/<plural>/{id}` |
| `create` | `POST` | `/<plural>` |
| `update` | `PATCH` | `/<plural>/{id}` |
| `destroy` | `DELETE` | `/<plural>/{id}` |

## Validator object

```json
{ "type": "presence", "fields": ["title"] }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | Validator type (e.g., `presence`, `uniqueness`, `format`) |
| `fields` | array | yes | Attribute names this validator applies to |

## Relation object

```json
{
  "relations": {
    "instructor": { "kind": "belongs_to", "resource": "Instructor", "required": true },
    "enrollments": { "kind": "has_many", "resource": "Enrollment" }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `kind` | string | yes | `belongs_to` or `has_many` |
| `resource` | string | yes | Target resource name (PascalCase) |
| `required` | boolean | no | For belongs_to: whether the FK is required. Default: true. |

Self-referential relations: use a different association name than the resource name.
```json
{ "parent": { "kind": "belongs_to", "resource": "Category", "required": false } }
```

## Collection object

```json
{
  "collection": {
    "sort": ["title", "created_at"],
    "filter": ["status", "instructor_id"],
    "search": ["title", "description"],
    "per_page": 20
  }
}
```

All fields optional. Controls pagination, sorting, filtering, and full-text search on the index endpoint.

## Transitions object

Declares a state machine on the resource.

```json
{
  "transitions": {
    "field": "status",
    "events": {
      "publish": { "from": "draft", "to": "published" },
      "archive": { "from": ["draft", "published"], "to": "archived" }
    }
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `field` | string | yes | Attribute that holds the state. Must exist in `attributes`. |
| `events` | object | yes | Map of event name → transition config |

Each event:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `from` | string or array | yes | Source state(s). Array for multiple sources. |
| `to` | string | yes | Target state. |

Implementations MUST auto-create HTTP routes for each transition event:
`POST /<plural>/{id}/<event_name>`

## Unique constraints

```json
{ "unique": ["slug", ["course_id", "student_id"]] }
```

Each element: a string (single column) or array of strings (compound unique).
Implementations SHOULD create unique database indexes.

## Actions (custom endpoints)

```json
{
  "actions": [
    { "name": "mark_read", "method": "PATCH", "on": "member" },
    { "name": "batch_read", "method": "POST", "on": "collection" }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Action name. Must not collide with CRUD or transition names. |
| `method` | string | yes | HTTP method: GET, POST, PATCH, PUT, DELETE |
| `on` | string | yes | `member` (acts on `/{id}`) or `collection` (acts on `/`) |

Routes:
- Member: `<METHOD> /<plural>/{id}/<name>`
- Collection: `<METHOD> /<plural>/<name>`

## Validation rules

Implementations MUST validate IR on parse:
- `$schema` starts with `"druid/"`
- `resources` is a non-empty array
- Each resource has required fields: `name`, `plural`, `attributes`, `permit`, `operations`
- Relation targets exist in the resources array
- Transition `field` exists in resource attributes
- Action names don't collide with CRUD operations or transition event names

## Complete example

```json
{
  "$schema": "druid/1.2",
  "resources": [
    {
      "name": "Course",
      "plural": "courses",
      "path": "/courses",
      "attributes": {
        "id":         { "type": "integer",  "nullable": false, "readonly": true },
        "title":      { "type": "string",   "nullable": false, "readonly": false },
        "slug":       { "type": "string",   "nullable": false, "readonly": false },
        "status":     { "type": "string",   "nullable": false, "readonly": false, "default": "draft" },
        "created_at": { "type": "datetime", "nullable": false, "readonly": true },
        "updated_at": { "type": "datetime", "nullable": false, "readonly": true }
      },
      "permit": ["title", "slug"],
      "operations": [
        { "action": "index",   "method": "GET",    "path": "/courses" },
        { "action": "show",    "method": "GET",    "path": "/courses/{id}" },
        { "action": "create",  "method": "POST",   "path": "/courses" },
        { "action": "update",  "method": "PATCH",  "path": "/courses/{id}" },
        { "action": "destroy", "method": "DELETE",  "path": "/courses/{id}" }
      ],
      "validators": [
        { "type": "presence", "fields": ["title"] }
      ],
      "relations": {
        "instructor": { "kind": "belongs_to", "resource": "Instructor", "required": true }
      },
      "collection": {
        "sort": ["title", "created_at"],
        "filter": ["status"],
        "search": ["title"],
        "per_page": 20
      },
      "transitions": {
        "field": "status",
        "events": {
          "publish": { "from": "draft", "to": "published" },
          "archive": { "from": ["draft", "published"], "to": "archived" }
        }
      },
      "unique": ["slug"]
    }
  ]
}
```
