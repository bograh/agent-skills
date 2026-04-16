---
name: codebase-onboarding
description: >
  Systematically onboard to a new or unfamiliar codebase by exploring structure, understanding
  architecture, mapping data flows, identifying key entry points, and producing a concise
  orientation report. Use this skill whenever the user says things like "help me understand
  this codebase", "I'm new to this project", "walk me through this repo", "onboard me to
  this code", "what does this project do", "explain the architecture", or uploads/points to
  a codebase they want to explore. Also trigger when the user asks "where do I start" or
  "how is this organized" about a code project, or when they need to understand an unfamiliar
  repository before making changes. Always use this skill proactively when the context
  suggests the user is encountering a codebase for the first time.
---

# Codebase Onboarding Skill

Helps Claude (and the user) get oriented in an unfamiliar codebase quickly and systematically.
The goal is a mental model, not exhaustive documentation — produce the clearest possible map
with the least reading time.

---

## Phase 0 — Locate the codebase

Before anything else, confirm where the code lives:

- Is a path provided? Use it directly.
- Is a repo URL given? Clone it: `git clone <url> /tmp/repo && cd /tmp/repo`
- Is a zip/archive uploaded? Extract it first.
- Nothing provided? Ask the user where the code is.

---

## Phase 1 — Orientation (start here, always)

Run these commands to get the lay of the land. Do them fast — this is scanning, not reading.

```bash
# 1. Top-level snapshot
ls -la <root>

# 2. Recursive tree (limit depth to avoid noise)
find <root> -maxdepth 3 -not -path '*/.git/*' -not -path '*/node_modules/*' \
  -not -path '*/__pycache__/*' -not -path '*/vendor/*' | sort

# 3. README / docs
cat <root>/README.md 2>/dev/null || cat <root>/README.rst 2>/dev/null || \
  cat <root>/README.txt 2>/dev/null || echo "No README found"

# 4. Changelog / release notes (signals project maturity, recent changes)
cat <root>/CHANGELOG.md 2>/dev/null | head -80 || true

# 5. License (signals open/closed source context)
ls <root>/LICENSE* 2>/dev/null || true
```

After Phase 1, form a hypothesis about:
- **Type**: web app / CLI / library / service / monorepo / data pipeline / other
- **Language(s)**: primary + secondary
- **Framework(s)**: if detectable

---

## Phase 2 — Language & Dependency Mapping

Detect the ecosystem and read its manifest files. Read the reference file for the detected
language/ecosystem: `references/ecosystems.md`.

Quick detection heuristics:

| File present            | Ecosystem        |
|-------------------------|------------------|
| `package.json`          | Node / JS / TS   |
| `requirements.txt` / `pyproject.toml` / `setup.py` | Python |
| `Cargo.toml`            | Rust             |
| `go.mod`                | Go               |
| `pom.xml` / `build.gradle` | Java / Kotlin |
| `Gemfile`               | Ruby             |
| `composer.json`         | PHP              |
| `*.csproj` / `*.sln`   | .NET / C#        |
| `mix.exs`               | Elixir           |

For each manifest found:
- List declared dependencies (focus on the top 5-10 most significant)
- Note dev vs. production dependencies
- Check for lock files (shows determinism hygiene)
- Look for scripts / tasks section (reveals common workflows)

---

## Phase 3 — Architecture Discovery

### 3a. Identify entry points

These are the "front doors" of the codebase:

```bash
# Web servers / HTTP handlers
grep -r "listen\|createServer\|app\.run\|uvicorn\|gunicorn\|gin\.Run\|http\.ListenAndServe" \
  <root> --include="*.py" --include="*.js" --include="*.ts" --include="*.go" \
  -l 2>/dev/null | head -10

# CLI entry points
grep -r "if __name__.*main\|func main()\|fn main()\|public static void main" \
  <root> -l --include="*.py" --include="*.go" --include="*.rs" --include="*.java" \
  2>/dev/null | head -10

# Config-declared entry points (check package.json main/bin, Cargo.toml [[bin]], etc.)
```

### 3b. Map the directory structure to concepts

For each top-level directory (or notable subdirectory), write one sentence about its purpose.
Look for naming patterns:

| Directory pattern | Likely purpose |
|-------------------|---------------|
| `src/`, `lib/`, `app/` | Core source code |
| `tests/`, `spec/`, `__tests__/` | Test suite |
| `docs/`, `doc/` | Documentation |
| `scripts/`, `bin/`, `tools/` | Dev tooling / automation |
| `migrations/`, `db/` | Database schema evolution |
| `config/`, `settings/` | Configuration |
| `public/`, `static/`, `assets/` | Static files |
| `infra/`, `terraform/`, `k8s/`, `helm/` | Infrastructure as code |
| `proto/`, `schema/`, `openapi/` | API / data contracts |

### 3c. Identify architectural pattern

After reading entry points and directory structure, classify the architecture. Read
`references/patterns.md` for detailed descriptions and signal words for each pattern.

Common patterns: MVC · Layered (n-tier) · Hexagonal/Clean · Event-driven · Microservices ·
Monorepo with packages · Pipeline/ETL · Plugin-based

---

## Phase 4 — Core Logic Exploration

Now read actual code, but strategically. Budget: read **at most 5–8 files** in this phase.

**Selection priority:**
1. The main entry point file (identified in Phase 3a)
2. The largest/most-imported module (signals central domain logic)
3. One model/entity definition (understand the data shape)
4. One test file (understand expected behavior and testing style)
5. One configuration file (understand environment/deployment assumptions)

For each file, scan for:
- Exported functions / classes / types
- External service calls (HTTP, DB, queues)
- Major control flow branches
- TODO / FIXME / HACK comments (signals known problems)

```bash
# Find the most-imported internal modules (Python example)
grep -r "^from \|^import " <root> --include="*.py" | \
  grep -v "node_modules\|__pycache__" | \
  awk '{print $2}' | sort | uniq -c | sort -rn | head -20

# For JS/TS: find most-required files
grep -r "require\|from '" <root> --include="*.js" --include="*.ts" | \
  grep -v node_modules | awk -F"'" '{print $2}' | sort | uniq -c | sort -rn | head -20
```

---

## Phase 5 — Dev Workflow Discovery

Understanding how to actually *run* the project:

```bash
# Look for task runners / scripts
cat <root>/Makefile 2>/dev/null | grep "^[a-zA-Z]" | head -30
cat <root>/package.json 2>/dev/null | python3 -c \
  "import sys,json; d=json.load(sys.stdin); [print(k,':',v) for k,v in d.get('scripts',{}).items()]"
cat <root>/.github/workflows/*.yml 2>/dev/null | grep "run:\|name:" | head -40

# Environment setup
ls <root>/.env* <root>/docker-compose* <root>/Dockerfile* 2>/dev/null
cat <root>/.env.example 2>/dev/null || cat <root>/.env.sample 2>/dev/null

# CI configuration (reveals test/build/deploy steps)
ls <root>/.github/workflows/ <root>/.circleci/ <root>/.gitlab-ci.yml 2>/dev/null
```

Extract and summarize:
- How to install dependencies
- How to run tests
- How to start the development server
- How to build for production
- Required environment variables

---

## Phase 6 — Produce the Orientation Report

Write a structured report in this format. Aim for clarity over completeness — a developer
should be able to read it in under 5 minutes.

```markdown
# Codebase Orientation: <project-name>

## What is this?
[1–3 sentence plain-language description of what the project does and who uses it]

## Technology Stack
- **Language(s)**: ...
- **Framework(s)**: ...
- **Key dependencies**: ...
- **Infrastructure**: ...

## Architecture
[2–4 sentences on the architectural pattern and why it's structured that way]

## Directory Map
| Path | Purpose |
|------|---------|
| ...  | ...     |

## Entry Points
- **Main entry**: `path/to/main.ext` — [what it does]
- **HTTP server**: `path/to/server.ext` — starts on port X
- **CLI**: `path/to/cli.ext` — ...

## Core Concepts / Domain Model
[3–6 bullet points naming the key entities, services, or concepts in the codebase]

## Data Flow (if applicable)
[Brief description or ASCII diagram of how data moves through the system]

## Dev Workflow
```bash
# Install
...

# Run tests
...

# Start dev server
...
```

## Key Files to Read Next
1. `path/to/file` — [why it's important]
2. `path/to/file` — [why it's important]
3. `path/to/file` — [why it's important]

## Notable Observations
- [Anything surprising, concerning, or worth highlighting]
- [Known TODOs or tech debt signals]
- [Testing coverage / quality signals]
```

---

## Adaptive Depth

Scale the depth of exploration to the request:

| Signal | Depth |
|--------|-------|
| "quick overview" / "just the gist" | Phases 1–2 + summary only |
| "onboard me" / default | All 6 phases |
| "deep dive" / "explain the architecture in detail" | All phases + read more files in Phase 4 |
| "help me make a change to X" | All phases, then focus Phase 4 on the relevant subsystem |

---

## Tips for Better Onboarding

- **Git history is a goldmine**: `git log --oneline -20` and `git log --stat -5` reveal recent
  focus areas and churn hotspots.
- **Test names are documentation**: Skim test file names and `describe`/`it`/`test` blocks
  for a behavior inventory without reading implementation.
- **Search for the "seam"**: In most projects there's one file that wires everything together
  (often called `app.py`, `main.go`, `index.js`, `bootstrap.php`). Finding it unlocks 80%
  of the mental model.
- **Be honest about uncertainty**: If something is unclear after reasonable exploration, say
  so and suggest where the user might find the answer (ask a teammate, read specific docs).

---

## Reference Files

- `references/ecosystems.md` — Per-ecosystem manifest reading guide (Node, Python, Go, Rust…)
- `references/patterns.md` — Architectural pattern descriptions and signal words