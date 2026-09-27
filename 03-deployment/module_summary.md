# Module 3 — Test, Containerize, and Deploy an AI-Assisted App

Takes the app from Module 2 (works on your machine) to something others can use, along a fixed path:

```
integration tests → containerization → continuous integration → deployment → continuous delivery
```

**Core philosophy:** AI tools are great at *drafting* these artifacts and bad at *owning* them. A generated `Dockerfile` that builds is not the same as one that builds the *right* thing; a green pipeline that skips the tests is worse than no pipeline. Workflow stays constant: let the agent produce v1, then **read it, run it, and break it on purpose** to check it actually catches anything.

---

## 1. Integration tests hit the real stack

Unit tests mock the world; integration tests exercise it. These run against the real database and cover the risky things unit tests can't: migrations, auth, and the frontend↔backend flow. They must be **fast enough to run on every push** (not nightly).

> *Practical example — the prompt that produced them:* "Create integration tests that run against `docker-compose.yaml`. What scenarios should we test?" — verify the frontend compiles and the backend actually talks to Postgres.

## 2. End-to-end tests drive a real browser

Playwright automates the full user journey in an actual browser against the running stack, in a dedicated `e2e/` folder, grouped behind one command (`make e2e`).

> *Practical example:* "Add an end-to-end test using Playwright to: log in as interviewer (session 1) → create an interview → share the join link → join as candidate (session 2) → move an element → verify session 1 sees the change." The test is a scripted version of a manual proof you'd otherwise do by hand.

## 3. Containerization: one image, multi-stage build

Dev runs two processes (Vite + FastAPI); **production needs one container** — the backend serves the built static frontend. A multi-stage Dockerfile splits this: stage 1 compiles the frontend in a Node image; stage 2 builds the Python backend and copies only the static output (no Node deps).

> *Practical example — run it, publishing port and mounting SQLite data:*
> ```bash
> docker build -t sdip:latest .
> docker run --rm -p 8000:8000 -v sdip-data:/data \
>   -e DATABASE_URL=sqlite:////data/sdip.db --name sdip sdip:latest
> ```
> Gotcha (in the homework): run uvicorn with `--host 0.0.0.0` inside the container, or `-p` looks broken (uvicorn defaults to `127.0.0.1`). `-p` is the flag that publishes a port; `--expose` doesn't.

## 4. Docker Compose swaps SQLite → PostgreSQL

SQLite suits local dev (single file, no server); production needs Postgres. SQLAlchemy was chosen in Module 2 precisely to make this switch trivial — just change the DB URL. Compose runs the app and Postgres together with a **DB health check** so the app waits for Postgres before connecting.

> *Practical example:*
> ```yaml
> services:
>   postgres:
>     image: postgres:16-alpine
>     environment: { POSTGRES_USER: sdip, POSTGRES_PASSWORD: sdip, POSTGRES_DB: sdip }
>     healthcheck:
>       test: ["CMD-SHELL", "pg_isready -U sdip"]
>   app:
>     build: .
>     environment: { DATABASE_URL: postgresql://sdip:sdip@postgres:5432/sdip }
>     depends_on: { postgres: { condition: service_healthy } }
> ```
> The API connects to the `postgres` **service hostname** (service name), not `localhost`. The DB URL is configurable via an env var (`DATABASE_URL`).

## 5. CI gates every pull request

GitHub Actions runs linting, unit tests, integration tests, and the container build on **every PR** — so a bad change never reaches main. Tests run in parallel (frontend + backend).

> *Practical example — the key idea:* the workflow builds the Docker Compose stack and runs the integration + E2E tests **against it**, so CI verifies the exact artifacts that will deploy.

## 6. Deploy to a public URL

One artifact, many platforms: Render, Fly.io, Railway, Cloud Run, or AWS (this article used **CloudFormation**: one EC2 instance running app + Postgres + Caddy for HTTPS/WSS). Managed DB + migrations run on deploy. For a POC a single box is fine; real apps should use a managed database like RDS.

> *Practical example:* `aws cloudformation delete-stack --stack-name sdip` tears the whole thing down cleanly.

## 7. Continuous delivery + release confidence

Merging to main is **gated on tests**, and a green merge automatically builds → migrates → redeploys. Hardening steps: staging vs. production environments, a **post-deploy smoke test** (CI checks the health endpoint after deploy), and a **documented rollback path**. If a test fails, keep the current version running and stop the deploy — never ship a broken image with a new tag.

---

**Module deliverables (repo should end with):**
```
tests/integration/   Dockerfile   docker-compose.yml
.github/workflows/ci.yml   .github/workflows/deploy.yml
docs/{testing,deployment,release-process}.md
```
The app runs at a public URL, rebuilds on merge to main, and is reproducible locally from the README. (This cohort's homework instead deploys to a **local Kubernetes cluster via `kind`** — same ideas, no cloud account needed.)
