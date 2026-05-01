# Skill: Domain Models

Deeply explore a service repo and generate `docs/domain-models.md` — a comprehensive human-readable domain model reference with Mermaid diagrams. Automatically discovers related transport/proto repos in sibling directories.

## When to use this skill

- When onboarding to a new service and need a full domain model reference
- When preparing for a cross-service change and need to understand data structures
- When documenting the data model of a repo for the first time or after major changes
- Called by `atlas domain-models` or invoked autonomously when the user asks for domain model documentation

---

## Step 1 — Resolve the target repo

If an argument was provided, treat it as either:
- An absolute path (use as-is)
- A repo name — search for it as a subdirectory of the current working directory

If no argument was provided, use the current working directory as the target repo.

Call the resolved path `TARGET_REPO`.

---

## Step 2 — Discover sibling repos with transport/proto models

From the **parent directory** of `TARGET_REPO`, list all sibling directories. Then identify which ones are likely to contain shared transport models or protobuf definitions by looking for:
- Directory names containing: `transport`, `proto`, `grpc`, `models`, `schema`, `contracts`, `events`
- Files matching: `**/*.proto`, `**/transport_models*`, `**/grpc_models*`
- Go modules or package names that are imported by `TARGET_REPO`

Scan the top-level `go.mod`, `package.json`, `Cargo.toml`, `Gemfile`, or `requirements.txt` of `TARGET_REPO` for imports that look like shared model packages, and cross-reference with sibling directories.

Call the list of discovered sibling model repos `TRANSPORT_REPOS`.

---

## Step 3 — Explore TARGET_REPO deeply

For each of the following, read actual source files (not just directory listings):

### Language & structure
- Detect language(s): Go, Rust, Ruby, TypeScript, Python, Java, etc.
- Read `README.md` and any existing architecture docs
- Map the top-level directory layout

### Domain models
Look in: `internal/domain/`, `app/models/`, `src/models/`, `lib/`, `domain/`, `pkg/`, `crates/`

For each model/struct/class found:
- All fields with types
- Relationships to other models (foreign keys, embedded structs, associations)
- Status enums or state machine fields
- Key methods or business logic on the model
- DRN/UUID/ID fields that cross service boundaries

### State machines & status flows
- Any status enums and their allowed transitions
- Find `switch/case` blocks or state machine libraries that encode transitions
- Note any special override transitions (like `COMPLETED → FAILED` race-condition patterns)

### Data persistence
- What databases: DynamoDB, Postgres, Redis, MySQL, MongoDB?
- Table/collection schemas and key structures (partition keys, sort keys, GSIs)
- Caching patterns (what's cached, TTLs)

### Event / message flows
- Kafka topics consumed and produced (with message types)
- SQS queues, SNS topics, webhooks
- DynamoDB streams or other triggers
- gRPC services called or served

### External integrations
- Other services called (by name or URL pattern)
- Auth patterns (OAuth, JWT, service tokens)

### API surface
- REST endpoints (path, method, request/response shape)
- gRPC methods
- WebSocket or streaming endpoints

---

## Step 4 — Explore TRANSPORT_REPOS

For each repo in `TRANSPORT_REPOS`:
- Read `.proto` files, Go structs, or schema files relevant to `TARGET_REPO`
- Focus on message types that `TARGET_REPO` produces or consumes
- Note the field names and types that cross the service boundary

---

## Step 5 — Write docs/domain-models.md

Create the file at `TARGET_REPO/docs/domain-models.md`. Include only sections that have content — skip empty sections rather than writing placeholders.

```
# {ServiceName} Domain Models & Architecture

> One-line description of what this service does.

---

## Table of Contents
(auto-generate based on sections present)

---

## Overview
- Language/framework
- Key responsibilities (3-5 bullet points)
- Top-level repo structure table

---

## High-Level Architecture
Mermaid `graph TD` showing:
- External inputs (Kafka topics, API clients, other services)
- This service's components (API, consumers, workers)
- Data stores (DynamoDB, Redis, Postgres, etc.)
- Outputs (Kafka topics published, webhooks, downstream services)

---

## Core Domain Models
For each major entity:
- Mermaid `classDiagram` with all fields and types
- Prose description: what it represents, its lifecycle, key invariants
- Relationships to other models

---

## [Entity] vs [Entity] (if disambiguation needed)
Side-by-side comparison table + flow diagram when two similar models need distinguishing

---

## Status Lifecycles
Mermaid `stateDiagram-v2` for each entity that has a status/state machine.
Add a note explaining any non-obvious transitions (e.g., race condition overrides).

---

## Key Flows
For each major operation (creation, update, cancellation, etc.):
Mermaid `sequenceDiagram` showing the full path from trigger to side effects.

---

## External Integrations
Table: service name | purpose | protocol | direction

---

## Kafka / Event Integration
Two tables:
- Topics consumed: topic | message type | handler | effect
- Topics produced: topic | message type | trigger condition

---

## Data Persistence
Table per store: store | purpose | key schema | notable patterns

---

## API Surface (if applicable)
Table of endpoints with method, path, auth, and brief description.
Include abbreviated request/response shape for the most important endpoint.

---

## Cross-Service ID Glossary (if multiple ID types exist)
Table: id name | owner service | what it identifies
Mermaid `graph LR` showing how IDs relate across service boundaries.
```

---

## Mermaid rules — avoid parse errors

Follow these rules strictly to prevent Mermaid parse errors:

1. **No `[]` in relationship targets** — use cardinality notation instead:
   - ❌ `Entity --> "[]OtherEntity"`
   - ✅ `Entity "1" --> "*" OtherEntity`

2. **No spaces or dots in quoted node IDs** in `graph` diagrams:
   - ❌ `Assignment --> "1..N DeliveryRequest"`
   - ✅ `Assignment -->|1..*| DR[DeliveryRequest]`

3. **No parentheses in bare node IDs** — use `NodeID[Label with (parens)]` form:
   - ❌ `delivery_drn (Roogo) --> X`
   - ✅ `DeliveryDrn["delivery_drn - Roogo"] --> X`

4. **Pipe characters in labels** must be inside `|label|` edge syntax only, never bare

5. **classDiagram field arrays** — use `List~Type~` or just `Type list` notation:
   - ❌ `Items: []Item` (may cause issues in some renderers)
   - ✅ `Items: Item list`

6. In `stateDiagram-v2`, state names with underscores are fine but avoid special chars

---

## Quality bar

The output should be good enough that a new engineer joining the team can:
- Understand what the service does in 5 minutes (Overview + Architecture diagram)
- Find any domain model and understand its fields and lifecycle (Core Domain Models)
- Trace a request end-to-end through the system (Key Flows)
- Know which IDs to use when querying across services (ID Glossary)

Do not write placeholder sections. If a section has no content (e.g. the service has no Kafka integration), omit it entirely.
