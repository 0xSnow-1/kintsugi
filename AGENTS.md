# AGENTS.md

> Project-level instructions for AI agents working in this repository.
> Read this file before starting any task.

## Project overview

Kintsugi is a fault-tolerant stateful agent orchestrator.
It guarantees at-most-once external side-effects across hard crashes.
It routes model calls by confidence between Groq System-1 and Claude Sonnet System-2.
Read `Constitution/mission.md`, `Constitution/techstack.md`, and `Constitution/roadmap.md` before any scope or design decision.

## Repository structure

```
kintsugi/       # Application code (flat package at the project root)
services/       # Sidecar services (mock ledger, phase 3)
tests/          # Test suite
specs/          # Feature specs, one directory per feature
Constitution/   # Mission, tech stack, roadmap
.github/        # CI workflows
```

## Conventions

### Code style

`ruff` is the formatter and linter. Run `uv run ruff check . --fix` before committing.

Strong typing throughout. Public functions and models carry annotations. The package ships a `py.typed` marker.

Pydantic V2 strict mode for every schema that crosses a process or model boundary.

### Naming

Packages and modules: `snake_case`.
Compose services and containers: `snake_case`, prefixed `kintsugi-` when they belong to the app stack.
Feature spec directories: `<YY-MM-DD>-<feature-name>`, for example `26-10-13-phase-0-scaffold`.

### Testing

```bash
# Run all tests
uv run pytest

# Run a single test file
uv run pytest tests/test_smoke.py

# Chaos tests (need the compose stack up; spawn and SIGKILL a real uvicorn)
uv run pytest tests/test_chaos.py
```

Tests assume `docker compose up -d --wait` already ran. Tests fail fast with a clear message if Postgres is unreachable.

### Spec checkboxes

Plan and verification files in `specs/` track work with `- [ ]` boxes.
Tick a box (`- [x]`) when its item passes the verify step.

Tick the matching roadmap verify box in `Constitution/roadmap.md` in the same change that makes the phase's verify command pass.

### Spec kickoff

Trigger: the user says `spec the next feature`.

The trigger means this request, word for word:

> Find the next phase on `Constitution/roadmap.md` and make a branch.
> Ask me about the feature.
> Create a new directory under `specs` named `<YY-MM-DD>-<feature-name>` for this feature work.
> In there:
> - `plan.md` as numbered task groups with unchecked boxes; each group ends with one `Verify:` line.
> - `requirements.md` for context, scope in and out, decisions, deliverable files, open questions.
> - `verification.md` for definition of done, gates with commands, the run procedure, human-in-the-loop steps, merge criteria, and a failure triage table.

Follow the steps in order.
Each step names its own done-condition.

1. Read the roadmap. Done when: you can name the first phase whose verify box is unticked. That phase is the feature.
2. Create the branch from `main`. Name format: `spec/phase-<N>-<short-name>`, for example `spec/phase-1-persistence`. Done when: `git branch --show-current` prints the new name. This rule authorizes the branch creation. Commits still wait for the user's explicit ask.
3. Interview the user about the feature before writing any file. Ask about every choice that changes the spec: forms, ports, rules, where data lives. Explain an unfamiliar term in one short sentence first, then ask the question again. Point the user at the source files for concepts they should read. Done when: no open question blocks writing the three files.
4. Create `specs/<YY-MM-DD>-<feature-name>/` with today's date. Done when: the directory exists under `specs/`.
5. Write the three files in ASD-STE style, every checkbox unticked:
   - `plan.md` - numbered task groups. Each group holds `- [ ]` items and ends with one `Verify:` line.
   - `requirements.md` - context, scope in and out, decisions `D1`, `D2`, ..., deliverable files, open questions.
   - `verification.md` - definition of done, gates with commands, run procedure, human-in-the-loop section, merge criteria checkboxes, failure triage table.
   The spec checkbox rule applies.
6. Report the files and the decisions in ASD-STE style. End the report with a kickoff prompt for the implementation. Then stop. Implementation begins only when the user asks.

### Explanations to the human (ASD-STE 100 style)

You write for a human in the loop.
Use this style for every explanation you give the human.
That includes: explanations after code changes, work summaries, progress updates, and PR descriptions.

Rules:

- One idea per sentence.
- Write a maximum of 20 words per sentence.
- Use active voice and present tense.
- Use plain words.
- Define a new term in one short sentence right after the first use.
- Put each sentence on its own line.

Example:

> The scaffold is ready.
> I added the `kintsugi` package at the project root.
> Python finds it because all commands run from the project root.
> The compose stack is healthy on ports 5432 and 6333.

### Commits

Short imperative subject, sentence case. No co-author trailer. No em dash.

Examples from the constitution kickoff:

- `Initial commit: Kintsugi constitution, phase 0 spec, AGENTS.md`

## Tool permissions

Allowed:

- Read and edit files under `kintsugi/`, `services/`, `tests/`, `specs/`, `Constitution/`, `.github/`
- Run `uv sync`, `uv lock`
- Run `uv run pytest`, `uv run ruff check .`
- Run `docker compose up -d`, `docker compose down`

Restricted (ask before proceeding):

- Changing dependencies in `pyproject.toml`
- Running destructive commands (`rm -rf`, database drops, and similar)
- Pushing branches or creating releases

Not allowed:

- Modifying CI pipeline configuration without explicit instruction
- Installing new dependencies without explicit instruction
- Committing `.env` or any real secret value

## Known constraints

- Run all commands from the project root.
- Tests assume the compose stack is up. Postgres is required from phase 1 on. Qdrant is required from phase 6 on.
- Chaos tests spawn and SIGKILL a real uvicorn subprocess. Never run them against a shared or production environment.
- The runtime is one asyncio process. Do not introduce background workers without a constitution amendment.

## Verification gates

Before marking any task complete, the agent must verify:

- [ ] Tests pass (`uv run pytest`)
- [ ] Linter passes (`uv run ruff check .`)
- [ ] Import smoke passes (`uv run python -c "import kintsugi"`)
- [ ] No new warnings introduced
- [ ] No secret values in the diff
- [ ] Changed files are within the permitted scope above

## Contact / escalation

If the agent cannot proceed without a decision that falls outside its permitted scope, it should stop and describe the blocker clearly rather than making an assumption.
