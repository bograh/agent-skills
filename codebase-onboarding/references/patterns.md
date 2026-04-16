# Architectural Patterns Reference

Read this file during Phase 3c to classify the architecture of the codebase being onboarded.
For each pattern, signal words are things you'd see in directory names, class names, or docs.

---

## MVC (Model-View-Controller)

**Description**: Separates data (Model), presentation (View), and business logic (Controller).
Classic web app pattern — a request hits a Controller, which reads/writes a Model and renders
a View.

**Signal words / directories**: `models/`, `views/`, `controllers/`, `templates/`,
`app/models`, `app/controllers`, `app/views`

**Common frameworks**: Rails, Django, Laravel, Spring MVC, ASP.NET MVC, Symfony

**What to read first**: A controller file + the corresponding model — this pair reveals the
core domain logic and how data flows from DB to response.

---

## Layered / N-Tier

**Description**: Code organized in horizontal layers — typically Presentation, Application/
Service, Domain, and Infrastructure/Data. Each layer only calls the one below it.

**Signal words / directories**: `services/`, `repositories/`, `handlers/`, `usecases/`,
`domain/`, `infrastructure/`, `data/`, `persistence/`

**Common frameworks**: Spring Boot (Java), NestJS (Node), .NET layered apps

**What to read first**: A service layer file — it's usually the most expressive of business
rules. Then find the repository it uses to understand the data contract.

---

## Hexagonal / Clean / Onion Architecture

**Description**: Business logic at the center, completely isolated from I/O and frameworks.
External concerns (DB, HTTP, queues) connect via "ports" (interfaces) and "adapters"
(implementations). Enables swapping infrastructure without touching business rules.

**Signal words / directories**: `core/`, `domain/`, `ports/`, `adapters/`, `application/`,
`use_cases/`, `entities/`, `internal/` (Go), `driven/`, `driving/`

**What to read first**: The `domain/` or `core/` directory — this is the heart of the system.
Then find one adapter (e.g., a DB repository implementation) to see how the boundary works.

---

## Event-Driven

**Description**: Components communicate by emitting and consuming events rather than direct
calls. Highly decoupled — producers don't know about consumers.

**Signal words / directories**: `events/`, `handlers/`, `listeners/`, `consumers/`,
`producers/`, `subscribers/`, `publishers/`, `commands/`, `sagas/`

**Common tech**: Kafka, RabbitMQ, SQS, Redis Pub/Sub, EventBridge, in-process event bus

**What to read first**: Find the event type definitions — they are the API of the system.
Then find one producer and one consumer to see the full cycle.

---

## Microservices / Service-Oriented

**Description**: Multiple small, independently deployable services, each owning its own data.
Often a monorepo with separate service directories, or separate repos per service.

**Signal words / directories**: Individual service folders at root (`user-service/`,
`payment-service/`, `notification-service/`), `services/`, `api-gateway/`, proto files,
`docker-compose.yml` with multiple services

**What to read first**: The `docker-compose.yml` or `k8s/` manifests give the best
bird's-eye view of what services exist. Then pick the service most relevant to the user's
task.

---

## Monorepo with Packages / Workspaces

**Description**: A single repository containing multiple logically separate packages or apps,
sharing tooling and potentially sharing code via internal packages.

**Signal words / directories**: `packages/`, `apps/`, `libs/`, `modules/`, `workspaces`
in package.json, `Cargo.toml` `[workspace]`, `go.work`

**Common tools**: Turborepo, Nx, Lerna, Bazel, pnpm workspaces

**What to read first**: The root `package.json` or workspace config to understand the package
graph. Then find the `packages/` or `apps/` directory listing and decide which package is
relevant.

---

## Pipeline / ETL / Data Pipeline

**Description**: Data flows through a series of transformation steps. May be batch or
streaming. Often has explicit DAGs (directed acyclic graphs) of tasks.

**Signal words / directories**: `pipelines/`, `jobs/`, `tasks/`, `stages/`, `transforms/`,
`sources/`, `sinks/`, `dags/` (Airflow), `workflows/`

**Common frameworks**: Apache Airflow, Prefect, Dagster, Luigi, Spark, dbt, Beam

**What to read first**: The DAG/pipeline definition file — it's the map of the whole system.
Then find one transform step to understand the node contract.

---

## Plugin / Extension Architecture

**Description**: A core host application loads and executes plugins/extensions at runtime.
Enables extensibility without modifying core.

**Signal words / directories**: `plugins/`, `extensions/`, `addons/`, `hooks/`, `middleware/`,
`contrib/`, plugin interfaces/base classes

**What to read first**: The plugin interface or base class — it defines the contract every
plugin must fulfill. Then read one example plugin.

---

## CQRS (Command Query Responsibility Segregation)

**Description**: Read (Query) and Write (Command) operations use separate models and often
separate data stores. Often paired with Event Sourcing.

**Signal words / directories**: `commands/`, `queries/`, `command_handlers/`,
`query_handlers/`, `read_models/`, `write_models/`, `projections/`, `aggregates/`

**What to read first**: One Command + its handler to understand write side, one Query + its
handler to understand read side.

---

## REST API / Backend-for-Frontend

**Description**: Server exposes HTTP endpoints consumed by a client (browser, mobile, other
services). May be thin (CRUD) or thick (business logic).

**Signal words / directories**: `routes/`, `endpoints/`, `api/`, `controllers/`, `resources/`,
`schemas/` (for validation), `serializers/`

**What to read first**: The router/routes file — it's the API surface. Then pick one
endpoint and trace it through to the data layer.

---

## How to pick the right pattern

When unsure, ask these questions:

1. **Is there a `models/`+`views/`+`controllers/` split?** → MVC
2. **Is there a `domain/` or `core/` folder clearly separated from `infrastructure/`?** → Hexagonal/Clean
3. **Are there multiple top-level service folders, each with their own `Dockerfile`?** → Microservices
4. **Is there a `packages/` or `apps/` folder with multiple sub-projects?** → Monorepo
5. **Are there `events/` and `consumers/` or Kafka/RabbitMQ in the deps?** → Event-driven
6. **Is there a DAG definition file or `pipelines/` folder?** → Pipeline/ETL
7. **Are there `services/` + `repositories/` side by side (no explicit domain)?** → Layered
8. **Is it just `routes/` + `controllers/` + an ORM with no other layering?** → REST API / thin MVC

Most real codebases blend patterns — pick the *dominant* one and note any secondary patterns.
