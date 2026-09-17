# TableScore — Sports-League Scoreboard, Project Spec

## App name
**TableScore** — "the league at a glance."

## Problem
A small sports league (roughly 4–10 teams) needs a simple place to record match
results and see the standings, without spreadsheet gymnastics or heavy league
management software.

## Core interaction (MVP)
A **web app** usable from any device. A **single manager** records match results;
everyone else views the standings, match history, and stats. No sign-up for viewers.

## Features (settled scope)
1. **Standings table** — ranked by points, then goal difference.
   - Win = 3 pts, draw = 1, loss = 0.
2. **Record results** — the named manager adds a match with a final score; the
   standings update automatically.
3. **Manage teams** — add, edit, and remove teams as the season evolves.
4. **Match history** — a list of recent results, filterable per team.
5. **Player/team stats** — simple derived stats (e.g. goals for/against, goal
   difference) alongside the standings.

## Explicitly out of scope (v1)
- Public/shared login or approvals — editing is manager-only, viewing is open.
- Fixtures/schedule generation and advanced tie-breakers beyond goal difference.
- Player-level records (lineups, individual scorers) — stays team-focused.
- Notifications, multiple seasons, or division structure.

## Design decisions
- **Audience:** a small league; one person is trusted to record results.
- **Scoring:** classic 3/1/0 with goal difference as the tie-breaker.
- **Access:** open read, manager-write (simple auth for the manager only).

## Open questions (for later)
- Exact auth mechanism for the manager (shared secret vs. real login).
- Whether stats should extend to top scorers / clean sheets (needs player data).
- Hosting and whether a future public share link needs a slug.
