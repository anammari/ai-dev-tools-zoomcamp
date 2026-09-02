# Module 1 Homework — Answers

<!-- Fill in your answers below as you complete the homework. -->

## Question 1

My coding agent is **Claude Code**.

## Question 2

The spec settled on these features:

1. **Shared board** — a few users can add chores to a communal list and see what's there.
2. **Claim & complete** — users claim open chores and mark them done; the board shows status.
3. **Mixed chore types** — recurring chores (weekly trash, daily dishes) with due dates/cadence, plus one-off tasks on demand.
4. **Done tracking** — a record of who did what and when.

Full spec: [HW1_AA_spec.md](HW1_AA_spec.md)

## Question 3

`settings.py` — because the `INSTALLED_APPS` list lives there, and that's the registry Django checks to know which apps to activate.

## Question 4

**Task 1: Project scaffold** — create the Django project and `chores` app, register it in `INSTALLED_APPS`, confirm `manage.py check` passes.

Backlog: [`_docs/backlog.md`](https://github.com/anammari/ai-dev-hw1-chores/blob/main/_docs/backlog.md)

## Question 5

Start the Django dev server with:

```
uv run python manage.py runserver
```

## Question 6

Run tests with:

```
uv run python manage.py test
```

(The plain form is `python manage.py test`; `uv run` just invokes it inside the project's virtualenv.)
