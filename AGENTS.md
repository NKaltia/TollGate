# AGENTS.md: project context for AI coding agents

**Read `.cursor/rules/00-mentor.mdc` first.** It defines how to work with Nikita (mentor mode, importance levels 🔴🟠🟢, workflow). This file describes the project itself.

## What Tollgate is

Tollgate is a final-year CS project. It measures the quality cost of changes to LLM systems.

- **Evaluation harness (core).** Runs a versioned dataset against a *system under test* many times, grades outputs (deterministic graders + an LLM judge validated against human labels), compares a *baseline* and a *candidate* variant with statistics (paired comparison, confidence intervals), lists regressions, and can fail CI.
- **LLM gateway (module).** An OpenAI-compatible proxy between the system under test and model providers: timeouts, retries with backoff, fallback, circuit breaker, per-tenant caching (exact and semantic), budgets, and a per-call **trace log** (input, output, model, tokens, cost, latency).
- **Main research question (RQ4).** How safe is semantic caching for a RAG system? A threshold sweep measuring savings vs. false hits vs. answer quality.
- **System under test.** Nikita's RAG assistant "AI Second Brain" (TypeScript, PostgreSQL + pgvector, Gemini API), evaluated on public datasets (BEIR SciFact, SQuAD v2, HotpotQA, Quora Question Pairs).

Full details: `docs/01-project-definition.md`. Plan: `docs/ROADMAP.md`.

## Vocabulary (use consistently)

- **System under test (SUT)**: the AI application being measured.
- **Gateway config / policy**: cache, threshold, routing, fallback settings.
- **Variant**: a SUT version plus a gateway config.
- **Baseline / candidate**: the current variant and the variant being tested.
- **Run**: one harness execution, i.e. variant × dataset version × N trials per case.
- **Experiment**: baseline vs. candidate on the same dataset version, **changing one variable at a time**.
- **Trial**: one attempt of one test case.
- **Grader**: deterministic check, LLM judge, or human label.

## Stack

TypeScript (strict), Node.js, Express, PostgreSQL + pgvector, Zod, Vitest, GitHub Actions, Docker Compose. Later: React dashboard, optionally Redis and Python for analysis.

## Repository layout (planned; update as it evolves)

```
docs/            project documents, ROADMAP.md, adr/, learning/
packages/        (planned) harness/, gateway/, shared/
datasets/        (planned) versioned dataset files
.cursor/rules/   agent rules
```

## Current status

Planning complete. Next: **Milestone 0 (project analysis + setup)** in `docs/ROADMAP.md`.
