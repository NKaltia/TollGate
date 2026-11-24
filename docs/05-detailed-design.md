# 5. Detailed Design

*Draft v0 (simple version) · November 2026 · Course deliverable: detailed design (due by class 8). This is a living document and is kept current as the design evolves.*

## 5.1 Scope of this design

This version covers:
- the overall **architecture** (modules, responsibilities, boundaries);
- the **data model** and its **invariants**;
- one core feature **end to end**: **R1, run a variant on a dataset version**, including its normal flow and an important failure path;
- the main **interface contracts**;
- the **deployment** choice and key **decisions** (ADRs);
- how the design supports **acceptance tests and experiments**.

R3 (comparison), R4 (CI gate) and the gateway features will be designed in more detail in later versions of this document.

## 5.2 Architecture

### Style: modular monolith

One codebase and one deployable application, divided into modules with explicit boundaries. Modules talk to each other only through their public interfaces and never read each other's tables directly. Microservices are not used: there is no team or load that would justify their operational cost (ADR-0002).

```
                    ┌──────────────── Tollgate (one codebase) ─────────────────┐
 Engineer / CI ───▶ │  CLI                                                      │
                    │   │                                                       │
                    │   ▼                                                       │
                    │  Harness module                                           │
                    │   ├─ Dataset loader (validate, version, freeze)           │
                    │   ├─ Runner (trials, concurrency limit, retries)          │
                    │   ├─ Graders (deterministic; LLM judge later)             │
                    │   └─ Comparer + Reporter (R3, R4)                         │
                    │   │                                                       │
                    │   │ SUT adapter (HTTP)                                    │
                    └───┼───────────────────────────────────────────────────────┘
                        ▼
              System under test (AI Second Brain, separate app)
                        │ every model call
                        ▼
                    ┌───────────────── Gateway module (HTTP server) ───────────┐
                    │  Auth (tenant) → Cache lookup → Provider client → Trace  │
                    └───┬───────────────────────────────────────────────────────┘
                        ▼
              LLM providers (Gemini, …)

 Both modules use one PostgreSQL database (+ pgvector), each owning its own tables.
```

### Responsibilities and boundaries

| Part | Responsibility | Boundary |
|---|---|---|
| **CLI** | Parse commands (`run`, `compare`), print results, return exit codes for CI | Calls the harness module's public functions only |
| **Harness: Dataset loader** | Read a dataset file, validate it with a schema, store it as an immutable version | Owns `datasets`, `dataset_versions`, `test_cases` |
| **Harness: Runner** | Create a run, execute N × T trials with a concurrency limit, retry transient errors, record results | Owns `runs`, `trials`; calls the SUT only through the SUT adapter |
| **Harness: Graders** | Turn a trial output into pass/fail plus a reason | Pure functions; owns `grader_results` |
| **Harness: Comparer / Reporter** | Paired comparison, confidence intervals, regression list, report file | Reads runs; writes report files |
| **SUT adapter** | Send a test case to the system under test and normalize its answer | Explicit HTTP contract (§5.5.2); the harness treats the SUT as a black box |
| **Gateway** | Proxy model calls: tenant auth, cache, provider call, trace record | OpenAI-compatible HTTP contract (§5.5.3); owns `tenants`, `gateway_calls`, `cache_entries` |
| **PostgreSQL + pgvector** | Durable storage, transactions, vector similarity search | Accessed only through each module's data layer |
| **External: system under test** | The application being measured (AI Second Brain) | Separate application; must implement the SUT contract |
| **External: LLM providers** | Generate answers and embeddings | Reached only through the gateway |

## 5.3 Data model

```
datasets ──< dataset_versions ──< test_cases
variants
runs  (variant × dataset_version) ──< trials (test_case × attempt) ──< grader_results
tenants ──< gateway_calls        (optionally linked to a run / trial)
tenants ──< cache_entries
```

| Entity | Essential fields | Why it matters |
|---|---|---|
| `datasets` | id, name | Groups versions of the same test set |
| `dataset_versions` | id, dataset_id, version, content_hash, frozen | Fair comparison requires identical test data |
| `test_cases` | id, dataset_version_id, external_id, input (JSONB), expected (JSONB) | One exam question with its answer key |
| `variants` | id, name, config (JSONB), config_hash | Exactly what was tested: prompt, model, retrieval and gateway settings |
| `runs` | id, variant_id, dataset_version_id, trials_per_case, status, started_at, finished_at, summary (JSONB) | One execution of the exam |
| `trials` | id, run_id, test_case_id, attempt_no, status, output (JSONB), latency_ms, error | One attempt at one question |
| `grader_results` | id, trial_id, grader, passed, reason | Why a trial passed or failed |
| `tenants` | id, name, api_key_hash | Isolation of clients in the gateway |
| `gateway_calls` | id, tenant_id, run_id?, trial_id?, request_id, model, input_tokens, output_tokens, cost_usd, latency_ms, upstream_latency_ms, cache_status, request / response (JSONB, redacted) | The trace: what was asked, what it cost, how long it took |
| `cache_entries` | id, tenant_id, request_type, model, prompt_text, embedding (vector), response (JSONB), hit_count, expires_at | Exact and semantic cache |

Status values: run `running | completed | completed_with_errors | failed`; trial `pending | succeeded | error`.

### Invariants (rules that must always hold)

| # | Invariant | How it is enforced |
|---|---|---|
| I1 | A dataset version that has been used by a run is never modified | `frozen = true` on first use; updates rejected; new content creates a new version |
| I2 | Every trial's test case belongs to the run's dataset version | Checked when trials are created; foreign keys |
| I3 | There is at most one trial per (run, test case, attempt) | `UNIQUE (run_id, test_case_id, attempt_no)`: a retried write cannot create a duplicate (idempotent write) |
| I4 | Trials with status `error` are excluded from the quality score and counted separately | Score computation filters by status |
| I5 | Runs are compared only on the same dataset version | Comparer refuses otherwise (R3) |
| I6 | A cache lookup never returns another tenant's entry | Every cache query filters by `tenant_id`; tested explicitly |
| I7 | A completed run has exactly N × T trials | Checked when the run is finalized |

## 5.4 Core feature end to end: R1, run a variant on a dataset version

### Normal flow

| Step | Interaction | Result |
|---|---|---|
| 1 | Engineer runs `tollgate run --dataset squad-mini@v1 --variant prompt-v2 --trials 3` | CLI validates the arguments |
| 2 | Dataset loader loads version `v1` and freezes it (I1) | N test cases available |
| 3 | Runner creates a `run` with status `running` and N × T `pending` trials | Plan of work stored before any call (restartable) |
| 4 | Runner executes trials with a concurrency limit (e.g. 5): for each trial, the SUT adapter sends `POST /ask` to the system under test | The SUT answers; its model calls go through the gateway and are traced |
| 5 | Runner stores each output; graders grade it (retrieval hit@k, answer match, correct refusal) | `trials` → `succeeded`, `grader_results` stored |
| 6 | When all trials are finished, the runner computes the summary and sets the final status | Run `completed`; CLI prints the score and the run ID |

### Important failure path: the system under test times out

| Step | Interaction | Result |
|---|---|---|
| 4a | `POST /ask` exceeds the timeout (e.g. 30 s) or returns 5xx | Adapter raises a *transient* error |
| 4b | Runner retries up to 2 times with exponential backoff | If a retry succeeds, the flow continues normally |
| 4c | All retries fail | Trial stored with status `error` and the error message; **not** graded as a wrong answer (I4) |
| 6' | Run finishes with at least one error trial | Run status `completed_with_errors`; report lists error trials separately |

Other failure paths (summary): a malformed dataset file is rejected at step 2, before any call (Zod schema validation). If the SUT is unreachable for every trial, the run ends `failed` with an infrastructure reason. A crash of the harness process leaves `pending` trials that can be resumed, and I3 prevents duplicate trials.

## 5.5 Interface contracts

### 5.5.1 CLI

| Item | Specification |
|---|---|
| Operation | `tollgate run --dataset <name>@<version> --variant <name> [--trials T] [--concurrency C]` |
| Success | Exit code 0; prints run ID, score, number of error trials |
| Expected failures | Exit code 2: invalid arguments or dataset; exit code 3: SUT unreachable (run `failed`) |
| Unexpected failure | Exit code 1 with a short message; full details in the log |

### 5.5.2 SUT adapter contract (what the system under test must implement)

| Item | Specification |
|---|---|
| Operation | `POST /ask` |
| Request | `{ "question": string, "case_id": string, "run_id": string }` (IDs are forwarded to the gateway for trace linking) |
| Success (200) | `{ "answer": string, "retrieved_document_ids": string[], "refused": boolean }` |
| Expected failures | `400` invalid input; `503` dependency unavailable; timeout. Timeouts and 5xx are retried, 4xx are not |
| Unexpected failure | Treated as a transient error: retry, then trial `error` |

### 5.5.3 Gateway contract

| Item | Specification |
|---|---|
| Operation | `POST /v1/chat/completions` (OpenAI-compatible) and `POST /v1/embeddings` |
| Headers | `Authorization: Bearer <tenant API key>`; optional `X-Run-Id`, `X-Trial-Id`, `X-Request-Type` |
| Success | Provider-compatible response; header `X-Tollgate-Cache: hit | miss | bypass` |
| Expected failures | `401` unknown tenant; `429` budget or rate limit exceeded; `502` provider error after retries |
| Unexpected failure | `500` with a request ID; the trace record stores the error |
| Side effect | Exactly one `gateway_calls` record per client request |

## 5.6 Deployment

**Simple starting point:**
- one codebase with two entry points: the **CLI** (harness) and the **gateway HTTP server**;
- **PostgreSQL + pgvector** in Docker;
- **Docker Compose** starts the database and the gateway with one command.

The system under test (AI Second Brain) runs as a separate application configured to send its model calls to the gateway. In CI, GitHub Actions starts the same Compose setup and runs the CLI.

**Reasons that would justify more complexity** (not present now): a measured bottleneck in the gateway, or a need to deploy the gateway independently for real users.

## 5.7 Decision records (summaries)

| ADR | Decision | Main reason | Cost we accept | Reconsider when |
|---|---|---|---|---|
| 0001 | PostgreSQL + pgvector | Statistics (SQL), transactions (budgets, idempotent writes), vector search for the semantic cache, all in one database | Less flexible than a document store; vector search less specialized than a dedicated vector DB | Vector search becomes a measured bottleneck |
| 0002 | Modular monolith | Solo developer, small load; simple deployment and debugging; clear module boundaries keep the option to split later | Modules share one database and one release cycle | A module needs independent scaling or deployment |
| 0003 | The harness treats the SUT as a black box over HTTP | The harness works with any system that implements `/ask`; no coupling to the SUT's internals | An HTTP contract must be implemented in the SUT | — |

Full ADRs are in `docs/adr/`.

## 5.8 How the design supports acceptance tests and experiments

- **Mock system under test.** A tiny fake SUT with predefined answers makes R1–R3 tests deterministic. It is used for injected-regression tests (R3, R4).
- **Mock provider** behind the gateway: used for the gateway overhead measurement (Q3) and for failure injection (timeouts, 5xx).
- **Recorded model responses** (cache in "replay" mode) make re-runs reproducible (Q1) and free.
- **Versioned datasets and variant `config_hash`** ensure every result can be traced back to exactly what was tested.
- **Trace records linked to runs and trials** (`X-Run-Id`, `X-Trial-Id`) connect quality results with cost and latency: the basis of the RQ4 experiment.

### Exit check: where does R1 become executable behaviour?

**Component:** Harness Runner. **Contract:** CLI `tollgate run` (§5.5.1) and SUT `POST /ask` (§5.5.2). **Tests:**
1. With the mock SUT and a 3-case dataset, `run --trials 2` stores exactly 6 trials and the expected score.
2. With the mock SUT configured to time out on one case, that case's trials are `error`, the score excludes them, and the run is `completed_with_errors`.

## 5.9 Open questions

- Concurrency default and timeouts: to be tuned against the SUT and the API budget.
- Exact statistical method for R3/R4 (bootstrap vs. paired test): next version of this document.
- How AI Second Brain exposes `retrieved_document_ids` (may require a small change in the SUT).
