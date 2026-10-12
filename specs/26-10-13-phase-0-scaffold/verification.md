# Verification: Phase 0 Scaffold

## Definition of done

A fresh clone reaches a green test run and a green lint run using only the README steps.
`docker compose up` yields a healthy Postgres and a healthy Qdrant.
CI is green on GitHub.
No secret value exists anywhere in git history.

## Gates

| Gate | Command | Passes when |
|---|---|---|
| Lint | `uv run ruff check .` | exit code 0 |
| Tests | `uv run pytest` | exit code 0, `test_smoke.py` passes |
| Import | `uv run python -c "import kintsugi"` | exit code 0 |
| Stack | `docker compose up -d --wait` | both services report healthy |
| CI | `git push` | the GitHub Actions run is green |
| Secrets | `git log --all --full-history -- .env` | empty output |

## Run procedure

1. `git clone <repo-url> && cd kintsugi`
2. `cp .env.example .env` (fill keys later; phase 0 tests need no live keys)
3. `uv sync`
4. `docker compose up -d --wait`
5. `uv run ruff check .`
6. `uv run pytest`

## Human-in-the-loop

- GitHub repository creation was a human-authorized step at project kickoff. It completed on 2026-10-13.
- The Docker daemon must be running on the builder machine before the Stack gate. The agent cannot start it.
- No CI secrets are required in this phase.
- Real `GROQ_API_KEY` and `ANTHROPIC_API_KEY` values are human-provisioned. They are first needed in phase 5, not here.
- Merging the scaffold branch to `main` is a human decision. The agent stops at a green branch and a report.

## Merge criteria

- [ ] All six gates exit 0 locally
- [ ] CI is green on the push or PR
- [ ] Roadmap phase 0 verify box is ticked in `Constitution/roadmap.md`
- [ ] Every box in this spec's `plan.md` is ticked
- [ ] The spec branch is merged to `main`

## Failure triage

| Symptom | Likely cause | Fix |
|---|---|---|
| `pytest` collection error | `uv sync` was not run, or wrong working directory | run `uv sync` from the project root |
| Compose healthcheck fails | port 5432 or 6333 already in use | stop the local service or remap the host port in compose |
| Ruff findings | style drift in new files | `uv run ruff check . --fix`, then re-run the gate |
| CI cannot install uv | workflow missing the setup-uv action | restore the workflow from plan group 5 |
| Import smoke fails | package not on `pythonpath` | confirm `pythonpath = ["."]` and the `kintsugi/` layout |
