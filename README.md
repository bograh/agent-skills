# Agent Skills

Reusable Codex skills for software design and code improvement workflows.

This repository is intended to grow over time. It currently includes:

- `design-patterns-agent`: helps identify, compare, explain, and implement GoF design patterns.
- `refactoring-agent`: helps analyze code smells and apply targeted refactorings without changing behavior.
- `springboot-agent`: helps design and build production-grade Spring Boot services across API, data, security, messaging, caching, testing, and observability concerns.

Each skill lives in its own directory and is defined by a `SKILL.md` file plus optional reference material.

## Repository Layout

```text
.
├── design-patterns-agent/
│   ├── SKILL.md
│   └── references/
├── refactoring-agent/
│   ├── SKILL.md
│   └── references/
├── springboot-agent/
│   ├── SKILL.md
│   └── references/
└── README.md
```

## Current Skills

### `design-patterns-agent`

Focused on the 23 GoF design patterns using the refactoring.guru taxonomy.

Use it when you need to:

- choose an appropriate pattern for a design problem
- explain how a pattern works
- compare similar patterns such as `Strategy` vs `State`
- implement a pattern in a specific language
- review architecture for coupling, extensibility, and object collaboration

Reference files:

- `references/creational-patterns.md`
- `references/structural-patterns.md`
- `references/behavioral-patterns.md`

### `refactoring-agent`

Focused on structured refactoring using code smells and refactoring techniques from refactoring.guru.

Use it when you need to:

- clean up messy code without changing behavior
- identify code smells before editing
- break down large methods or classes
- reduce duplication and coupling
- review technical debt in an existing codebase
- prefer incremental refactoring over full rewrites

Reference files:

- `references/code-smells.md`
- `references/techniques.md`

### `springboot-agent`

Focused on production-grade Spring Boot engineering across the full backend stack.

Use it when you need to:

- scaffold a new Spring Boot service with a sane package structure
- build REST or GraphQL APIs
- wire Spring Data, JPA, Flyway, or transactional service layers
- configure Spring Security, OAuth2, or JWT-based auth
- add Spring Cloud components for microservices
- integrate Kafka, RabbitMQ, Redis, or Caffeine
- add testing, observability, and operational defaults for production

Reference files:

- `references/project-structure.md`
- `references/data.md`
- `references/security.md`
- `references/rest-graphql.md`
- `references/cloud-microservices.md`
- `references/messaging.md`
- `references/caching.md`
- `references/observability.md`
- `references/testing.md`

## How These Skills Are Meant To Be Used

These skills are designed for Codex-style skill loading:

- each skill declares a `name` and `description` in frontmatter
- the description includes trigger guidance so the assistant knows when to use it
- the body defines a concrete workflow instead of generic advice
- `references/` files hold deeper source material that should be loaded only when needed

In practice, a skill should help the assistant do three things well:

1. Detect when the skill is relevant.
2. Follow a repeatable workflow.
3. Produce consistent, high-signal output.

## Installing In A Codex Environment

If your Codex setup supports local skills, copy or symlink the skill directories into your skills directory so each skill remains a top-level folder containing `SKILL.md`.

Example:

```bash
mkdir -p "$CODEX_HOME/skills"
ln -s "/path/to/agent-skills/design-patterns-agent" "$CODEX_HOME/skills/design-patterns-agent"
ln -s "/path/to/agent-skills/refactoring-agent" "$CODEX_HOME/skills/refactoring-agent"
ln -s "/path/to/agent-skills/springboot-agent" "$CODEX_HOME/skills/springboot-agent"
```

If your environment uses a different skills path, adapt the target directory accordingly.

## Authoring Conventions

This repository follows a simple structure for each skill:

- `SKILL.md` contains the frontmatter, trigger guidance, workflow, and usage rules
- `references/` contains supporting material that is too detailed for the main skill file
- each skill should stay narrow in scope and opinionated in workflow

When adding a new skill:

1. Create a new top-level directory.
2. Add a `SKILL.md` with clear trigger conditions.
3. Define a workflow the assistant can execute repeatedly.
4. Add only the reference files needed to support that workflow.
5. Keep the skill focused on a single job or tightly related set of jobs.

## Future Additions

Additional skills can be added over time for areas such as:

- testing and verification
- debugging and root-cause analysis
- API design
- architecture review
- documentation writing

## License

Add a license file if you intend to distribute or reuse these skills outside your local environment.
