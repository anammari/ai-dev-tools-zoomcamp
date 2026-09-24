# Homework 3 — Answers

<!-- Fill in your answers below as you complete the homework. -->

## Question 1 — Understand the project

**Which description matches the project's architecture?**

> **Agents claim tasks from a DB through an HTTP API.**

Run: `uv sync` then `uv run uvicorn main:app --reload`; dashboard at
`http://127.0.0.1:8000/` (token-gated), Swagger at `/docs`.

## Question 2 — Register agents and test the task flow

**Which task status does the sender see after the recipient submits its result?**

> **`completed`**

Turned SPEC acceptance scenario 1 (register two agents → send → claim →
complete → sender reads the result) into an API integration test against the
real API + SQLite DB: `test_acceptance_register_send_claim_complete_result` in
`test_agent_relay.py`. It asserts the task transitions `queued → completed`
with the output visible to the sender. Suite confirms green:

```bash
uv run pytest -q
# 5 passed
```

## Question 3 — Containerization

**Which Docker option publishes a container's port to your machine?**

> **`-p`** (e.g. `-p 8000:8000`). `--expose` only documents a port; it does not
> publish it.

Single-stage `Dockerfile` (pure Python + static `dashboard.html`, no Node build
step). Key lines:

- `FROM python:3.11-slim`; install deps via `uv sync --frozen --no-dev`.
- Copy the app modules and `dashboard.html`.
- `CMD [".venv/bin/uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]`
  — `--host 0.0.0.0` is required inside the container, otherwise the published
  `-p` looks broken (uvicorn defaults to `127.0.0.1`).

Build and run (stopped the earlier host `uvicorn`, kept the DB in a named
volume so it survives restarts):

```bash
docker build -t agent-relay:local .
docker run -d --rm --name agent-relay -p 8000:8000 \
  -v agent-relay-data:/data -e RELAY_DATABASE_URL=sqlite:////data/agent-relay.db \
  agent-relay:local
```

Confirmed `/health` + `/ready` on `http://127.0.0.1:8000`, then repeated the Q2
flow against the containerized API: register two agents → send → claim →
complete → sender reads the result (`status: completed`, output visible).

## Question 4 — Docker Compose and PostgreSQL

**Which hostname should the API use to connect to the `postgres` service in Docker Compose?**

> **`postgres`** — the Compose **service name**, not `localhost`. Services reach
> each other by service name on Compose's internal network.

`compose.yaml` runs `postgres` (with a `pg_isready` healthcheck that `app`
waits on via `depends_on.condition: service_healthy`) and `app` (built from the
Dockerfile). The app connects with:

```yaml
DATABASE_URL: postgresql+psycopg://relay:relay@postgres:5432/relay
```

Two gotchas hit while switching SQLite → Postgres:
1. **psycopg v3**: this project depends on `psycopg[binary]`, so the URL scheme
   must be `postgresql+psycopg://` (plain `postgresql://` defaults to psycopg2
   in SQLAlchemy 2 → `ModuleNotFoundError: No module named 'psycopg2'`).
2. **Storage seam**: `database.py`'s `immediate_transaction` hard-coded SQLite's
   `BEGIN IMMEDIATE`, which Postgres rejects (`syntax error at or near
   "IMMEDIATE"`). Made it dialect-aware: keep `BEGIN IMMEDIATE` on SQLite, use a
   normal SQLAlchemy write transaction on Postgres (full `FOR UPDATE SKIP LOCKED`
   row-locking stays the documented student exercise).

Stack: `docker compose up --build`. Q2 integration test run against the real
Postgres:
```bash
RELAY_DATABASE_URL=postgresql+psycopg://relay:relay@localhost:5432/relay \
  uv run pytest test_agent_relay.py -k accept -v   # 1 passed
```
(The full SQLite suite still passes — the seam keeps `BEGIN IMMEDIATE` there.)

## Question 5 — Deploy to Kubernetes

**Which Kubernetes resource keeps the requested number of application replicas running and manages updates?**

> **Deployment** (the others are a load-balancer / config holder / secret store,
> not replicas-and-rollouts).

Deployed to a local kind cluster (`kind` v0.33.0, node `kind-control-plane`
Ready). Manifests in `k8s/`:

- `k8s/postgres.yaml` — Deployment + ClusterIP Service (`postgres`, port 5432)
  + `PersistentVolumeClaim` (1Gi, default `standard` local-path StorageClass)
  mounted at `/var/lib/postgresql/data`, with a `pg_isready` readiness/liveness
  probe. The PV proves DB storage survives in-cluster.
- `k8s/app.yaml` — Deployment + ClusterIP Service (`agent-relay`, port 8000),
  env `DATABASE_URL=postgresql+psycopg://relay:relay@postgres:5432/relay`
  (hostname `postgres` = the Service), HTTP `readinessProbe` on `/ready` and
  `livenessProbe` on `/health`, `imagePullPolicy: Never` (image loaded into
  kind, not pulled).

Steps:
```bash
kind load docker-image agent-relay:local --name kind
kubectl apply -f k8s/postgres.yaml -f k8s/app.yaml
kubectl rollout status deploy/agent-relay   # pods 1/1 Running
```
Dashboard via port-forward: `kubectl port-forward svc/agent-relay 8000:8000`
then open `http://127.0.0.1:8000/`. The Docker-only compose stack was removed
(`docker compose down`) so the kind deployment is what serves port 8000.

One gotcha: the pod initially crash-looped because `agent-relay:local` was the
**pre-Q4 image** (still SQLite-only `BEGIN IMMEDIATE`) — only the compose-app
image had the fix. Fixed by `docker build -t agent-relay:local .` again, reload
into kind, `kubectl rollout restart deploy/agent-relay`.

## Question 6 — CI/CD

**What should happen if a test fails in this workflow?**

> **Keep the existing version running and stop the deployment.**

A single `build-test-deploy` job runs the tests, then the docker build, then the
kind deploy as separate steps. If `pytest` fails the job aborts before any
deploy step, so the currently-running version in kind is never touched.

`.github/workflows/ci.yml` (runs on every push):

1. A `postgres` **service** container on port `5432`; the test step sets
   `RELAY_DATABASE_URL` to that Postgres and runs the full suite:
   the acceptance integration test plus the protocol tests (4 passed, and the
   SQLite-only `test_sqlite_atomic_claims_distribute_without_overlap` is skipped
   on Postgres).
2. `docker build -t agent-relay:<git-sha>` — a **unique tag per commit**.
3. `kind load docker-image` then `kubectl apply` Postgres + app, `kubectl set
   image`, and `kubectl rollout status` to **wait for the rollout**.
4. A verify step port-forwards and curls the dashboard, grepping the `<h1>`.

Run locally against your own kind cluster with act:

```bash
act --container-architecture linux/amd64 \
    -s KUBECONFIG_FILE="$(cat ~/.kube/config)" \
    -j build-test-deploy
```

act runs containers on `--network host` (its default), so inside the runner the
kind API on `127.0.0.1:<port>` is reachable; the `KUBECONFIG_FILE` secret
writes the host kubeconfig into the runner. Docker access already works because
act binds the host docker socket. I installed `kubectl` + `kind` inside the
runner workflow step (the act image doesn't ship them; kind v0.33.0 ships a
plain `kind-linux-amd64` binary, not a `.tar.gz`).

Results: image `agent-relay:478823e…` loaded into kind, rollout finished, and
the live dashboard now serves `<h1>Agent Relay v2</h1>`. The v2 heading came
from one source-code edit to `dashboard.html`; the new commit produced a new
SHA image tag automatically.
