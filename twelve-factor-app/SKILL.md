---
name: twelve-factor-app
description: Apply The Twelve-Factor App methodology when designing, reviewing, building, or advising on software-as-a-service (SaaS) applications. Use this skill whenever the user asks about app architecture, cloud-native design, microservices, deployment best practices, environment configuration, scaling, logging, containerization, CI/CD pipelines, or when reviewing code/config for cloud readiness. Also trigger when users mention Heroku, Docker, Kubernetes, environment variables, stateless services, health of deployment pipelines, or any "12-factor" / "twelve-factor" by name. This skill ensures advice is grounded in the authoritative twelve-factor methodology from https://12factor.net.
---

# The Twelve-Factor App Skill

A comprehensive guide for applying the Twelve-Factor App methodology — the industry-standard set of principles for building modern, portable, scalable SaaS applications, authored by Adam Wiggins and published at https://12factor.net.

## Overview

The twelve factors address three core concerns:
- **Portability** — apps that run the same everywhere
- **Continuous deployment** — safe and frequent releases
- **Scalability** — horizontal scale without re-architecture

The methodology applies to any language or backing service combination.

## The Twelve Factors (Quick Reference)

| # | Factor | Principle |
|---|--------|-----------|
| I | [Codebase](#i-codebase) | One codebase, many deploys |
| II | [Dependencies](#ii-dependencies) | Explicitly declare and isolate |
| III | [Config](#iii-config) | Store config in environment variables |
| IV | [Backing Services](#iv-backing-services) | Treat as attached resources |
| V | [Build, Release, Run](#v-build-release-run) | Strictly separate stages |
| VI | [Processes](#vi-processes) | Stateless, share-nothing |
| VII | [Port Binding](#vii-port-binding) | Self-contained, export via port |
| VIII | [Concurrency](#viii-concurrency) | Scale out via the process model |
| IX | [Disposability](#ix-disposability) | Fast startup, graceful shutdown |
| X | [Dev/Prod Parity](#x-devprod-parity) | Keep environments as similar as possible |
| XI | [Logs](#xi-logs) | Treat as event streams to stdout |
| XII | [Admin Processes](#xii-admin-processes) | One-off tasks run as processes |

---

## How to Use This Skill

**Reviewing an app or codebase**: Check each factor systematically. Identify violations and provide concrete remediation advice.

**Designing a new app**: Use the factors as a checklist during architecture decisions.

**Explaining a factor**: Load the relevant reference file from `references/` for authoritative detail and examples.

**Answering "is my app twelve-factor compliant?"**: Walk through each factor, ask clarifying questions where needed, and report findings.

For deep guidance on any individual factor, read:
- `references/factors-i-iv.md` — Codebase, Dependencies, Config, Backing Services
- `references/factors-v-viii.md` — Build/Release/Run, Processes, Port Binding, Concurrency
- `references/factors-ix-xii.md` — Disposability, Dev/Prod Parity, Logs, Admin Processes

---

## Summary of Each Factor

### I. Codebase
One codebase tracked in version control; many deploys from it. Multiple apps must not share one repo — extract shared code into a library. Every deploy (dev, staging, prod) runs from the same codebase, just different versions. **Source**: https://12factor.net/codebase

### II. Dependencies
Never rely on implicit system-wide packages. Declare all dependencies in a manifest (e.g., `requirements.txt`, `Gemfile`, `package.json`) and use an isolation tool (virtualenv, bundler, npm) so nothing leaks in from the host. **Source**: https://12factor.net/dependencies

### III. Config
Anything that varies between deploys (DB URLs, API keys, hostnames) must live in **environment variables**, not in code or config files committed to the repo. The litmus test: could the codebase be open-sourced right now without leaking credentials? **Source**: https://12factor.net/config

### IV. Backing Services
Databases, queues, caches, SMTP — all are "attached resources" accessed via URL/credentials from config. Local and third-party services are interchangeable; swapping one for another requires only a config change, not a code change. **Source**: https://12factor.net/backing-services

### V. Build, Release, Run
Three strictly separate pipeline stages: **Build** (compile + fetch deps), **Release** (build + config, immutable, gets a unique ID), **Run** (execute processes against a release). Code changes at runtime are impossible by design. **Source**: https://12factor.net/build-release-run

### VI. Processes
Run the app as one or more **stateless, share-nothing** processes. All persistent state lives in a backing service (database, cache). No sticky sessions. No in-process memory assumed to persist across requests. **Source**: https://12factor.net/processes

### VII. Port Binding
The app is **self-contained** — it exports HTTP (or any protocol) by binding to a port, without relying on an injected web server container. A webserver library (Tornado, Puma, Jetty) is declared as a dependency. **Source**: https://12factor.net/port-binding

### VIII. Concurrency
Scale out by running **more processes**, not by making processes bigger. Assign work types to process types (web workers, background workers). Use the OS process manager (systemd, Foreman) rather than daemonizing. **Source**: https://12factor.net/concurrency

### IX. Disposability
Processes are **disposable**: fast startup (seconds), graceful shutdown on SIGTERM (finish current requests/jobs, then exit), and resilience to sudden death. Jobs must be reentrant or idempotent. **Source**: https://12factor.net/disposability

### X. Dev/Prod Parity
Minimize the gap across three dimensions: **time** (deploy frequently), **people** (devs own deploys), **tools** (same backing services in dev and prod — resist SQLite-in-dev/Postgres-in-prod). **Source**: https://12factor.net/dev-prod-parity

### XI. Logs
Never write or manage logfiles. Each process writes its event stream **unbuffered to stdout**. The execution environment routes streams to their final destination (Splunk, Loggly, Fluentd, etc.). The app is unaware of log storage. **Source**: https://12factor.net/logs

### XII. Admin Processes
One-off tasks (DB migrations, console inspection, fix scripts) run as **one-off processes** in the same environment, same release, same codebase, same dependency isolation as the regular app. Ship admin code with app code. **Source**: https://12factor.net/admin-processes

---

## Common Violations & Quick Fixes

| Violation | Factor | Fix |
|-----------|--------|-----|
| API key hardcoded in source | III | Move to env var; use dotenv locally |
| `requirements.txt` missing versions | II | Pin versions; add lockfile |
| Sessions stored in-process memory | VI | Move to Redis/Memcached |
| `console.log` to file in `/var/log/app.log` | XI | Write to stdout; let infra route it |
| SQLite in dev, PostgreSQL in prod | X | Use PostgreSQL everywhere; use Docker locally |
| Migration run manually on prod server | XII | Automate as a one-off process in CI/CD |
| Shared monorepo for 3 microservices | I | Split into separate repos (or use a monorepo with independent deploys per service) |
| `config/database.yml` committed to repo | III | Move credentials to env vars |
| App bundles Apache/Nginx internally as a module dependency | VII | Embed a library webserver; bind to a port |
| Cronjob writing PID file + daemonizing | VIII | Use a process manager; write to stdout |

---

## Compliance Checklist

Use this when auditing an existing application:

- [ ] **I** — Single git repo per deployable unit; shared code in libraries
- [ ] **II** — All dependencies declared in manifest; no reliance on system tools
- [ ] **III** — No secrets or environment-specific values in code or committed files
- [ ] **IV** — All backing services swappable via config change alone
- [ ] **V** — Build, release, and run are separate pipeline stages; releases are immutable
- [ ] **VI** — No sticky sessions; no in-memory state assumed to survive a restart
- [ ] **VII** — App self-starts and binds to a port; no external server injection needed
- [ ] **VIII** — Horizontal scaling works by adding process instances
- [ ] **IX** — Startup < a few seconds; graceful SIGTERM handling; idempotent jobs
- [ ] **X** — Same backing service types/versions in dev, staging, and production
- [ ] **XI** — All logging to stdout; no logfile management in app code
- [ ] **XII** — Admin tasks run as isolated one-off processes against the current release

---

*Source: The Twelve-Factor App by Adam Wiggins — https://12factor.net — last updated 2017, published by Salesforce/Heroku.*
