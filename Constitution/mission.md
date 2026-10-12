# Mission

## What this project is

Kintsugi is a fault-tolerant, stateful agent orchestrator built on LangGraph and PostgreSQL.
It runs multi-step agent graphs that make external side-effects - payments, API calls, writes - at most once, even when the process dies mid-step.
The name comes from kintsugi, the Japanese art of repairing broken pottery with gold: the crash leaves a scar, but the run survives intact.
It serves two audiences.
First, recruiters and interviewers evaluating the builder's AI infrastructure engineering: every claim on the resume is backed by an executable test.
Second, GitHub reviewers who can clone the repo and run the crash-recovery proof themselves.
It exists as a portfolio artifact built to production standards.
The chaos test, not the README, is the source of truth.
It must never depend on an account or service the builder does not control, beyond the two model-provider keys and LangSmith listed in `Constitution/techstack.md`.

## The core promise

Every external side-effect executes at most once per `(thread_id, step_number, tool_args)`, even across SIGKILL and process restart.

Every completed node is checkpointed atomically in PostgreSQL.
A restarted process resumes from the last checkpoint. It never restarts the whole graph from zero.

Every user turn receives an explicit routing decision: `{complexity, confidence, escalate}`.
Confidence below `0.85` escalates from the cheap Groq System-1 tier to the Claude Sonnet System-2 tier.

The chaos test is deterministic.
It kills the server at a barrier - after the ledger insert, before the checkpoint commit - not on a timer.
After recovery it asserts exactly one ledger row and a completed thread.

Duplicate detection replays the stored result.
The caller cannot tell a replay from a first run. A log line records the replay.

Every phase of work ends with a verification command that runs locally.
A promise without a killing test is not a promise.

## Success looks like

A fresh clone reaches a green `uv run pytest` and a healthy `docker compose up` in under five minutes by following the README alone.

The chaos demo runs end to end: POST a payment task, SIGKILL the server mid-charge, restart, GET the thread, see completion and exactly one ledger row.

`uv run pytest tests/test_chaos.py` passes three consecutive runs locally and in CI.

The router report shows System-1 handling the routine query set and escalations appearing only on the labeled hard set.

The resume bar is reached when roadmap phases 0 through 5 are all ticked: scaffold, persistence, durable graph, idempotency, chaos verification, FinOps router.

A reviewer who has never seen the code can explain the crash-safety guarantee after one README read and one chaos test run.

## What this project is not

It is not a general multi-agent framework and not a LangGraph replacement.

It is not a payments product. The ledger is a local mock service we control.

It is not a SaaS. There is no first-party GUI; the surface is REST only.

It does not stream tokens. SSE was cut: it proves none of the core promises.

3-tier memory and human-in-the-loop interrupt gates are stretch roadmap phases (6 and 7), not core promises.

It is not a claim about production traffic volume. It is a claim about crash-safety engineering, proven by test.

## Operating principles

If a promise has no killing test, it is not a promise.

Wrap proven machinery. LangGraph's `AsyncPostgresSaver` is the checkpointer. The custom layer is the idempotency manager, where the differentiation lives.

Scope is ruthless. Core phases 0-5 ship before stretch phases 6-7. Stretch stays stretch until the resume bar is reached.

One process to kill. The runtime is a single asyncio server. No hidden workers muddy the crash story.

Every component earns its place or leaves the stack. Rejected choices are recorded in `Constitution/techstack.md` with reasons.

Docs are written as decisions are made: this Constitution first, then one spec directory per feature under `specs/`. Never after the fact.

Every phase ends with a local verification command. CI re-runs the same gates.

Explanations to humans use ASD-STE style: one idea per sentence, maximum 20 words per sentence, active voice, plain words, each sentence on its own line.

---
*Created: 2026-10-13*
*Last amended: 2026-10-13 - initial constitution, renamed from Janus Runtime to Kintsugi, Groq System-1 + Claude Sonnet System-2, resume bar set at phase 5*
