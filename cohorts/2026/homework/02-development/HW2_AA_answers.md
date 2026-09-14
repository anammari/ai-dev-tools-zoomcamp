# Module 2 Homework — Answers

<!-- Fill in your answers below as you complete the homework. -->

## Question 1

I chose **Sports-league scoreboard** for this homework.

## Question 2

Spec: [`HW2_AA_spec.md`](HW2_AA_spec.md)

App name I chose: **TableScore**

## Question 3

Repo: [anammari/ai-dev-hw2-table-score](https://github.com/anammari/ai-dev-hw2-table-score)

Commit sha1: `b087763b68681eff466853e81a9b04cf0ba78bba`

## Question 4

Start the frontend with:

```
npm run dev
```

(Run from the `frontend/` directory; serves the app at http://localhost:5173.)

## Question 5

Start the backend with:

```
uv run uvicorn app.main:app --reload --port 8000
```

(Run from the `backend/` directory; serves the API at http://localhost:8000, docs at /docs)

## Question 6

The frontend talks to the backend at:

```
http://localhost:8000
```

This is the `BASE_URL` in `frontend/src/api.js` — the single place all backend calls are centered.

## Question 7

Run tests with:

```
uv run pytest
```

(The plain form is `pytest`; `uv run` invokes it inside the backend's virtualenv.)
