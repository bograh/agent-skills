# Factors V–VIII: Build/Release/Run, Processes, Port Binding, Concurrency

Source: https://12factor.net — by Adam Wiggins

---

## V. Build, Release, Run — Strictly separate build and run stages

**Reference**: https://12factor.net/build-release-run

### Core Rule
A codebase is transformed into a deploy through exactly **three separate, strictly ordered stages**. No mixing, no going backwards.

### The Three Stages

**1. Build stage**
Transforms the code repo into an executable bundle (the *build*):
- Checks out a specific commit
- Fetches vendor dependencies (Factor II)
- Compiles binaries and assets

**2. Release stage**
Combines the build with the deploy's current config (Factor III) to produce a *release*:
- Release = Build + Config
- Every release gets a **unique, immutable release ID** (e.g., timestamp `2024-03-15T14:32:00Z` or incrementing integer `v247`)
- Releases are an append-only ledger — a release cannot be mutated once created
- Any change (code or config) must produce a new release

**3. Run stage** ("runtime")
Runs the app in the execution environment by launching processes against a selected release:
- Starts the app's process(es)
- Can happen automatically (server reboot, crashed process restart)
- No developer may be present

### Why Strict Separation Matters
- It is **impossible to change code at runtime** because there is no path back to the build stage
- The run stage has fewer moving parts → failures at 3 AM are recoverable
- The build stage can tolerate more complexity because a developer is always present to fix issues
- Rollbacks become trivial: just activate a previous release ID

### Tooling Examples
- **CI/CD**: GitHub Actions, CircleCI, GitLab CI produce builds
- **Release management**: Capistrano's `releases/` subdirectory; Heroku slugs; container image tags
- **Runtime**: systemd, Kubernetes Deployments, Heroku dynos, AWS ECS tasks

### Immutable Infrastructure Pattern
Container images are the natural embodiment of a twelve-factor release:
```
git commit abc123
  → docker build → image:abc123-v247   (build)
  → + env vars/secrets injected        (release)
  → kubectl apply / docker run         (run)
```

---

## VI. Processes — Execute the app as one or more stateless processes

**Reference**: https://12factor.net/processes

### Core Rule
**Twelve-factor processes are stateless and share-nothing.** Any data that needs to persist must be stored in a stateful backing service (Factor IV), typically a database.

### What "Stateless" Means
- No local file storage assumed to persist across requests
- No in-memory state assumed to survive a restart or be shared across process instances
- A future request may be served by a completely different process instance

### Memory and Filesystem
These may be used as a **brief, single-transaction cache only**:
- Download a large file → process it → store results in DB → done
- The next request cannot assume that downloaded file is still there

### Asset Compilation
Asset packagers (webpack, Rails asset pipeline, etc.) should compile assets during the **build stage** (Factor V), not at runtime. Twelve-factor apps never do on-the-fly compilation and cache results on the local filesystem.

### Sticky Sessions: A Violation
Sticky sessions (routing a user's requests to the same process to reuse in-memory session state) are **explicitly forbidden**. Session data belongs in a time-expiring datastore:
- Redis (`express-session` with Redis store, Django's cache session backend)
- Memcached
- Database-backed sessions

```python
# Violation: in-memory session
sessions = {}  # dies on restart, not shared across workers

# Correct: Redis-backed session
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'  # points to Redis via CACHE_URL env var
```

### Practical Impact
- Enables horizontal scaling (Factor VIII) — any process can serve any request
- Enables disposability (Factor IX) — any process can die without data loss
- Enables dev/prod parity (Factor X) — single-process dev works the same as multi-process prod

---

## VII. Port Binding — Export services via port binding

**Reference**: https://12factor.net/port-binding

### Core Rule
The twelve-factor app is **completely self-contained**. It does not rely on runtime injection of a web server (e.g., Apache module, Tomcat container) to become a web-facing service. Instead, it **exports HTTP as a service by binding to a port** using a library-level web server declared as a dependency.

### Self-Contained Web Server
The web server is declared as a dependency (Factor II) and embedded in the app:

| Language | Library |
|----------|---------|
| Python | Gunicorn, Uvicorn, Tornado |
| Ruby | Puma, Thin, Unicorn |
| Node.js | `http.createServer` / Express built-in |
| Java/JVM | Jetty, Undertow (embedded) |
| Go | `net/http` standard library |
| Rust | Axum, Actix |

### How It Works
```bash
# Local development
PORT=5000 python app.py
# → binds to localhost:5000, developer visits http://localhost:5000

# Production
PORT=8080 python app.py
# → binds to 0.0.0.0:8080, routing layer forwards traffic
```

The **routing layer** (load balancer, reverse proxy, Kubernetes Ingress) maps public hostnames to port-bound processes. The app is unaware of public hostnames.

### Beyond HTTP
Port binding works for any protocol:
- Redis speaks the Redis protocol on port 6379
- ejabberd speaks XMPP on port 5222
- A gRPC service binds to a port and speaks gRPC

### App-as-Backing-Service
Port binding enables composability (Factor IV): App B can use App A as a backing service simply by storing App A's URL in App B's config.

### What to Avoid
```python
# Violation: assumes Apache mod_wsgi injects the WSGI environment
application = create_app()  # depends on Apache to run it

# Correct: app binds to a port itself
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ['PORT']))
```

---

## VIII. Concurrency — Scale out via the process model

**Reference**: https://12factor.net/concurrency

### Core Rule
In a twelve-factor app, **processes are a first-class citizen**. The app scales horizontally by running more processes, not by making processes larger. Each type of work is assigned to a *process type*.

### Process Types
Different workloads are handled by different process types:
- **web**: handles incoming HTTP requests
- **worker**: handles background jobs from a queue
- **clock**: runs scheduled tasks (cron-like)
- **release**: runs DB migrations at deploy time (one-off — Factor XII)

The array of process types and the count of each is the *process formation*:
```
web=2 worker=4 clock=1
```

### The Unix Process Model
Twelve-factor takes strong cues from the Unix process model for daemons. Each process type maps to a Unix process; the OS manages lifecycle. No internal thread management required at the app layer (though threads within a process are fine).

### Internal Multiplexing Is Acceptable
An individual process may use threads, coroutines, or async I/O internally (EventMachine, asyncio, Node.js event loop). But this is a local optimization — horizontal scale still requires multiple process instances.

### Scaling Is Simply Adding Processes
Because processes are stateless (Factor VI) and share-nothing, scaling is:
```bash
# Heroku
heroku ps:scale web=5 worker=10

# Kubernetes
kubectl scale deployment web --replicas=5
kubectl scale deployment worker --replicas=10
```
No architectural changes required.

### No Daemonizing
Twelve-factor processes **must not**:
- Daemonize themselves (`daemon(3)`, double-fork)
- Write PID files

Instead, rely on the OS process manager:
- **Development**: Foreman (`Procfile`), Overmind, `docker compose`
- **Production**: systemd, supervisord, Kubernetes, Heroku dynos

```
# Procfile (development + production)
web: gunicorn app:application --bind 0.0.0.0:$PORT
worker: celery -A tasks worker --loglevel=info
clock: celery -A tasks beat --loglevel=info
```

### Output Streams
Processes output logs to stdout (Factor XI). The process manager routes those streams.
