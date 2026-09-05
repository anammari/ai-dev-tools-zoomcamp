# Module 1 — AI-Native Developer Workflow

**Core idea:** Coding agents write code faster than we can read it, so our job shifts from *typing* to *saying precisely what we want* and *checking what came back*. A vague request makes the agent fill the gaps with its own assumptions — a strong agent that misunderstands you will happily build 8 files, wire them together, and add passing tests for the *wrong* thing.

The module walks a deliberately vague idea — *"a tool for weekly feedback for projects"* — through specification, context, a backlog, and a three-role agent team (PM, engineer, QA). The cautionary example is `weekly-feedback` (a CLI built from one vague sentence); the correct result is `retroloop` (a Django retrospective app built spec-first).

---

## 1. Spec-driven development — write the spec before any code

Think the product through in detail and give explicit instructions *before* the agent writes a line of code. If the spec is vague, the agent guesses.

**Example:** Given only *"a tool for weekly feedback for projects"*, Claude Code produced `weekly-feedback` — a CLI tracking wins/issues/blockers with 62 passing tests. It worked perfectly but was the wrong product. The real need was a **web** team-retrospective tool using **Start/Stop/Continue** cards.

## 2. Start in a chat assistant — brainstorm the scope

Don't feed the vague idea straight to a coding agent. Talk it through in a chat app (e.g. ChatGPT dictation) to pin down the requirements.

**Example prompt:**
```
I want to build a tool for weekly feedback for projects.
Help me set the scope for this project precisely. I want to brainstorm with you
and understand how the tool should work. Give me options.
Ask me one question at a time and keep your output short.
```
This surfaced the real requirements: all team members contribute, Start/Stop/Continue format, optional anonymity, facilitator reveals all cards at once, then clustering + 3 votes per person, decisions/action items recorded. Finish with *"Save everything to a markdown file that I can download"* → `plan.md`.

## 3. Bootstrap the project & choose the stack

Create the repo, commit the plan early and often (so you can review agent changes and roll back to a good state), then let the agent propose tech-stack options.

**Example:**
```bash
mkdir project-name && cd project-name && git init
mkdir -p _docs && mv ~/Downloads/plan.md _docs/plan.md
git add _docs/plan.md && git commit -m "Add project plan"
```
Then: *"Read _docs/plan.md. Propose multiple options for the tech stack and explain each option. Don't write code yet."* → pick Django (or whatever you can review).

## 4. Turn decisions into a backlog

Ask the agent to decompose the spec into small, independent tasks, then move them to a tracker (GitHub issues). Each task must fit in one session and be handoff-able to someone who hasn't read the others.

**Example task template:**
```
## <number>. <title>
Goal: <one line>
Description: <two or three sentences>
```
First task = *"setting up an empty project with a passing test."* Review, merge/split tasks, trim to an MVP, then: *"Create a public GitHub repo… Move each task from _docs/tasks.md into a GitHub issue."* From here, GitHub issues are the only active backlog.

## 5. Context engineering — control what the agent knows each session

Prompt engineering controls one message; **context engineering** controls what the agent knows when it starts and can find while working. Put durable facts and working rules in `AGENTS.md` (read by Codex/OpenCode; Claude Code reads `CLAUDE.md`, so add a one-line `@AGENTS.md` to it).

**Example `AGENTS.md`:**
```
Commands
- `uv sync` - install dependencies
- `uv run pytest` - the whole suite
Rules
- Dependencies are added in `pyproject.toml`. Do not add one without asking
- Commit regularly
```
Keep `AGENTS.md` short by linking other living docs in `_docs/` (e.g. `process.md`, `testing-guidelines.md`, `design-system.md`, `api.md`) and loading each only when relevant. When you correct an agent, ask it to update these docs so you don't correct it twice.

## 6. Grooming — the Product Manager agent

Tasks aren't precise enough to implement directly. **Grooming** makes a task specific *before* code is written, so an engineer can implement it without asking questions. Catching a misunderstanding here costs one sentence; after implementation it costs a rewrite.

**PM role (`_docs/team/pm.md`):** rewrite the issue using a template, make acceptance criteria *checkable* (point at the screen → yes/no), think of edge cases, file follow-up issues for out-of-scope items, never write code.

**Groomed task template (`_docs/task-template.md`):** `Goal` · `Acceptance criteria` (checkable) · `Out of scope` (linked to follow-up issues) · `Constraints` (files, libraries, prior decisions).

## 7. Loop engineering — let the harness drive repetition

Instead of prompting the agent repeatedly, give it a **goal** with a *checkable stop condition*; the harness resumes the agent until the condition holds.

**Example:** `/goal groom all issues` — stop condition: *"all issues are groomed."* Good conditions are checkable ("all tests pass", "no file is over 200 lines"); *"make the code better"* is not, so the agent stops too early or runs forever.

> *"Stop prompting your agents and start designing the loops that prompt them."* — Peter Steinberger

## 8. Implementation & testing — the Engineer and QA agents

**Software Engineer (`_docs/team/software-engineer.md`):** implements one groomed task, works against (never changes) the acceptance criteria, stays inside the named files/constraints, writes tests, commits, does **not** close the issue.

**QA Engineer (`_docs/team/qa-engineer.md`):** checks finished work against the issue's acceptance criteria, runs the tests, looks for cases the criteria describe but tests miss, and **does not fix anything** — it posts a `PASS`/`FAIL` verdict comment. A single failing criterion ⇒ `FAIL`.

**Example QA comment:**
```
## QA: FAIL
- [x] A visitor can create an account with a username and password - PASS
- [ ] A duplicate username shows a visible error - FAIL
      Submitted an existing username and received an unhandled error
Tests: `uv run pytest`, 18 passed, 0 failed
```
On `FAIL`, start a new engineer session with the QA comment as input and iterate until `PASS`. (The same agent writing and judging its own code is grading its own homework.)

## 9. Graph engineering — orchestrate multiple agents

With three roles, the workflow is a **graph**: PM grooms → Engineer implements → QA tests → on `FAIL` back to Engineer, on `PASS` close the issue. An **orchestrator** (the main session) launches these as subagents and drives the loop.

**Orchestrator lifecycle (`process.md`):** pick next open issue → PM grooms → Engineer implements → QA verifies → on FAIL return to Engineer with the QA comment → on PASS close the issue → repeat until the backlog is empty. Rules: don't skip grooming, engineer never closes the issue, QA only outputs PASS/FAIL, orchestrator closes only after PASS.

**Launch:** `/goal work through the backlog` — the agent reads `AGENTS.md`, finds `process.md`, and dispatches the role agents until the backlog is empty.

> **Caveat:** this full team approach costs significantly more time and tokens than a simple engineer loop. Often a plain prompt or loop is enough — add the graph only when the complexity pays off.

---

**Key takeaway:** The four "engineering" levels build on each other — **prompt** (what you say) → **context** (what the agent knows) → **loop** (when it stops) → **graph** (who does what with multiple agents). Even with graphs, you still need a loop to drive the work.

*Source: [AI-Native Development: Specifications, Loop and Graph Engineering](https://aishippingblog.com/p/ai-native-development-specifications) · [retroloop repo](https://github.com/alexeygrigorev/retroloop)*
