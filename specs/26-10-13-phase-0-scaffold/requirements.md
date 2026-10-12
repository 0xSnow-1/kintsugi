# Requirements: Phase 0 Scaffold

## Context

The Kintsugi constitution is committed: `Constitution/mission.md`, `Constitution/techstack.md`, `Constitution/roadmap.md`, plus `AGENTS.md` and the MIT license.
This phase turns the empty repository into a runnable Python project with its local service stack.
No graph, idempotency, routing, or chaos logic lands here. Those are phases 1 through 5.

## Scope in

- `uv` project with a committed lockfile
- Runtime dependencies: `langgraph`, `langchain-core`, `langchain-groq`, `langchain-anthropic`, `fastapi`, `uvicorn`, `sqlalchemy[asyncio]`, `asyncpg`, `httpx`, `pydantic>=2`
- Dev dependencies: `pytest`, `pytest-asyncio`, `ruff`, `respx`
- `ruff` and `pytest` configuration in `pyproject.toml`
- `kintsugi/` package (importable, typed) and `tests/` layout with one smoke test
- `docker-compose.yml` bringing up Postgres 16 and Qdrant with healthchecks
- `.env.example` naming every environment variable the later phases will read
- GitHub Actions CI running `ruff` and `pytest` on every push and PR
- README stub pointing readers at the Constitution

## Scope out

- No application code beyond the empty package and the smoke test
- No migrations framework (no Alembic)
- No mock ledger service (phase 3)
- No chaos harness (phase 4)
- No LangSmith tracing wiring (phase 2)
- No router, no model instantiation (phase 5)

## Decisions

- D1: Package name is `kintsugi`. GitHub slug is `kintsugi`, with `kintsugi-runtime` as the fallback if the slug is taken.
- D2: Qdrant ships in compose from day one even though it is stretch-only (phase 6). The stack shape stays stable; no compose churn mid-project.
- D3: Dependencies are locked with `uv.lock`. CI installs with `uv sync --frozen` so local and CI resolve identically.
- D4: Python floor is 3.12, matching the builder's other projects and enabling modern typing.
- D5: Tests assume the compose stack is already up. No testcontainers. Tests fail fast with a clear message if Postgres is unreachable.

## Deliverable files

- `pyproject.toml`, `uv.lock`
- `kintsugi/__init__.py`, `kintsugi/py.typed`
- `tests/test_smoke.py`
- `docker-compose.yml`
- `.env.example`, `.gitignore`
- `.github/workflows/ci.yml`
- `README.md` (stub)
- `LICENSE` (already present; this phase only verifies it)

## Open questions

- None. Providers, stack, naming, and the compose-first test policy were settled in the constitution grilling on 2026-10-13.
