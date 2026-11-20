# Roadmap

*v1 · November 2026. Dates are targets, not promises; adjust after each milestone review.*

Legend: 🔴 understand deeply · 🟠 solid working understanding · 🟢 awareness

**Every milestone ends with:**
1. its Definition of Done met;
2. ADRs written for the decisions made;
3. learning log updated;
4. a **milestone interview** (oral check of the 🔴 topics, done outside the IDE) before the next milestone starts.

---

## M0: Project analysis and setup *(target: early December)*

**Goal:** know exactly what to build before building it; have a working skeleton.

- Project analysis document: functional and non-functional requirements, main use cases, architecture diagram, data flow, first data model.
- ADR-001 (database), ADR-002 (repository structure: monorepo with packages).
- Repo setup: TypeScript strict, ESLint/Prettier, Vitest, Docker Compose with PostgreSQL + pgvector, `.env.example`, GitHub Actions running lint + tests.

**Concepts**
- 🔴 Requirements vs. architecture; data flow through the system; relational data model (entities and relations: dataset → case, run → trial → grader result)
- 🟠 Docker Compose; environment variables and secrets; CI pipeline basics; monorepo layout
- 🟢 Linter and formatter configuration; tsconfig options

**Done when:** analysis doc reviewed with the supervisor; `docker compose up` gives a running database; CI is green on an empty test.

---

## M1: Baseline harness *(target: late December)*

**Goal:** the first real number: "my RAG answers X% of these questions correctly".

- Dataset format with versions; loader for a small public subset (e.g. 50–100 cases from SQuAD v2 / BEIR SciFact).
- Adapter that calls the system under test (AI Second Brain) over HTTP.
- Deterministic graders: retrieval hit@k, answer contains or matches expected, "no answer" handling.
- CLI: `tollgate run --dataset … --variant …` printing a summary.

**Concepts**
- 🔴 What an eval is (case, trial, grader, outcome); deterministic grading; separating retrieval quality from answer quality; adapter pattern (harness independent of the SUT)
- 🟠 Dataset versioning; CLI design; HTTP client with timeouts
- 🟢 Dataset licences and formats

**Done when:** a reproducible baseline score of the RAG system is printed from one command.

---

## M2: Persistence, repeated trials and statistics *(target: January, around exams)*

**Goal:** turn "a number" into "a trustworthy comparison".

- Store runs, trials and grader results in PostgreSQL.
- N trials per case, with a concurrency limit for API calls; retries for transient errors.
- Paired comparison of baseline vs. candidate per case; confidence intervals (bootstrap); regression list (passed before, fails now).
- Markdown/HTML run report.

**Concepts**
- 🔴 Non-determinism and why repeated trials are needed; paired comparison; confidence intervals; SQL joins and aggregation for comparison; concurrency limits
- 🟠 Bootstrap resampling (intuition + implementation); transactions when writing run results; report generation
- 🟢 Power analysis (concept only at this stage)

**Done when:** a report shows "candidate vs. baseline: +X% (95% CI …), regressions: …" for two real variants (e.g. two prompts).

---

## M3: LLM judge and CI gate *(target: February–March)*

**Goal:** grade open-ended answers, and only trust the judge after checking it.

- Rubric-based LLM judge.
- Small labelling flow: Nikita labels 100–200 outputs by hand.
- Judge validation: agreement with human labels (accuracy, Cohen's kappa), known biases.
- GitHub Actions gate: small smoke set on each PR; fail on a significant quality drop.
- Injected-regression test: deliberately break the SUT and check that the gate catches it (RQ3).

**Concepts**
- 🔴 LLM-as-judge and its biases; validating a judge against human labels; why the CI gate must tolerate noise (false alarms vs. missed regressions)
- 🟠 Prompting a judge for structured output; agreement metrics; GitHub Actions workflows and secrets
- 🟢 Other judge techniques (pairwise judging, multiple judges)

**Done when:** judge agreement is measured and reported (RQ1); the CI gate catches injected regressions with a known false-alarm rate.

---

## M4: LLM gateway core *(target: March–April)*

**Goal:** a reliable proxy between the SUT and model providers, with tracing.

- OpenAI-compatible HTTP proxy; the SUT routes all model calls through it.
- Timeouts, retries with exponential backoff and jitter, fallback model, circuit breaker.
- Idempotent handling of retried calls.
- Trace log: one row per model call (tenant, run/request ID, model, tokens, cost, latency, input/output).

**Concepts**
- 🔴 Proxy / gateway architecture; retries, backoff and jitter; idempotency; circuit breaker states; trace data model
- 🟠 Streaming vs. non-streaming responses; cost calculation from token usage; API keys per tenant
- 🟢 OpenTelemetry GenAI conventions

**Done when:** killing or slowing the primary provider (simulated) still yields answers via retries/fallback, and every call is visible in the trace log with cost and latency.

---

## M5: Cache and budgets: the main experiment *(target: April–May)*

**Goal:** answer RQ4: how safe is semantic caching for RAG?

- Exact-match cache; semantic cache with pgvector similarity search.
- Per-tenant isolation; caching enabled per request type.
- Budgets with correct accounting under concurrent requests.
- Threshold sweep (e.g. 0.80 / 0.85 / 0.90 / 0.95) on Quora Question Pairs and the Q&A set: hit rate, savings, false hits, quality via the harness.
- Results report with a "savings vs. errors" curve and a justified policy.

**Concepts**
- 🔴 Embeddings and cosine similarity; semantic cache design and false hits; race conditions and transactions/locks for budgets; experiment design (one variable at a time)
- 🟠 pgvector indexes; cache invalidation and TTL; tenant isolation tests
- 🟢 Adaptive thresholds from research (vCache) as future work

**Done when:** the RQ4 report exists with real numbers, including any negative results.

---

## M6: Results, dashboard and defense *(target: June)*

- React dashboard for runs, comparisons and the cache experiment (optional, time-boxed).
- Final project report; demo script; defense preparation.

**Concepts**
- 🟠 React data fetching and charts
- 🔴 Being able to explain the whole system end-to-end, every ADR, and every result

---

## Scope rule

Stages M0–M3 alone form a complete project. M4–M5 add the gateway and the main research result. M6 is presentation. If time runs short, cut from the end, never the quality of M0–M3.
