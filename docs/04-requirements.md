# 4. Requirements: Testable MVP Scope

*Draft v1 · November 2026 · Course deliverable: requirements document (due by class 6)*

## 4.1 Scope and stakeholders

### Stakeholders

| Stakeholder | Question | What we need to learn |
|---|---|---|
| **AI engineer** (primary user): builds and changes an LLM feature | "Did my change make the system better or worse, and what did it cost?" | Which quality signals matter, acceptable run time and cost per evaluation |
| **Team lead / product owner** | "Is it safe to release this change? What does the AI feature cost us?" | Release criteria (thresholds), cost visibility |
| **End users of the AI feature** (indirect) | "Will I get correct answers, and is my data kept separate from others?" | Correctness, privacy and isolation requirements |
| **Course team** | "Where will the demo run, and can results be reproduced?" | Deployment and resource constraints, reproducibility |

### Core journey

1. The engineer changes one thing in the system: a prompt, a model, a retrieval setting or a gateway policy.
2. An evaluation run starts, either manually or automatically in CI for a pull request.
3. Tollgate runs the dataset against the new variant, grades the results and compares them with the baseline.
4. The engineer receives a report: better / worse / no significant difference, the list of regressions, and the cost and latency impact.
5. In CI, a significant quality drop blocks the merge.

### Exclusions (out of scope)

Training or fine-tuning models; executing arbitrary generated code; supporting more than one system under test at the same time; a commercial-grade UI; multi-region deployment; Kafka, Kubernetes, microservices.

## 4.2 Prioritized functional requirements

Priorities: **Must** (essential to the agreed first version), **Should** (valuable after the core works), **Could** (if capacity remains), **Later** (explicitly deferred).

| ID | Requirement | Acceptance check (summary) | Priority |
|---|---|---|---|
| R1 | Run a variant on a dataset version and store per-case results and an aggregate score | A known dataset produces a stored run with one result per case and a score | Must |
| R2 | Grade outputs with deterministic graders (retrieval hit@k, answer match, correct refusal) | Hand-made fixtures with known correct / incorrect outputs are graded as expected | Must |
| R3 | Compare a baseline run with a candidate run: paired difference, 95% confidence interval, regression list | A candidate with known injected differences is reported with the correct regressions | Must |
| R4 | CI gate: a pull request fails when quality on the smoke set drops significantly | A deliberately broken variant fails the CI check; an unchanged variant passes | Must |
| R5 | LLM judge for open-ended answers, validated against human labels | Agreement with a hand-labelled subset is computed and reported | Should |
| R6 | Gateway: proxy with timeouts, retries, fallback, and a trace record per model call (model, tokens, cost, latency) | With the primary provider failing, requests still succeed via fallback and every call appears in the trace log | Should |
| R7 | Semantic cache with per-tenant isolation, and a threshold-sweep experiment (RQ4) | Report of hit rate, savings, false hits and quality per threshold; zero cross-tenant hits | Should |
| R8 | Per-tenant budgets; results dashboard | Budget is never exceeded under concurrent requests; dashboard shows run comparisons | Could |
| R9 | Adaptive cache thresholds; multiple systems under test | — | Later |

### Detailed acceptance criteria for the Must requirements

**R1: Run a variant on a dataset version**
- **Given** a valid dataset version with N cases and a reachable system under test,
- **When** the engineer runs the harness for a variant with T trials per case,
- **Then** a run is stored with N × T trial results, each linked to its case, and an aggregate score is reported.
- **Also check (failure):** if the system under test times out or returns an error for a trial, that trial is stored with status `error` and shown separately. It is **not** counted as a wrong answer, and the run finishes with status `completed_with_errors`.
- **Also check (invalid input):** a malformed dataset file is rejected **before** any call is made, with a validation message naming the invalid field.

**R2: Deterministic grading**
- **Given** a test case with an expected answer and expected source document(s),
- **When** a trial output is graded,
- **Then** each grader returns pass/fail with a reason: retrieval hit@k (an expected document is among the top k retrieved), answer match (normalized exact or contains match), and correct refusal (for unanswerable cases, the system declines instead of inventing an answer).
- **Evidence:** unit tests on fixed fixtures; the same input always produces the same grade.

**R3: Compare baseline and candidate**
- **Given** two completed runs on the **same dataset version**,
- **When** the engineer requests a comparison,
- **Then** the report shows the per-metric difference (candidate − baseline) with a 95% confidence interval from a paired comparison, and lists cases that passed in the baseline but fail in the candidate.
- **Also check (failure):** comparing runs on **different** dataset versions is refused with a clear error, because the comparison would not be fair.

**R4: CI regression gate**
- **Given** a pull request that changes the system under test or its configuration,
- **When** the CI workflow runs the smoke set against the PR variant and compares it with the stored baseline,
- **Then** the check fails only if the quality drop is statistically significant at the configured level, and the report is attached to the CI run.
- **Also check (failure):** if the model API is unavailable or the budget is exhausted, the check ends as **failed with an infrastructure reason**, never as a false "passed".

## 4.3 Quality requirements (measurable)

| ID | Requirement | Measurement method and environment |
|---|---|---|
| Q1 | **Reproducibility.** Re-running a variant with recorded model responses yields identical deterministic scores in 100% of cases. | Run the same variant twice with model-response caching enabled; compare per-case results. Local Docker Compose environment. |
| Q2 | **Cost control.** A run never exceeds its configured budget by more than the cost of the calls already in flight at the moment the limit is reached. | Run with a deliberately low budget and concurrency 10; compare recorded spend to the limit. |
| Q3 | **Gateway overhead.** 95% of requests through the gateway add ≤ 50 ms compared with calling the provider directly. | A mock provider with a fixed response replaces the real model; 10 concurrent clients, 1,000 requests; measure p95 of (gateway latency − direct latency). Record hardware and versions. |

Q1 and Q2 apply to the Must scope. Q3 applies once R6 is implemented.

## 4.4 Constraints

| Constraint | Reason / justification |
|---|---|
| Runs locally with Docker Compose on a single laptop; no paid cloud infrastructure required | Course demo environment and zero hosting budget |
| LLM API spend ≤ $10 per month | Personal budget; addressed through cheap models during development, cached model responses and small CI smoke sets |
| Only public datasets (e.g. BEIR SciFact, SQuAD v2, HotpotQA, Quora Question Pairs) | Reproducibility and no private data |
| Single developer, about 5–8 hours per week | Scope is prioritized so that the Must requirements alone form a complete project |
| PostgreSQL + pgvector as the database | Design choice, justified in ADR-001 |

## 4.5 Traceability

| Requirement | Research question | Milestone |
|---|---|---|
| R1, R2 | (baseline for all RQs) | M1 |
| R3 | RQ2 | M2 |
| R4 | RQ3 | M3 |
| R5 | RQ1 | M3 |
| R6 | RQ4 (enabler) | M4 |
| R7 | RQ4 | M5 |

## 4.6 Largest open questions

- **Significance rule for the CI gate (R4):** which test and which threshold balance false alarms against missed regressions. To be decided by experiment in M3.
- **Smoke-set size:** how many cases and trials fit the budget while still detecting a meaningful drop (RQ2).
- **Integration with the system under test:** the exact HTTP contract of the RAG assistant that the harness calls. To be specified in the detailed design.
