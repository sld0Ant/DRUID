# DRUID — Type Mapping

How IR types map to language-specific types.

## IR type vocabulary

9 types. All map to every common language.

| IR type | Description | JSON value type |
|---------|------------|----------------|
| `string` | Short text (typically VARCHAR, ≤255 chars) | string |
| `text` | Long text (unlimited length, TEXT/CLOB) | string |
| `integer` | Whole number (64-bit) | number |
| `float` | IEEE 754 floating point | number |
| `decimal` | Arbitrary precision decimal | number |
| `boolean` | True or false | boolean |
| `date` | Calendar date (no time) | string (ISO 8601: `2025-01-15`) |
| `datetime` | Date + time with timezone | string (ISO 8601: `2025-01-15T10:30:00Z`) |
| `time` | Time of day | string (`10:30:00`) |

## Language mappings

### Ruby

| IR type | Ruby type | ActiveModel type symbol |
|---------|-----------|----------------------|
| `string` | `String` | `:string` |
| `text` | `String` | `:string` (same Ruby type, different DB column) |
| `integer` | `Integer` | `:integer` |
| `float` | `Float` | `:float` |
| `decimal` | `BigDecimal` | `:decimal` |
| `boolean` | `TrueClass` / `FalseClass` | `:boolean` |
| `date` | `Date` | `:date` |
| `datetime` | `DateTime` / `Time` | `:datetime` |
| `time` | `Time` | `:time` |

### TypeScript

| IR type | TypeScript type | JSON parse |
|---------|----------------|-----------|
| `string` | `string` | as-is |
| `text` | `string` | as-is |
| `integer` | `number` | as-is |
| `float` | `number` | as-is |
| `decimal` | `number` | `parseFloat()` (precision loss possible) |
| `boolean` | `boolean` | as-is |
| `date` | `string` | ISO string, or `Date` via constructor |
| `datetime` | `string` | ISO string, or `Date` via constructor |
| `time` | `string` | as-is |

### Python

| IR type | Python type | Pydantic/SQLAlchemy |
|---------|-------------|-------------------|
| `string` | `str` | `String(255)` |
| `text` | `str` | `Text` |
| `integer` | `int` | `Integer` |
| `float` | `float` | `Float` |
| `decimal` | `Decimal` | `Numeric` |
| `boolean` | `bool` | `Boolean` |
| `date` | `datetime.date` | `Date` |
| `datetime` | `datetime.datetime` | `DateTime` |
| `time` | `datetime.time` | `Time` |

### Go

| IR type | Go type | SQL driver |
|---------|---------|-----------|
| `string` | `string` | `string` |
| `text` | `string` | `string` |
| `integer` | `int64` | `int64` |
| `float` | `float64` | `float64` |
| `decimal` | `float64` (or `shopspring/decimal`) | `float64` |
| `boolean` | `bool` | `bool` |
| `date` | `time.Time` | `time.Time` |
| `datetime` | `time.Time` | `time.Time` |
| `time` | `time.Time` | `time.Time` |

### Java

| IR type | Java type | JPA annotation |
|---------|-----------|---------------|
| `string` | `String` | `@Column(length = 255)` |
| `text` | `String` | `@Lob` |
| `integer` | `Long` | `@Column` |
| `float` | `Double` | `@Column` |
| `decimal` | `BigDecimal` | `@Column(precision = 19, scale = 4)` |
| `boolean` | `Boolean` | `@Column` |
| `date` | `LocalDate` | `@Column` |
| `datetime` | `Instant` | `@Column` |
| `time` | `LocalTime` | `@Column` |

### Rust

| IR type | Rust type | Diesel type |
|---------|-----------|------------|
| `string` | `String` | `Varchar` |
| `text` | `String` | `Text` |
| `integer` | `i64` | `BigInt` |
| `float` | `f64` | `Double` |
| `decimal` | `bigdecimal::BigDecimal` | `Numeric` |
| `boolean` | `bool` | `Bool` |
| `date` | `chrono::NaiveDate` | `Date` |
| `datetime` | `chrono::NaiveDateTime` | `Timestamp` |
| `time` | `chrono::NaiveTime` | `Time` |

### SQL

| IR type | SQLite | PostgreSQL | MySQL |
|---------|--------|-----------|-------|
| `string` | `TEXT` | `varchar(255)` | `varchar(255)` |
| `text` | `TEXT` | `text` | `text` |
| `integer` | `INTEGER` | `bigint` | `bigint` |
| `float` | `REAL` | `double precision` | `double` |
| `decimal` | `REAL` | `numeric(19,4)` | `decimal(19,4)` |
| `boolean` | `INTEGER` (0/1) | `boolean` | `tinyint(1)` |
| `date` | `TEXT` | `date` | `date` |
| `datetime` | `TEXT` | `timestamp` | `datetime` |
| `time` | `TEXT` | `time` | `time` |

## Edge cases

### `text` vs `string`

Both map to `String`/`string`/`str` in application code. The difference is storage:
- `string` → VARCHAR (limited length, indexed efficiently)
- `text` → TEXT/CLOB (unlimited, not efficiently indexed)

Use `string` for: names, slugs, emails, short labels.
Use `text` for: descriptions, body content, bio, comments.

Round-trip: extractors must read the DB column type to distinguish. Entity code alone cannot — both are `String`.

### `decimal` precision

IR does not specify precision/scale. Implementations SHOULD use a sensible default:
- Money: `decimal(19, 4)` or use integer cents (`price_cents: integer`)
- Ratings: `decimal(3, 2)` for 1.00–5.00
- Percentages: `decimal(5, 2)` for 0.00–100.00

If precision matters, prefer integer representation (cents, basis points) over decimal.

Note: IR 1.2 does not support precision/scale parameters on `decimal`. Implementations SHOULD use a sensible default (e.g., `(19, 4)`). A future IR version may add `precision` and `scale` as optional fields on decimal attributes.

### State fields and enums

IR represents state fields as `string` with a `transitions` block. In some languages, these map better to enums:

| Language | State field representation |
|----------|--------------------------|
| Ruby | `String` with `validates :status, inclusion: {in: [...]}` |
| TypeScript | `type Status = "draft" \| "published" \| "archived"` (union type) |
| Python | `enum.StrEnum` with values from transition states |
| Go | `string` type alias with constants |
| Rust | `enum Status { Draft, Published, Archived }` |
| Java | `enum Status { DRAFT, PUBLISHED, ARCHIVED }` |

Implementations MAY generate enum types for state fields. The set of valid values is derivable from transitions: collect all unique `from` and `to` values.

### Nullable types

IR `nullable: true` means the field can be null/nil/None.

| Language | Nullable representation |
|----------|----------------------|
| Ruby | No special syntax (everything nullable) |
| TypeScript | `string \| null` |
| Python | `Optional[str]` / `str \| None` |
| Go | `*string` (pointer) or `sql.NullString` |
| Rust | `Option<String>` |
| Java | `@Nullable String` or `Optional<String>` |
