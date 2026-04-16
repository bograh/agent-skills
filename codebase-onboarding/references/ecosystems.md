# Ecosystem Reference Guide

Read this file during Phase 2 of codebase onboarding to understand what to extract
from each ecosystem's manifest files.

---

## Node.js / JavaScript / TypeScript

**Key files**: `package.json`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`,
`tsconfig.json`, `.nvmrc`, `.node-version`

**What to extract from `package.json`**:
- `name`, `version`, `description` — project identity
- `main` / `module` / `exports` — library entry points
- `bin` — CLI commands
- `scripts` — all tasks (test, build, start, dev, lint, etc.)
- `dependencies` — runtime deps; look for framework (express, fastify, next, nuxt, nest,
  react, vue, svelte, etc.)
- `devDependencies` — tooling; look for bundler (vite, webpack, esbuild, rollup),
  test runner (jest, vitest, mocha, playwright), linter (eslint, biome)
- `workspaces` — signals monorepo

**TypeScript signals**:
- `tsconfig.json` present → TypeScript project
- Check `compilerOptions.strict`, `target`, `paths` (alias map), `outDir`

**Common framework signals**:
| Dependency | Framework type |
|-----------|----------------|
| `next` | React SSR/full-stack |
| `nuxt` | Vue SSR/full-stack |
| `express` / `fastify` / `koa` | Node HTTP server |
| `nestjs` | Opinionated Node MVC |
| `react` | UI library |
| `vue` / `svelte` | UI framework |
| `electron` | Desktop app |
| `@remix-run/node` | Full-stack React |

---

## Python

**Key files**: `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements.txt`,
`requirements-dev.txt`, `Pipfile`, `poetry.lock`, `.python-version`

**What to extract**:
- `pyproject.toml [project]` → name, version, dependencies (modern standard)
- `pyproject.toml [tool.poetry]` → same if using Poetry
- `setup.py` / `setup.cfg` → legacy packaging; check `install_requires`
- `requirements.txt` → flat dep list; look for framework and major libs
- `[tool.pytest.ini_options]` or `pytest.ini` → test configuration

**Common framework signals**:
| Package | Type |
|---------|------|
| `django` | Full-stack web |
| `flask` / `bottle` | Micro web |
| `fastapi` / `starlette` | Async web API |
| `celery` | Task queue |
| `sqlalchemy` / `tortoise-orm` | ORM |
| `alembic` | DB migrations |
| `pydantic` | Data validation / settings |
| `click` / `typer` | CLI |
| `pandas` / `numpy` / `scikit-learn` | Data science / ML |
| `torch` / `tensorflow` | Deep learning |
| `pytest` | Test framework |
| `mypy` / `pyright` | Type checker |

---

## Go

**Key files**: `go.mod`, `go.sum`

**What to extract from `go.mod`**:
- Module path (tells you the import root and often org/project name)
- `go` directive → minimum Go version
- `require` blocks → dependencies
  - Direct (your deps) vs indirect (transitive)

**Common framework/library signals**:
| Package prefix | Type |
|----------------|------|
| `github.com/gin-gonic/gin` | HTTP framework |
| `github.com/labstack/echo` | HTTP framework |
| `google.golang.org/grpc` | gRPC |
| `gorm.io/gorm` | ORM |
| `github.com/spf13/cobra` | CLI |
| `github.com/spf13/viper` | Config |
| `go.uber.org/zap` | Logging |
| `github.com/stretchr/testify` | Test assertions |

**Entry point**: `main` package; look for `cmd/` directory (common pattern for multi-binary
repos using `cmd/<name>/main.go`).

---

## Rust

**Key files**: `Cargo.toml`, `Cargo.lock`

**What to extract from `Cargo.toml`**:
- `[package]` → name, version, edition (2021 is current)
- `[[bin]]` → binary targets (executable entry points)
- `[lib]` → library crate signals
- `[dependencies]` → runtime deps
- `[dev-dependencies]` → test-only deps
- `[workspace]` → signals monorepo / multiple crates

**Common crate signals**:
| Crate | Type |
|-------|------|
| `tokio` / `async-std` | Async runtime |
| `actix-web` / `axum` / `warp` | Web framework |
| `serde` / `serde_json` | Serialization |
| `sqlx` / `diesel` | DB access |
| `clap` | CLI argument parsing |
| `tracing` | Logging / instrumentation |
| `anyhow` / `thiserror` | Error handling |

---

## Java / Kotlin / JVM

**Key files**: `pom.xml` (Maven), `build.gradle` / `build.gradle.kts` (Gradle),
`settings.gradle` (multi-module)

**Maven (`pom.xml`)**:
- `<groupId>`, `<artifactId>`, `<version>` — coordinates
- `<dependencies>` — look for Spring, Hibernate, etc.
- `<plugins>` — build lifecycle

**Gradle (`build.gradle`)**:
- `plugins {}` block → framework (spring-boot, android, etc.)
- `dependencies {}` → runtime vs testImplementation

**Common framework signals**:
| Dep | Type |
|-----|------|
| `spring-boot` | Full-stack / microservices |
| `spring-web` / `spring-webflux` | HTTP (reactive vs servlet) |
| `hibernate` / `spring-data-jpa` | ORM |
| `quarkus` / `micronaut` | Cloud-native frameworks |
| `android` | Android app |
| `junit` / `kotest` | Testing |

---

## Ruby

**Key files**: `Gemfile`, `Gemfile.lock`, `.ruby-version`, `Rakefile`

**What to extract from `Gemfile`**:
- `ruby` version constraint
- `gem` declarations, especially `rails` version
- Groups (`development`, `test`, `production`)

**Common signals**:
| Gem | Type |
|-----|------|
| `rails` | Full-stack web |
| `sinatra` / `hanami` | Lightweight web |
| `sidekiq` / `resque` | Background jobs |
| `devise` | Authentication |
| `rspec` / `minitest` | Testing |
| `rubocop` | Linting |

---

## PHP

**Key files**: `composer.json`, `composer.lock`

**What to extract**:
- `require` → runtime deps
- `require-dev` → dev deps
- `autoload` → namespace/directory mapping (reveals structure)
- `scripts` → task shortcuts

**Common signals**:
| Package | Type |
|---------|------|
| `laravel/framework` | Full-stack |
| `symfony/*` | Component/full-stack |
| `slim/slim` | Micro framework |
| `doctrine/*` | ORM |
| `phpunit/phpunit` | Testing |

---

## .NET / C#

**Key files**: `*.csproj`, `*.sln`, `NuGet.config`, `global.json`

**What to extract from `*.csproj`**:
- `<TargetFramework>` → .NET version (net8.0, net6.0, netstandard2.0…)
- `<PackageReference>` → dependencies
- `<OutputType>Exe` → executable vs library

**Common signals**:
| Package | Type |
|---------|------|
| `Microsoft.AspNetCore.*` | Web API / MVC |
| `Microsoft.EntityFrameworkCore` | ORM |
| `Blazor` | WASM / Server UI |
| `xunit` / `NUnit` / `MSTest` | Testing |
| `Serilog` / `NLog` | Logging |
