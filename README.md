# APS — Quantitative Training Terminal

APS is a training platform for quantitative trader, researcher and developer candidates. It combines a curriculum, high-speed
drills, market simulations, cognitive training, a research lab, a coding lab, assessments and analytics. Everything works offline
on the device, and syncs to an account when you sign in.

All questions, games, lessons and data are original and synthetic. Assessment formats are unofficial practice and do not
replicate any firm's proprietary test.

## What works today

| Area | Status |
|---|---|
| **Dashboard** | Streak, training time, accuracy, response time, weekly chart, skill mastery, personal bests, week-on-week changes, recommendations, readiness (with coverage), recent sessions, baseline prompt |
| **Learning Academy** | 3 tracks from a shared curriculum contract; 26 written lessons with derivations and exactly-graded concept checks. Unwritten topics are listed as *planned* |
| **Mental Mathematics** | 15 question types plus mixed: offline, seeded, exact rationals. Untimed, sprint, fixed-count, continuous and personal-best modes. Adaptive difficulty, retry-mistakes mode, per-type statistics |
| **Sequences** | 12 rule families with ambiguity rejection, hint ladders and explanations; per-family results |
| **Fractions** | Conversion rounds plus timed recognition rounds (speed and accuracy reported separately) |
| **Probability & EV** | Python-generated problems checked against exhaustive enumeration; learning, practice and assessment modes; cached for offline use |
| **Competition problems** | Combinatorics, partitions, Diophantine, modular arithmetic, recurrences, derangements, all brute-force verified, with 3-step hints |
| **Market games** | Market-making dice/cards (benchmarked against a reference maker), ETF arbitrage (NAV, FX, costs, depth, legging, 3 AI arbitrageurs), event contracts (Bayesian), Fermi (interval scoring), betting board (EV/Kelly, virtual money) |
| **Cognitive** | Task switch, Flanker + Simon response conflict, Pincode working memory (adaptive staircase, error classification) |
| **Coding lab** | 6 problems with real tests. Python runs in WebAssembly in a Web Worker with a timeout. C++ is interface-only until the sandbox service exists. Systems performance simulator with real browser benchmarks |
| **Assessments** | Baseline staircase. Server-validated timed formats (server-side grading from seeds) and local practice runs |
| **Progress** | History charts, skills table with spaced-repetition due dates, daily plan, readiness method, game stats, server summary and cohort percentile (suppressed for small cohorts), measured performance diagnostics, sync queue |


## Architecture

See [`docs/architecture.md`](docs/architecture.md) for components, state strategy, data flow, integrity model, scoring definitions,
measured performance and the security model.

```
apps/web            Next.js 16 · React 19 · TypeScript (strict) · Tailwind v4 · KaTeX · Zustand · TanStack Query · Zod
services/api        FastAPI · Pydantic v2 · SQLAlchemy 2 (async) · Alembic · PostgreSQL 16 · Redis (optional)
packages/contracts  curriculum.json (shared by web + API seed)
docs/               architecture
```

## Running locally

### Option A — Docker Compose

```bash
cp .env.example .env
```

```bash
docker compose up --build
```

Open http://localhost:3000. API docs are at http://localhost:8000/api/docs. The API container runs migrations and seeds the curriculum on start.

### Option B — Native

Prerequisites: Node 22, Python 3.12+, PostgreSQL 16. Redis is optional.

```bash
createuser -s aps && createdb -O aps aps && createdb -O aps aps_test
```

Set the role's password to `aps` (or set `APS_DATABASE_URL`).

Backend:

```bash
cd services/api && python -m venv .venv && . .venv/bin/activate && pip install -r requirements.lock.txt
```

```bash
alembic upgrade head && python -m app.seed
```

```bash
uvicorn app.main:app --reload --port 8000
```

Frontend (in another shell). `npm install` also copies the Pyodide runtime into `public/pyodide`:

```bash
cd apps/web && npm install
```

```bash
npm run dev
```

The web app proxies `/api/*` to `APS_API_ORIGIN` (default `http://localhost:8000`). Without the backend, every local trainer,
game, drill, lesson and lab still works. Probability problem sets need the API once per level, after which they are cached.

## Testing

```bash
cd services/api && . .venv/bin/activate && python -m pytest -q
```

This runs 154 tests: generator validity and determinism, Hypothesis property tests (quotes, P&L identity, Bayes, no-arbitrage
band, interval-score propriety), and API integration tests against PostgreSQL. The integration tests cover auth, CSRF,
idempotent batches, server-graded assessments, benchmark suppression, games, export and delete.

```bash
cd apps/web && npm test
```

This runs 48 Vitest tests: rationals, generators, sequence ambiguity, engine timing and duplicate guard, the console's Enter/focus/stats behaviour, game engines (ledger exactness, replay determinism, inventory limits), cognitive designs and research numerics.

```bash
cd apps/web && npx playwright test
```

This runs 9 Playwright journeys: dashboard, flat navigation, a speed-maths session (Enter, immediate next question, stats, persistence), a market game, the betting board, task switch, progress, account sync plus a server-graded assessment, and keyboard use and mobile layout. Playwright starts the API and web servers if they are not already running.

Type checking:

```bash
cd apps/web && npm run typecheck
```

## Environment

All configuration lives in environment variables; see [`.env.example`](.env.example). For production you must set
`APS_ENV=production`, a strong `APS_JWT_SECRET` (startup refuses the default), `APS_COOKIE_SECURE=true` and the real CORS origin.

## Production deployment (outline)

1. Provision PostgreSQL 16 and, optionally, Redis. Store secrets in your platform's secret manager.
2. Build the images: `docker build -f services/api/Dockerfile .` and `docker build -f apps/web/Dockerfile --build-arg APS_API_ORIGIN=https://api.internal .` (both use the repo root as the build context).
3. Run the API container. Its entrypoint applies migrations and seeds the curriculum idempotently. Put it behind TLS.
4. Run the web container (Next.js standalone server, port 3000) behind the same TLS origin.
5. Point monitoring at `GET /api/health` and collect the JSON logs from stdout.
6. Deploy the code-execution sandbox as a separate, network-isolated service before enabling server-side judging (`APS_SANDBOX_URL`).

## Future development

Server-side game replay validation; the isolated execution service (C++/Python, secret tests); IRT/Bayesian mastery;
walk-forward and purged-CV modes in the research lab; more lessons (options, Poisson processes, Girsanov, ensembles, NUMA,
networking); multiplayer games on an authoritative server event model; an OpenTelemetry metrics pipeline.
