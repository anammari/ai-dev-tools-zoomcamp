# Homework 4 — Answers

Starter repo (my fork): https://github.com/anammari/ai-dev-hw4-order-tracker

## Question 1 — Run the app

**What does the health check return?**

- [x] `{"status":"ok"}`
- [ ] `{"status":"error"}`
- [ ] `{"orders":3}`
- [ ] `pong`

`/healthz` runs a DB check and returns 200.

## Question 2 — Instrument one endpoint

**Which HTTP status code does the metric record for this lookup?**

- [x] 200
- [ ] 301
- [ ] 404
- [ ] 500

`standard-1001` is a seeded order, so the lookup succeeds with 200.

## Question 3 — Build the telemetry pipeline

**Which HTTP status code does the metric show?**

- [x] 404
- [ ] 200
- [ ] 301
- [ ] 500

`standard-1002` is not a seeded order, so the lookup returns 404. Its log and trace agree:
`trace_id=4444cd87d4f8a98ddf7e4ee721579896`.

## Question 4 — Configure the alert

**What state does Grafana show?**

- [x] Normal
- [ ] Firing
- [ ] Pending
- [ ] No data

The alert watches **5xx** only; the lookup returned **404** (a 4xx), so no 5xx exists and
`noDataState: OK` resolves the rule to Normal instead of "No data".

## Question 5 — Build the automatic responder

**What did the agent respond? Include the last line from its answer.**

> VERDICT: This is an explicit test notification (ResponderTest) — no incident, no code change needed.

It checked logs, traces and metrics (all empty), confirmed the app was healthy, ran the tests
(3 passed), and changed nothing.

## Question 6 — Watch the agent fix the incident

**What was the problem?**

- [x] The express delivery date calculation tried to use a day that does not exist in that month.
- [ ] The order timestamp could not be parsed because it had no time zone.
- [ ] The app rejected the order's `preparing` status.
- [ ] The lookup searched the wrong database column for express orders.

`express-1002` was seeded with `created_at` = Sep 30, and `order_detail` asked for
`replace(day=30 + 2)`: day 32 of September, which does not exist. Fixed with `timedelta(days=2)`.

## Submission

- [ ] Pushed telemetry/alert config, responder, incident evidence, agent's fix
- [ ] Submitted the repo link on the [course platform](https://courses.datatalks.club/ai-dev-tools-2026/homework/hw4)
