# Shared Household Chores — Project Spec

## Problem
Households with a few users (mainly a solo planner but with light participation
from roommates/family) need a simple way to share, claim, and track chores
without scheduling overhead.

## Core interaction (MVP)
A **web app** with a **shared claim board**: anyone in the household can add
chores and claim them. Completion is tracked per chore.

## Features (settled scope)
1. **Shared board** — a few users can add chores to a communal list and see what's there.
2. **Claim & complete** — users claim open chores and mark them done; the board shows status.
3. **Mixed chore types** — recurring chores (e.g. weekly trash, daily dishes) with due
   dates/cadence, plus one-off tasks added on demand.
4. **Done tracking** — a record of who did what and when.

## Explicitly out of scope (v1)
- Gamification / points / streaks
- Rotation or scheduling rules/assignments
- Per-person roles or approval flows (e.g. parent approval)
- Heavy automation — recurring chores simply roll back around with a due date

## Design decisions
- **Audience:** a household of a few users; one person drives planning.
- **Chores:** a mix of recurring and one-off, claimed on demand rather than assigned.
- **Simplicity first:** no auth complexity beyond a few users; no scheduler engine.

## Open questions (for later)
- Account model: per-user accounts vs. a shared household with named members.
- Reminders/notifications (email, push) for due recurring chores.
- Mobile responsiveness vs. dedicated mobile app.
