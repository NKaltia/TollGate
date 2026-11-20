# Tollgate

**Measuring the quality cost of optimizing LLM systems.**

*Tollgate is a working name.*

Final-year B.Sc. Computer Science project, Afeka College of Engineering.
Author: Nikita Kaltia · Supervisor: Yair

---

## In one paragraph

Teams that put LLMs into production face three linked problems:
- **Quality.** Model outputs are non-deterministic, so a change to a prompt or model can silently break behaviour that used to work.
- **Reliability and cost.** Calls to model providers fail, slow down and cost money.
- **Visibility.** It is hard to see what each AI feature costs and how long it takes.

Tollgate addresses all three with one system built from two parts:
- **An evaluation harness (the core).** It runs a versioned test set against an LLM system many times, grades the results with validated graders, and reports with statistical confidence whether a change made things better or worse. It can block a pull request in CI when quality drops.
- **An LLM gateway.** It sits between the application and model providers and adds retries, fallback models, per-tenant caching, budgets and a per-call trace of cost and latency.

The two parts are connected on purpose. Every cost optimization the gateway makes (a cache, a cheaper model, a fallback) is measured by the harness for its effect on quality. Engineering decisions then rest on numbers, not intuition.

## Documents

| # | Document | Status |
|---|----------|--------|
| 1 | [Project definition](docs/01-project-definition.md) | Draft v1 |
| 2 | [Literature review](docs/02-literature-review.md) | Draft v1 |
| 3 | [Competitors review](docs/03-competitors-review.md) | Draft v1 |
| 4 | [Requirements: testable MVP scope](docs/04-requirements.md) | Draft v1 |
| 5 | Detailed design | Planned (M0) |
| — | [Roadmap and milestones](docs/ROADMAP.md) | v1 |
| — | [Course deliverables tracker](docs/COURSE_DELIVERABLES.md) | Ongoing |
| — | [Architecture Decision Records](docs/adr/README.md) | Ongoing |
| — | [Learning log](docs/learning/LEARNING_LOG.md) | Ongoing |

## Status

Planning phase. No implementation code yet. The first system under test will be the author's existing RAG assistant ([AI Second Brain](https://github.com/NKaltia/AI_Second_Brain)), evaluated on public question-answering and retrieval datasets.

## Main research focus

**How safe is semantic caching for a RAG system?** We measure savings, false hits and quality loss across similarity thresholds, and derive a safe caching policy.

## Planned stack

TypeScript, Node.js / Express, **PostgreSQL + pgvector**, GitHub Actions, Docker, LLM provider APIs, with React for a later dashboard. The database choice (PostgreSQL over MongoDB) will be documented in ADR-001 during the project analysis.
