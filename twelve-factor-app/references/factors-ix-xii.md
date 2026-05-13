# Factors IX–XII: Disposability, Dev/Prod Parity, Logs, Admin Processes

Source: https://12factor.net — by Adam Wiggins

---

## IX. Disposability — Maximize robustness with fast startup and graceful shutdown

**Reference**: https://12factor.net/disposability

### Core Rule
Processes are **disposable** — they can be started or stopped at any moment. This enables fast elastic scaling, rapid deploys, and robust production operation.

### Fast Startup
Processes should be ready to serve requests within **a few seconds** of launch. Benefits:
- Faster scaling (spin up new instances without lag)
- Faster deploys (new release processes come up quickly)
- Process manager can relocate processes to new machines easily

If startup is slow, investigate: lazy initialization, deferred connection pooling, async startup checks.

### Graceful Shutdown on SIGTERM
When the process manager sends `SIGTERM`, the process must shut down gracefully:

**For web processes**:
1. Stop listening on the service port (refuse new requests)
2. Allow in-flight requests to complete
3. Exit cleanly

Implicit constraint: HTTP requests must be **short** (a few seconds max). For long-polling, the client must handle reconnection gracefully.

**For worker processes**:
1. Return the current job to the queue (do not drop it)
2. Exit cleanly

```python
# Example: Celery worker handles SIGTERM by returning job to queue automatically
# Example: RabbitMQ — send NACK to return job
# Example: Beanstalkd — job auto-returns to queue on disconnect

import signal
import sys

def handle_sigterm(sig, frame):
    # Finish current work, then exit
    server.shutdown()
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

### Robustness Against Sudden Death
Hardware can fail. Processes must be architected to handle **unexpected termination** without data loss or corruption. Techniques:
- Use transactional job queues (Beanstalkd, SQS with visibility timeout, RabbitMQ with acks)
- Make jobs **idempotent** (safe to run twice)
- Wrap results in database transactions

### Idempotency Pattern
```python
def process_payment(payment_id):
    payment = db.get(payment_id)
    if payment.status == 'completed':
        return  # already done — safe to re-run
    # ... process ...
    payment.status = 'completed'
    db.save(payment)
```

### Crash-Only Design
The logical extreme: design the app so that the only way to stop it cleanly is to crash it. If crash recovery is robust, graceful shutdown and crash recovery converge to the same code path.

---

## X. Dev/Prod Parity — Keep development, staging, and production as similar as possible

**Reference**: https://12factor.net/dev-prod-parity

### Core Rule
The twelve-factor app is designed for **continuous deployment** by keeping the gap between development and production as small as possible across three dimensions.

### The Three Gaps

| Gap | Traditional | Twelve-Factor |
|-----|-------------|---------------|
| **Time gap** (code to deploy) | Weeks to months | Hours or minutes |
| **Personnel gap** (who deploys) | Ops team, not devs | Same people who wrote it |
| **Tools gap** (what runs it) | Different services in dev vs prod | Same services everywhere |

### Tools Gap: The Most Insidious
Developers are often tempted to use lightweight services locally:
- SQLite in dev vs PostgreSQL in prod
- In-process memory for sessions in dev vs Redis in prod
- Mock queue in dev vs RabbitMQ in prod

Even with good adapters, this creates subtle incompatibilities that only surface in production. **The twelve-factor developer resists this temptation.**

### The Modern Solution
Lightweight local alternatives are no longer necessary. Modern tooling makes parity easy:

```yaml
# docker-compose.yml — runs exact same versions as production
services:
  db:
    image: postgres:16
  redis:
    image: redis:7
  rabbitmq:
    image: rabbitmq:3-management
```

Use the same Docker images in CI and production as in local development.

### Continuous Deployment Flow
```
commit → CI build (minutes) → staging deploy (auto) → production deploy (auto or 1-click)
```
The developer who wrote the code monitors its behavior in production. Fast feedback loops surface issues immediately.

### Adapter Caution
ORMs and adapter libraries (ActiveRecord, SQLAlchemy, Eloquent) abstract over backing services but do **not** eliminate incompatibilities:
- PostgreSQL has different constraint behavior than SQLite
- MySQL handles `TIMESTAMP` differently than PostgreSQL
- Redis data structures have no equivalent in in-memory mocks

All deploys (dev, staging, prod) must use the **same type and version** of each backing service.

---

## XI. Logs — Treat logs as event streams

**Reference**: https://12factor.net/logs

### Core Rule
A twelve-factor app **never concerns itself with routing or storage of its output stream**. It writes all log output, **unbuffered**, to **stdout**. Nothing else.

### Logs Are Streams, Not Files
Logs are the time-ordered stream of events from all running processes. They have no fixed beginning or end — they flow continuously as the app operates. Thinking of logs as files to be written and rotated is the wrong mental model.

### What the App Does
```python
import sys
import logging

logging.basicConfig(
    stream=sys.stdout,        # stdout — that's it
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)

logger = logging.getLogger(__name__)
logger.info("Request received", extra={"path": "/api/users", "status": 200})
```

The app writes to stdout. That's all. It does **not**:
- Open log files
- Rotate log files
- Configure log destinations
- Know where logs end up

### What the Execution Environment Does
The execution environment captures stdout from each process, collates streams from all processes, and routes to one or more destinations:
- **Terminal** (local dev): developer sees logs in foreground
- **File** (if needed): redirected by the platform
- **Log aggregator**: Logplex (Heroku), Fluentd, Logstash, Vector
- **Indexing/analysis**: Splunk, Datadog, Elasticsearch/Kibana, Loki/Grafana

### Structured Logging (Best Practice)
JSON-structured logs make downstream analysis far more powerful:
```python
import json, sys
print(json.dumps({
    "time": "2024-03-15T14:32:00Z",
    "level": "info",
    "event": "request_complete",
    "method": "GET",
    "path": "/api/users",
    "status": 200,
    "duration_ms": 42
}), file=sys.stdout, flush=True)
```

### What Good Log Infrastructure Enables
- Find specific past events by querying structured fields
- Graph trends over time (requests/min, error rates)
- Alert on thresholds (errors/min > 100 → page on-call)
- Correlate events across multiple services via trace IDs

### Anti-Patterns to Avoid
```python
# Violation: writing to a file
with open('/var/log/myapp.log', 'a') as f:
    f.write(message)

# Violation: buffered stdout (flush=False loses logs on crash)
print(message)  # may be buffered — use flush=True or PYTHONUNBUFFERED=1

# Violation: routing to syslog directly from app code
import syslog
syslog.syslog(message)  # app should not know about syslog
```

---

## XII. Admin Processes — Run admin/management tasks as one-off processes

**Reference**: https://12factor.net/admin-processes

### Core Rule
One-off administrative tasks run as **isolated, one-off processes** in the **same environment** as the regular app processes — same release, same codebase, same config, same dependency isolation.

### What Admin Processes Include
- **Database migrations**: `manage.py migrate`, `rake db:migrate`, `alembic upgrade head`, `flyway migrate`
- **REPL/console**: `rails console`, `python`, `node`, `psql` connected to the production DB via the app's credentials
- **One-time fix scripts**: data backfills, record repairs, cache warm-ups committed to the app repo
- **Seeding**: initial data population in a fresh environment

### Key Requirements

**Same release**: Admin processes run against the current release, with the same build and config as the running web/worker processes. No running migrations against last week's schema with this week's code.

**Same dependency isolation**: Use the app's own dependency environment:
```bash
# Violation: using system ruby/python
rake db:migrate
python scripts/fix_data.py

# Correct: using bundled/isolated environment
bundle exec rake db:migrate
./venv/bin/python scripts/fix_data.py
# Or in Docker:
docker run --env-file .env myapp:v247 bundle exec rake db:migrate
```

**Ship admin code with app code**: Admin scripts must live in the same repo as the app to avoid version skew. Never maintain a separate "ops repo" for migration scripts.

### REPL Languages Preferred
Twelve-factor strongly favors languages with an interactive REPL shell (`irb`, `python`, `node`, `iex`) for easy one-off investigation of production state.

### Execution in Different Environments

**Local development**:
```bash
# Direct shell command in checkout directory
bundle exec rake db:migrate
python manage.py createsuperuser
```

**Production**:
```bash
# Heroku
heroku run bundle exec rake db:migrate

# Kubernetes — run a one-off pod with the current image
kubectl run migration --image=myapp:v247 --restart=Never -- bundle exec rake db:migrate

# SSH (classic)
ssh prod-server "cd /app && bundle exec rake db:migrate"

# Docker
docker run --rm --env-file prod.env myapp:v247 bundle exec rake db:migrate
```

### CI/CD Integration
Migrations are commonly run as a step in the deployment pipeline, after the release is built but before traffic switches to new processes:
```yaml
# GitHub Actions example
- name: Run migrations
  run: docker run --rm $IMAGE bundle exec rake db:migrate
  env:
    DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}

- name: Deploy
  run: kubectl apply -f deploy/
```

This satisfies the "same release" requirement: the migration uses the same image (`$IMAGE`) as what will be deployed.
