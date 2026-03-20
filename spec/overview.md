# DRUID — Overview

## Problem

Building domain-driven APIs requires writing the same structural boilerplate for every resource: entity, repository, service, endpoint, migration, routes. Business logic is mixed with infrastructure. AI agents generating code lack a formal contract for what the domain looks like.

DRUID separates **structure** (what resources exist and how they're shaped) from **behavior** (what business rules apply), giving each a formal specification that both humans and AI agents can read, write, and verify.

## Core idea

```
Structure (IR)  →  generated code (skeleton)
Behavior (spec) →  hand-written logic (by human or AI)
```

IR describes WHAT the system IS.
Behavioral specs describe HOW it BEHAVES.
Neither replaces the other.

## Applicability

### DRUID is designed for systems that have:

- **Named resources** with typed attributes (User, Order, Product)
- **CRUD operations** over HTTP (REST APIs)
- **Relations** between resources (belongs_to, has_many)
- **Domain logic** beyond simple persistence (validations, guards, state transitions, authorization)
- **Multiple roles** with different access levels

### DRUID works well when:

- An AI agent builds the API from a human's domain description
- The team wants a formal contract between "what we build" and "what we described"
- Multiple implementations share the same domain model (e.g., Ruby backend + TypeScript SDK)
- The project has 5+ resources with interconnected business rules

### DRUID is NOT a fit for:

- **Stateless pipelines** — no named resources, no CRUD
- **Event-driven / streaming systems** — DRUID models resources, not event flows
- **CLI tools / scripts** — no HTTP layer, no REST model
- **Pure CRUD without domain logic** — standard MVC is simpler, DDD layers add overhead
- **Graph APIs (GraphQL)** — DRUID assumes REST; GraphQL has its own schema language
- **Real-time systems** — DRUID doesn't model WebSocket channels, pub/sub, or message queues

## Architectural principles

### 1. Four DDD layers

Every DRUID implementation must separate code into four layers:

```
UI            Endpoint / Controller — HTTP adapter, no logic
Application   Service — orchestration, guards, side effects
Domain        Entity — domain object, validations, state machine, no DB
Infrastructure  Repository + Record — persistence, ORM, hidden from domain
```

### 2. IR as structural contract

IR (Intermediate Representation) is a JSON file that describes resources. It drives code generation. It does NOT describe business logic — that's in behavioral specs.

### 3. Behavioral specs as logic contract

Behavioral specs are markdown files using five formal methods:

| Section | Based on | Captures |
|---------|----------|----------|
| Contracts | Design by Contract (Meyer, 1986) | Invariants, preconditions, postconditions |
| Authorization | Decision Tables (1957) | Role × action × condition matrix |
| Lifecycle | Statecharts (Harel, 1987) | State transitions and guards |
| Scenarios | BDD / Gherkin (North, 2003) | Concrete examples, edge cases |
| Side Effects | Tagged NL annotations | Consequences beyond direct response |

### 4. AI-native

DRUID is designed for AI agents to consume:
- IR is machine-readable JSON
- Behavioral specs use structured conventions that LLMs parse reliably
- The flow is sequential and deterministic (spec → IR → code → logic)
- Each step produces a verifiable artifact

### 5. Zero coupling

- IR is language-agnostic (JSON with type vocabulary that maps to any language)
- Behavioral specs are framework-agnostic (markdown)
- Implementations are independent (Rails, NestJS, FastAPI can each implement DRUID)

## Versioning

DRUID IR uses the `$schema` field: `"druid/1.2"`.

Major version = breaking changes. Minor version = new optional fields.

Implementations SHOULD check the major version and reject incompatible files.
Implementations SHOULD accept any minor version of the same major version.
