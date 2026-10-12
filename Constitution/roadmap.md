# Roadmap

High-level implementation order in small phases for Kintsugi.

Each phase is sized to 1-3 days and ends with a verification command.

Each phase is a task list.
The verify box is the last item.
Tick the verify box only when the verify command passes.

Phases are sequential unless marked parallel-safe.

The resume bar is the end of phase 5.
Phases 6 and 7 are stretch.
Phase 8 is polish.

## Phase 0: Scaffold

- [ ] `uv` environment with locked dependencies (`pyproject.toml`, `uv.lock`)
- [ ] `ruff` and `pytest` (+ `pytest-asyncio`) wired
- [ ] `kintsugi` package layout with `py.typed`
- [ ] `docker-compose.yml`: Postgres 16 + Qdrant, healthchecks, named volumes
- [ ] `.env.example` (Groq, Anthropic, `DATABASE_URL`, LangSmith, `MOCK_LEDGER_URL` placeholder)
- [ ] `.gitignore` (never commit `.env`)
- [ ] MIT `LICENSE`
- [ ] GitHub Actions CI: `ruff` + `pytest` on push and PR
- [ ] README stub pointing at the Constitution
- [ ] Verify: `uv sync && uv run ruff check . && uv run pytest` all pass; `docker compose up -d --wait` reaches healthy.

## Phase 1: Persistence layer

- [ ] `database.py`: async SQLAlchemy engine and session factory for Postgres
- [ ] `idempotency_keys` table: `key_hash`, `thread_id`, `status`, `result_payload`, `created_at`
- [ ] Wire LangGraph `AsyncPostgresSaver` as the graph checkpointer (wrap, do not rewrite)
- [ ] Checkpoint write/read round-trip against a live thread
- [ ] Idempotency row insert + select round-trip
- [ ] Verify: `uv run pytest tests/test_persistence.py` passes against a running compose Postgres.

## Phase 2: Durable graph and API

- [ ] `graph.py`: typed `StateGraph` with a multi-step flow (classify, plan, respond)
- [ ] Compile the graph with the `AsyncPostgresSaver` from phase 1
- [ ] FastAPI app with three JSON endpoints: `POST /threads`, `POST /threads/{id}/messages`, `GET /threads/{id}`
- [ ] uvicorn entrypoint, single process
- [ ] Restart durability: stop the server, restart it, thread state is intact and the run can continue
- [ ] LangSmith tracing on graph and model spans
- [ ] Verify: `docker compose up`, POST a message, GET returns completed state with checkpoints; restart the server, GET still returns the same state; `uv run pytest tests/test_graph.py` passes.

## Phase 3: Idempotency and mock ledger

- [ ] `services/mock_ledger`: tiny FastAPI app, `POST /charges` inserts one row, `GET /charges` lists, own database, compose service
- [ ] Charge tool in `kintsugi`: HTTP `POST` to the mock ledger via `httpx`
- [ ] `idempotency.py`: deterministic key `hash(thread_id, step_number, tool_args)`; statuses `pending` / `complete`
- [ ] Guard: check the table before the side-effect; store `result_payload` on completion; replay the stored result on duplicate
- [ ] Wire the charge tool through the guard as a graph node
- [ ] Same-key double invocation yields one ledger row and identical results both times
- [ ] Verify: `uv run pytest tests/test_idempotency.py` passes; a manual double POST leaves exactly one ledger row.

## Phase 4: Chaos verification

- [ ] `tests/test_chaos.py`: pytest-asyncio harness
- [ ] Deterministic barrier: the charge tool completes the ledger insert, then blocks on a test-controlled gate before the node checkpoint
- [ ] Harness spawns uvicorn as a subprocess, POSTs a payment task, waits for the barrier marker
- [ ] SIGKILL the subprocess at the barrier
- [ ] Restart uvicorn, GET the thread, poll until completed
- [ ] Assert exactly one ledger row, completed thread, replayed `result_payload`
- [ ] Repeat three times in one session to prove non-flakiness
- [ ] Verify: `uv run pytest tests/test_chaos.py` passes three consecutive runs against a running compose stack.

## Phase 5: FinOps router

- [ ] `router.py`: Pydantic V2 strict decision model `{complexity, confidence, escalate, rationale}`
- [ ] Structured self-classification: one LLM call returns the decision as strict JSON
- [ ] System-1: Groq `llama-3.3-70b-versatile`
- [ ] System-2: Claude Sonnet
- [ ] Escalate when confidence is below `0.85`
- [ ] Wire the router into the graph entry node; log every route decision to LangSmith and structured logs
- [ ] Router report: same query set routed through both tiers, print the System-1 / System-2 split
- [ ] Verify: `uv run pytest tests/test_router.py` passes; a 20-query sample run shows escalations only on the labeled hard set.

## Phase 6 (stretch): 3-tier memory

- [ ] Tier 1: working memory in LangGraph thread state, flushed on node completion, token-bounded
- [ ] Tier 2: Postgres episodic tables; background summarizer triggers at more than 20 turns into structured JSON facts
- [ ] Tier 3: Qdrant semantic store; LRU decay combined with cosine-distance thresholding prunes stale vectors
- [ ] Verify: `uv run pytest tests/test_memory_tier.py` passes; turn 21 produces a summary; stale vectors are pruned.

## Phase 7 (stretch): HITL interrupt gates

- [ ] `interrupt()` at high-risk graph nodes; thread pauses in durable state
- [ ] `POST /threads/{id}/resume` accepts an authorized approval payload and resumes execution
- [ ] Pause across a server restart (days-long pause simulated by a restart)
- [ ] Reject unauthorized or malformed resume payloads
- [ ] Verify: `uv run pytest tests/test_hitl.py` passes; interrupt, restart server, resume, run completes; unauthorized resume returns 4xx.

## Phase 8: Polish

- [ ] README: cold start under five minutes, chaos demo walkthrough, architecture diagram
- [ ] Demo recording (GIF or short video) of the SIGKILL recovery
- [ ] Resume bullets drafted from the phase 0-5 outcomes
- [ ] Defense drill: a two-sentence answer per kept component; a failure simplifies or cuts that component
- [ ] Verify: follow the README on a clean clone without skipping steps; the builder passes the defense drill.

## Dependency notes

Phases 0 and 1 block everything downstream.

Phase 2 blocks phases 3, 4, and 7.

Phase 3 blocks phase 4.

Phase 5 is parallel-safe with phases 3 and 4 once phase 2 has landed.

Phase 6 needs only the phase 1 persistence layer and can run parallel to 3-5.

Phase 7 needs the phase 2 graph and API.

The mock ledger service must exist before the phase 4 chaos harness can assert on ledger rows; it is built in phase 3.

Human-only actions: GitHub repo creation (completed at kickoff, 2026-10-13), Docker daemon running before any verify gate, real `GROQ_API_KEY` and `ANTHROPIC_API_KEY` in `.env` before phase 5 live model calls.

---
*Created: 2026-10-13*
*Last amended: 2026-10-13 - initial nine-phase roadmap, resume bar at phase 5, stretch marked on phases 6-7*
