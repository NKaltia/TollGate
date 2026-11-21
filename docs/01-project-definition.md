# 1. Project Definition

*Draft v1 · November 2026*

## 1.1 Problem

LLM-based features are now common in production software. Engineering them reliably is still hard, for three connected reasons.

1. **Changes are risky and hard to verify.** LLM outputs are non-deterministic, and there is often no single correct answer. A change to a prompt, model, retrieval step or configuration can improve one behaviour and silently break another. Classic unit tests do not catch this. Many teams still check changes by manually trying a few examples.
2. **Model calls are unreliable and expensive.** Provider APIs time out, rate-limit and fail. Every call costs money, and costs grow quickly with traffic.
3. **Optimizations have hidden quality costs.** The usual ways to cut cost and latency are caching, routing to cheaper models and falling back to other providers. Each can lower answer quality, and teams rarely measure by how much.

The third problem is the focus of this project. Mature open-source tools exist for evaluation and, separately, for gateways. Commercial platforms have recently started combining the two (see the [competitors review](03-competitors-review.md)), which shows the problem is real. The project builds this connection as an open, measured system: every gateway optimization comes with statistically sound evidence about its effect on quality.

## 1.2 Goal

Build a system that lets an engineer answer, with evidence:

> "Is version B of my LLM system better than version A, how sure am I, and what does it cost?"

The same question applies to gateway policies:

> "This cache or routing rule saves X% of cost. How much quality does it lose?"

## 1.3 Solution overview

```
 Evaluation harness  ──runs test set──▶  System under test (RAG assistant)
        │                                        │ every model call
        │                                        ▼
        │                                  LLM gateway ──▶ model providers
        │                                        │
        └──────── reads results, traces, cost ◀──┘  (per-call trace log)
```

### Part A: Evaluation harness (core)

- **Versioned datasets.** Test cases with inputs and expected outcomes, stored with versions so that two runs are always compared on the same data.
- **Repeated trials.** Each case runs several times, because the outputs are non-deterministic.
- **Graders**, in increasing cost order:
  1. *deterministic*: exact or structural checks, such as whether the correct document was retrieved or the output is valid;
  2. *LLM-as-judge*: a model grades open-ended answers against a rubric;
  3. *human labels*: a small hand-labelled subset used to **validate the LLM judge** before trusting it.
- **Statistical comparison.** Paired comparison of two versions on the same cases, with confidence intervals. The result is not just "85% vs 78%" but whether that difference is real.
- **Regression detection.** A list of cases that passed before and fail now.
- **CI gate.** A small "smoke" set runs on every pull request in GitHub Actions, and the full set runs nightly. A statistically significant drop in quality fails the build.
- **Report.** A readable run report (Markdown or HTML first; a web dashboard is a later stage).

### Part B: LLM gateway (module)

An OpenAI-compatible proxy between the application and model providers:

- timeouts and **retries with exponential backoff**;
- **fallback** to an alternative model or provider, and a circuit breaker;
- **caching**: exact-match first, semantic caching as an experiment. Caching is enabled per request type and isolated per tenant. Conversational replies are not cached by default, because repeating identical answers hurts the user experience and a shared cache can leak data between users;
- **budgets** per tenant, with correct accounting under concurrent requests;
- **idempotent handling of retried calls**, so that a retry after a failure is not paid for twice;
- **per-call trace log**: input, output, model, tokens, cost and latency, grouped by run or request ID. This provides agent and RAG tracing as a by-product of the gateway, without a separate SDK.

### How the parts connect

Every gateway policy can be evaluated by the harness:

| Gateway policy | Saves | Quality risk measured by the harness |
|---|---|---|
| Exact / semantic cache | cost, latency | stale or wrong reused answers |
| Cheaper model routing | cost | lower correctness |
| Fallback provider | availability | different answer quality |

### Where test cases come from

1. **Public datasets with known answers** (e.g. SQuAD v2, BEIR SciFact, Quora Question Pairs). Used for the research results, because they provide ground truth and are reproducible.
2. **Real failures captured by the gateway.** When the system gives a bad answer, its trace (the conversation and every model call) can be **promoted to a test case** after review: the input, plus the behaviour the system should have shown. The case stays in the dataset as a permanent regression test.

This forms a continuous loop: *failure → trace → new test case → configuration change → evaluation on all cases → release only if nothing else regressed*. The gateway and tracing run all the time; evaluations run on every change (smoke set in CI) and before major upgrades (full set).

### Privacy of real user data

Turning real conversations into test cases means storing user data, so:
- a trace becomes a test case only through an **explicit, reviewed action**, never automatically;
- **personal data is redacted** (names, emails, phone numbers, IDs) before a case is stored;
- traces and cases are **isolated per tenant** and have a **retention limit**;
- in this project, real-user cases come only from the author's own system and consenting test users; **all published research results use public datasets only**.

## 1.4 System under test

The first target is the author's existing RAG knowledge assistant (TypeScript, PostgreSQL + pgvector, Gemini API), evaluated on public datasets with known answers, for example **BEIR SciFact** for retrieval and **HotpotQA** / **SQuAD v2** for question answering, including unanswerable questions. A tool-using agent may be added in a later stage.

## 1.5 Research questions

- **RQ1.** How closely does an LLM judge agree with human labels on this task, and how stable is it across runs?
- **RQ2.** How many trials and test cases are needed to reliably detect a quality change of a given size?
- **RQ3.** Can the CI gate detect deliberately injected regressions (for example worse chunking or truncated context), and what is its false-alarm rate?
- **RQ4 (main research focus).** How safe is semantic caching for a RAG system? For a range of similarity thresholds, what are the hit rate, cost and latency savings, **false-hit rate** (a cached answer returned for a question that is actually different), and the end-to-end drop in answer quality measured by the harness? Is there a threshold, or a per-request-type policy, that saves cost without measurable quality loss, and does tenant isolation hold?

### RQ4 method (summary)

1. **Ground truth.** Question pairs labelled as duplicate / not duplicate (e.g. the public Quora Question Pairs dataset), plus the project's own Q&A test set with known answers. Hard negatives, such as "How do I cancel my order?" vs "How do I cancel my subscription?", are the main risk case.
2. **Baseline.** Cache disabled: quality, cost, latency.
3. **Sweep.** Semantic cache at several thresholds (e.g. 0.80 / 0.85 / 0.90 / 0.95). For each: hit rate, savings, false hits (using the labels), and quality via the harness.
4. **Result.** A "savings vs. errors" curve and a justified policy, compared against an exact-match cache and against no cache.
5. **Isolation check.** Zero cross-tenant cache hits.

This reproduces, on a concrete RAG system and with end-to-end quality measurement, a problem that recent research has identified: static similarity thresholds give unpredictable error rates ([literature review](02-literature-review.md), §2.6). The project does not propose a new caching algorithm.

## 1.6 Scope

**In scope:** one system under test (the RAG assistant), a small number of carefully checked datasets, deterministic and LLM-based graders with human validation, statistical comparison, CI gate, gateway with retries, fallback, caching, budgets and tracing, and written reports.

**Out of scope:** training or fine-tuning models, running arbitrary generated code in a sandbox, multi-region deployment, Kafka / Kubernetes / microservices, and a full commercial-grade UI.

## 1.7 What this project is not: "Isn't this an LLM wrapper?"

An LLM wrapper is an application whose value comes from the model's output (a chat UI on top of an API). Here the model is the **object under test**, not the product:

- The harness works the same with any model, or with a deterministic mock model. Its value is in **measurement**: experimental design, statistics, grader validation and regression detection.
- The gateway is **distributed-systems engineering**: retries, idempotency, circuit breaking, concurrency-safe budgets and cache isolation.
- The output of the project is **evidence about engineering decisions**, not generated text.

## 1.8 Stages

Each stage ends with a working, demonstrable result.

1. **Baseline harness.** CLI, versioned dataset, one model adapter, deterministic graders; first measured score of the RAG assistant.
2. **Persistence and comparison.** Runs stored in PostgreSQL, repeated trials with a concurrency limit, paired comparison with confidence intervals.
3. **LLM judge and CI.** Rubric-based judge validated against a human-labelled subset, GitHub Actions regression gate, run report.
4. **Gateway.** Proxy with timeouts, retries, fallback, circuit breaker, cost and latency tracing.
5. **Optimizations measured.** Caching (per tenant, per request type), budgets, cheaper-model routing, each with a quality-impact report from the harness.
6. **Optional.** Web dashboard, tool-using agent as a second system under test.

Stages 1–3 alone form a complete project. Stages 4–5 add the systems-engineering part and the main research result (RQ4).

## 1.9 Technology decisions

The core stack is TypeScript, Node.js / Express, GitHub Actions and Docker, with React for a later dashboard. **Database: PostgreSQL + pgvector** (preliminary decision; the full reasoning will be written up as **ADR-001** in the project analysis). The alternatives considered:

| | PostgreSQL (+ pgvector) | MongoDB (+ Mongoose) |
|---|---|---|
| Traces and run results (JSON of varying shape) | JSONB columns | Native document model |
| Statistics: joins, grouping, paired comparisons | SQL, strong fit | Aggregation pipeline, less natural |
| Concurrent budget accounting | Transactions, row locks | Atomic update operators (`$inc`) |
| Semantic cache similarity search | pgvector in the same database | Atlas Vector Search; local option to be verified |

**Why PostgreSQL:** the core of the project is statistical comparison of runs, consistent budget accounting and similarity search for the semantic cache. PostgreSQL covers all three in one database: SQL for analysis, transactions, pgvector. Its JSONB columns are flexible enough for traces of varying shape. Running two databases would add operational complexity with no measurable benefit at this scale.

## 1.10 Deliverables

- Source code with tests and CI.
- Datasets and the human-labelled validation subset.
- Architecture Decision Records (ADRs) for key design choices.
- Experiment reports answering RQ1–RQ4, including negative results.
- Final project report and demo.

## 1.11 Main risks

| Risk | Mitigation |
|---|---|
| Weak or noisy dataset makes results meaningless | Start with public datasets that have known answers; keep a held-out set; label carefully |
| API cost of many trials and LLM judging | Cheap models during development, cached model responses for reproducibility, budgets |
| Scope growth (a "clone of an existing platform") | Fixed single system under test; stages 1–3 are the minimum complete project |
| Statistical mistakes | Follow published guidance on eval statistics ([literature review](02-literature-review.md), §2.3) |
