# Factors I–IV: Codebase, Dependencies, Config, Backing Services

Source: https://12factor.net — by Adam Wiggins

---

## I. Codebase — One codebase tracked in revision control, many deploys

**Reference**: https://12factor.net/codebase

### Core Rule
There is always a **one-to-one correlation** between the codebase and the app. One app = one repo (or one root commit in a decentralized VCS like Git).

### Key Distinctions

**Multiple codebases → not an app, it's a distributed system.**
Each component in a distributed system is its own app and should independently comply with twelve-factor.

**Multiple apps sharing one codebase → violation.**
The fix: factor shared code into libraries that are included via the dependency manager (Factor II).

### Deploys
A *deploy* is a running instance of the app. There are many deploys of the same app: production, one or more staging environments, every developer's local environment. All run from the **same codebase**, but may run different versions (commits).

### Practical Guidance
- Use a single repo per microservice if you split a monolith
- Monorepos are acceptable if each service has its own independent deploy pipeline
- Tag or version releases; don't branch-per-environment (that creates multiple effective codebases)
- `git tag v1.2.3` or timestamp-based release IDs are the right pattern

---

## II. Dependencies — Explicitly declare and isolate dependencies

**Reference**: https://12factor.net/dependencies

### Core Rule
A twelve-factor app **never relies on the implicit existence of system-wide packages**. All dependencies are declared completely and exactly in a *dependency declaration manifest* and isolated at runtime.

Both declaration **and** isolation are required. Neither alone is sufficient.

### By Language

| Language | Declaration | Isolation |
|----------|-------------|-----------|
| Python | `requirements.txt` / `pyproject.toml` | `virtualenv` / `venv` |
| Ruby | `Gemfile` | `bundle exec` |
| Node.js | `package.json` | `node_modules` (local) |
| Go | `go.mod` | module system |
| Java | `pom.xml` / `build.gradle` | Maven/Gradle local repo |
| Rust | `Cargo.toml` | Cargo workspace |
| C | `Autoconf` | static linking |

### System Tools
The app must not assume the existence of system tools like `curl`, `ImageMagick`, `ffmpeg`, etc. If needed, vendor them into the app or declare them as a dependency.

### Benefits
- New developer setup is a single deterministic build command (e.g., `pip install -r requirements.txt`, `bundle install`, `npm ci`)
- No "works on my machine" failures from ambient system state
- Reproducible builds across environments (Factor X)

### Lockfiles
Always commit lockfiles (`package-lock.json`, `Pipfile.lock`, `Gemfile.lock`, `Cargo.lock`). They pin exact transitive versions for deterministic builds.

---

## III. Config — Store config in the environment

**Reference**: https://12factor.net/config

### Core Rule
Config is everything that **varies between deploys** (staging, production, developer environments). It must be stored in **environment variables**, not in the code.

### What Is Config?
- Resource handles: database URLs, cache addresses
- Credentials: API keys, OAuth tokens, passwords
- Per-deploy values: canonical hostname, feature flags, port numbers

### What Is NOT Config (per twelve-factor)?
Internal application config that does NOT vary between deploys — e.g., routing tables (`config/routes.rb`), framework wiring (Spring beans), module structure. This can and should remain in code.

### The Litmus Test
> "Could the codebase be made open source at any moment, without compromising any credentials?"

If yes → config is properly externalized. If no → there are credentials in the code.

### Why Not Config Files?
Config files not checked into the repo (e.g., `config/database.yml`) are better than hardcoded constants, but still have problems:
1. Easy to accidentally commit to the repo
2. Tend to scatter into multiple files in different formats
3. Language/framework-specific — not portable

### Why Not Named Environment Groups?
Grouping env vars into named environments (`development`, `test`, `production`) doesn't scale. Adding a new deploy type (e.g., `staging`, `joes-staging`) requires new named groups, creating a combinatorial explosion.

**The twelve-factor way**: each env var is an independent, orthogonal control. Each deploy manages its own set independently.

### Practical Patterns
```bash
# Local development — use a .env file (never committed)
DATABASE_URL=postgres://localhost/myapp_dev
REDIS_URL=redis://localhost:6379
SECRET_KEY=dev-only-secret-not-real

# .gitignore must include:
.env
```

Tools: `dotenv` (Ruby/Node), `python-dotenv`, `direnv`, Docker `--env-file`, Kubernetes Secrets, AWS SSM Parameter Store, HashiCorp Vault.

---

## IV. Backing Services — Treat backing services as attached resources

**Reference**: https://12factor.net/backing-services

### Core Rule
A *backing service* is any service the app consumes over the network as part of normal operation. The twelve-factor app makes **no distinction between local and third-party services** — both are attached resources accessed via URL/credentials from config.

### Examples of Backing Services
- **Datastores**: MySQL, PostgreSQL, CouchDB, MongoDB, SQLite (local only — not recommended in prod)
- **Messaging/queues**: RabbitMQ, Beanstalkd, Amazon SQS, Kafka
- **SMTP**: Postfix (local), Postmark, SendGrid (third-party)
- **Caches**: Memcached, Redis
- **Metrics**: New Relic, Datadog, Loggly
- **Blob storage**: Amazon S3, Google Cloud Storage
- **APIs**: Twitter, Google Maps, Stripe, Twilio

### Attached Resources
Each backing service is a *resource* identified by a URL or handle in config. Swapping a local PostgreSQL for Amazon RDS requires **only a config change** — no code changes.

```bash
# Before (local)
DATABASE_URL=postgres://localhost/myapp

# After (Amazon RDS) — config change only, zero code changes
DATABASE_URL=postgres://user:pass@myapp.abc123.us-east-1.rds.amazonaws.com/myapp
```

### Loose Coupling
Resources can be attached and detached from deploys at will. If a database is misbehaving, an administrator can spin up a database from a recent backup, detach the old one, and attach the new one — all without touching code.

### Multiple Instances = Multiple Resources
Two MySQL databases (e.g., for sharding) count as two distinct resources. Each gets its own URL in config:
```bash
DB_PRIMARY_URL=postgres://...
DB_REPLICA_URL=postgres://...
```

### Composability
Port binding (Factor VII) means one app can be a backing service for another. App B's URL is stored in App A's config as a resource handle.
