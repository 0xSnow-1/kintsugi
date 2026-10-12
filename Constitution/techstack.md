# Tech Stack

Decisions live here.
The mission (`Constitution/mission.md`) states the promises. This file records what we build with and why.

## Runtime and tooling

Python 3.12 with `uv` for environment and dependency management.

`ruff` for lint and format.

`pytest` + `pytest-asyncio` for tests, with `respx` for mocking HTTP in tool contract tests.

## Core framework choices

| Layer | Choice | Why |
|---|---|---|
| Orchestration | LangGraph | Checkpointed durable graphs are LangGraph's core strength; `AsyncPostgresSaver` is production-proven; the skill is job-market relevant |
| API | FastAPI + uvicorn | The chaos test needs one killable server process; three JSON endpoints, no SSE |
| Persistence | PostgreSQL 16 via async SQLAlchemy + asyncpg | Checkpoints and the idempotency table live in one atomic store the API and the test both reach after restart |
| Checkpointer | LangGraph `AsyncPostgresSaver`, wrapped | Writing a custom checkpointer was rejected; our custom layer is the idempotency manager, not the checkpoint protocol |
| System-1 LLM | Groq `llama-3.3-70b-versatile` | Fast and cheap; carries the routine traffic; key already in the builder's environment |
| System-2 LLM | Claude Sonnet (Anthropic API) | Escalation tier below the `0.85` confidence threshold; Sonnet is the floor - Haiku is too weak a story, Opus rejected on cost |
| Router | Pydantic V2 strict, structured self-classification | One LLM call returns `{complexity, confidence, escalate}`; no separate judge round-trip |
| Side-effect demo | Mock ledger: separate FastAPI compose service with its own database | A real process boundary; the chaos test counts rows to prove no duplicates |
| Vector store (stretch) | Qdrant in compose | Tier-3 semantic memory; local, no cloud account; service present from phase 0 so the stack shape never churns |
| Tracing | LangSmith | LangGraph traces graph, LLM, and tool spans natively; free-tier keys already provisioned |
| Infra | Docker Compose: Postgres 16 + Qdrant (+ mock ledger from phase 3) | One command brings up the stack; tests assume compose is already up and fail fast with a clear message if Postgres is unreachable |
| CI | GitHub Actions | `ruff` + `pytest` on every push and PR; chaos test runs against a Postgres service container |
| License | MIT | Portfolio repos should be readable and reusable |

## Provider wiring (Groq + Anthropic)

Environment names go in `.env.example`. `.env` is never committed.

- `GROQ_API_KEY` - instantiates `langchain-groq.ChatGroq(model="llama-3.3-70b-versatile")` for System-1
- `ANTHROPIC_API_KEY` - instantiates `langchain-anthropic.ChatAnthropic(model=<current Sonnet>)` for System-2
- `DATABASE_URL` - async SQLAlchemy Postgres URL, e.g. `postgresql+asyncpg://kintsugi:kintsugi@localhost:5432/kintsugi`
- `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` - optional locally, set as CI secrets
- `MOCK_LEDGER_URL` - base URL of the mock ledger service, e.g. `http://localhost:9010`

The router owns model selection. Nodes never instantiate a model directly; they receive the routed client.

## Self-hosted services

PostgreSQL 16 runs as a pinned compose service with a healthcheck and a named volume.
It stores graph checkpoints, the `idempotency_keys` table, and (stretch) episodic memory tables.

Qdrant runs as a pinned compose service with a healthcheck.
It is unused until stretch phase 6. It ships in compose from phase 0 on purpose.

The mock ledger is added to compose in phase 3.
It is a tiny FastAPI app living in `services/mock_ledger` in this repo.
`POST /charges` inserts exactly one row per request and returns the charge record.
`GET /charges` lists all rows. The chaos test counts rows through this endpoint or its database.
It owns its own database. It is the only external system the agent's charge tool ever talks to.

## Rejected choices (with reasons)

Fully custom checkpointer: LangGraph already ships `AsyncPostgresSaver`. Reimplementing the checkpoint protocol steals time from the idempotency manager, which is the actual differentiator.

Celery, RQ, Kafka: the runtime is one asyncio process. A single killable process keeps the chaos story legible.

SSE streaming: proves none of the core promises. Cut, the same call the builder made on Shippy.

SQLite: the checkpointer and the post-restart recovery need a real server that both the API process and the test reach independently.

testcontainers: tests assume `docker compose up` already ran. Faster, fewer moving parts. The README documents the prerequisite; tests fail fast if Postgres is unreachable.

Timed SIGKILL: a sleep-then-kill test is flaky and proves little. The barrier design kills at an exact, deterministic state.

Replay-with-flag or conflict-error on duplicate detection: silent replay of the stored `result_payload` is the true idempotent contract. Observability is a log line, not API surface.

Haiku as System-2: too weak a story for deep-reasoning escalation. Sonnet is the floor.

OpenAI or Bedrock as providers: no keys in hand. A provider interface is deferred - YAGNI until a third backend is real.

Redis: no cache layer is needed at this scale. Postgres is fast enough.

Alembic migrations: two app-owned tables at demo scale. `create_all` plus schema-as-code is enough. Revisit if the schema grows past the checkpointer's own tables.

A first-party GUI: the surface is REST. A GUI is a third thing to maintain and proves none of the promises.

## External dependencies

Groq and Anthropic keys, provisioned and paid by the builder.
LangSmith free tier, already provisioned by the builder.
Zero other paid accounts.
Every other service runs locally under Docker.

---
*Created: 2026-10-13*
*Last amended: 2026-10-13 - initial stack: LangGraph + AsyncPostgresSaver wrap, Groq System-1, Claude Sonnet System-2, mock ledger compose service, Qdrant from phase 0, MIT*
