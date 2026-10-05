# Module 4 — DevOps and Observability for AI-Built Apps

Module 3 ends with an app that deploys automatically when tests pass. That gets it online — it does **not** tell you whether it still works. If an endpoint starts returning `500`, you must know users are affected, find the failed requests, tie them to the deployment that caused them, and decide what to do. **CI/CD ships a bad release as efficiently as a good one.**

This module builds the loop that closes that gap:

```text
change → observe user impact → alert with context → investigate from evidence
→ authorize a bounded response or escalate → verify recovery → audit code and response
```

**Core philosophy:** the model may **reason**; the system must **observe, authorize, verify, and remember**. A model supplies confidence; code outside the model enforces permission.

---

## 1. Separate dev and prod environments

Auto-deploying every push is fine before you have users. With real users you want two copies of the same infrastructure: **dev** takes every tested push automatically; **prod** is what users use and receives only deliberately promoted releases. Infrastructure-as-code means the second copy is mostly a reuse — usually just a machine size change.

> *Practical example — the promoting prompts:* "Create a second, independent copy of our infrastructure. We will use the copy as production, and the existing infrastructure as a dev environment." → then "Create a manual GitHub Actions workflow that promotes the dev version to production." (a workflow with a confirmation checkbox and optional release tag).

## 2. Build once, promote the same image

Building the Docker image *inside* the deploy stage is an anti-pattern: (1) build and deploy are two responsibilities — a failed build must not deploy; (2) promoting dev to prod would **rebuild**, and the result may differ from what you tested. Split CI into **Build** (push a tagged image to a registry, e.g. Amazon ECR) and **Deploy** (pull that exact image). Promotion then pulls the *same* image to prod.

> *Practical example:* "Split it into two stages: Build — build the image and push it to a container registry (ECR); Deploy — pull the image from the registry and serve it. The manual prod promotion workflow pulls the currently deployed dev image to prod. Tag each image using `YYYYMMDD-HHMMSS-shortsha` (e.g. `20260818-163457-83242da`)."

## 3. Observability = metrics + logs + traces

"Observability" means collecting information about the app so you can understand its behaviour and find problems fast. Minimum: CPU/memory utilization and requests per second — growing CPU with dropping RPS signals trouble, but says nothing about *what happened inside a request*. For that you need all three signals, unified by **OpenTelemetry (OTel)**, the industry standard, sent over the **OTLP** protocol:

- **Metrics** — numerical measurements over time (RPS, latency, error counts)
- **Logs** — timestamped records of individual events (an error, a failed query)
- **Traces** — every step ("span") one request makes through the system

> *Practical example — instrumenting is a few lines:* "Instrument the backend with OpenTelemetry. Include in the telemetry: service name, environment, deployed version." Auto-instrumentation covers libraries like FastAPI without code changes; stamps on every signal are what let you filter by environment and version later.

## 4. Wire the telemetry pipeline

The app exports over OTLP but nothing stores it. Put an **OpenTelemetry Collector** between the app and storage, and pick backends with your assistant.

> *Practical example:* "Add an OpenTelemetry Collector. Create an `observability/` directory with Docker Compose for: OpenTelemetry Collector, Prometheus, Loki, Tempo, Grafana. Keep this as a separate Compose project from the application stack." (Prometheus = metrics, Loki = logs, Tempo = traces, Grafana = dashboards.) If you prefer a managed service — CloudWatch, Grafana Cloud, Datadog, Sentry — you can skip hosting this yourself.

## 5. Define app metrics and dashboard them

Infrastructure metrics aren't product metrics. Instrument the things that describe *your* app's health, then display them.

> *Practical example:* "Track these application metrics: interview rooms created, active interview participants, canvas elements created, failures in component creation. Include the environment and deployed version." → "Add a Grafana panel with these metrics. Make it possible to filter by environment and deployed version." Test it by creating a room and a canvas element and confirming it shows up in Grafana. Then: "Deploy the observability stack. It should be separate from the application stack. Connect both development and production to it."

> *Gotcha:* deploying Grafana openly on the Internet is a POC shortcut. In practice observability services and databases belong on **private networks**, with network access restricted and authentication required.

## 6. Alert on real user impact, not on graphs

Nobody watches dashboards all day. An alert must fire when impact is real, and must carry enough context to act on without a scavenger hunt.

> *Practical example:* "Add an actionable alert for repeated canvas component-creation failures. Use a threshold and duration that represent real user impact. Include the service, environment, deployed version, owner, and dashboard URL in the alert."

## 7. The AI on-call responder

Normally an on-call engineer receives the alert, diagnoses it, and stops the bleeding. An AI agent can be that first responder — but the design that matters is what it is *given*: collect the same evidence you would collect (**read-only, allowlisted queries**), compare it with recent changes, and propose an action. It does **not** get general production credentials.

> *Practical example — the responder prompt:* "You are the on-call engineer for this repository. An alert just fired. Investigate the root cause. Read the code and reproduce the failure. If you find a real bug, make the smallest correction, run the backend tests, and commit the fix with a clear message. If the alert is a false positive, explain why and do not change the code." A poller script passes alert details to a headless coding agent (Codex or Claude Code); realistic wiring is alert → SNS → Lambda → isolated container job → session log saved, compute terminated.

## 8. Gate every automated action

The article's responder is deliberately a **proof of concept**: it lacks the allowlists, escalation path, and recovery verification a production setup needs — and the module's own lessons add those guardrails. Fixes go to an **isolated branch** so normal review and CI/CD still apply; a real deployment also needs **autonomy levels** (read-only → propose → execute allowlisted actions → escalate), a human escalation decision for serious incidents, and a **verified recovery** rather than a claimed one.

> *Practical example — the boundary rule:* an alert fires; the agent may read code, logs, and metrics and open a branch. It may **not** run destructive commands or touch prod unless the action is on the allowlist. When a model proposes a rollback, the *system* executes it and re-checks the metric.

## 9. Treat the responder as attack surface

An agent with repo access and incident context is a target. Audit it explicitly: inventory its **capabilities, credentials, and provenance**, and run **recurring security audits** that combine a deterministic scanner (**Semgrep**) with model review and human validation — run them repeatedly, not once.

> *Practical example:* the `security-audit/` deliverable keeps `audit-brief.md`, `findings.schema.json`, `capability-table.md`, and `runs/`, so every agent capability is written down and dispositioned by a human.

## 10. Prove the loop works — break it on purpose

You have no evidence the responder wakes up until an alert actually fires. So introduce a realistic bug and watch the whole chain react.

> *Practical example:* "Introduce a realistic bug in canvas component creation. For some requests, creating a component should fail even though the existing tests pass. Keep the failure reproducible so we can test that the bug causes an alert and the on-call response." Iterate until the on-call agent wakes up and fixes it.

---

**Module deliverable (operations and security report):**
```text
observability/     collector.yaml  compose.yaml  dashboard.json  alerts.yaml
incident-response/ collect-evidence.sh  responder-task.md  response.schema.json
                   autonomy-policy.yaml  runbooks/{rollback,verify-recovery}.sh  incidents/
security-audit/    audit-brief.md  findings.schema.json  capability-table.md  runs/
docs/operations-and-security-report.md
```
Given one incident ID, the report must reconstruct: the deployed version and user impact · the alert and evidence inspected · the model and configuration used, and the action proposed · the policy decision and the command actually executed · recovery verification or the escalation packet · any related security finding and its human disposition.

**Clean up when done:** "Delete the CloudFormation stacks." — and verify yourself in the console, or the learning environment becomes a monthly bill.

**Deliberate non-goals** (obscure the main loop, not unimportant): SLOs/error budgets, synthetic & real-user monitoring, sampling and retention cost control, canary/progressive delivery, on-call ownership and postmortems, SBOMs/signing/pen testing. Add an SLI and SLO for your most important user journey first. Also: deploying to EC2 via a shell script is fine for a POC but use a container management system (ECS) in production, adopt a managed database with off-stack backups you have actually restored, and put internal services on private subnets.

*Source: [DevOps and Observability for an AI-Built App](https://aishippingblog.com/p/devops-and-observability-for-an-ai) · [Interview Canvas repo](https://github.com/alexeygrigorev/interview-canvas-share)*
