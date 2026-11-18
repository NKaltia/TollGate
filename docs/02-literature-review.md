# 2. Literature Review

*Draft v1 · November 2026*

This review covers the six areas the project builds on (the sixth, semantic caching, is §2.6):
1. what evaluations of LLM systems are and why practitioners consider them essential;
2. using LLMs as judges, and the limits of that approach;
3. statistical rigour in evaluations;
4. evaluation of retrieval-augmented generation (RAG);
5. reliability patterns for calling unreliable external services.

---

## 2.1 Evaluations as engineering practice

**Anthropic, "Demystifying evals for AI agents" (2026).**
https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

The article defines an eval as a test for an AI system: an input is given, and grading logic is applied to the output to measure success. It argues that without evals teams get stuck in reactive loops, where problems are found only in production and fixing one failure creates others. The article describes the structure of an evaluation harness (tasks, repeated trials, graders, transcripts, outcomes) and three kinds of graders: code-based, model-based and human. It also separates **capability** evals from **regression** evals.

*Relevance:* this gives the vocabulary and overall architecture of the evaluation harness (Part A). In particular, it supports checking **outcomes**, not only the final text.

**Hamel Husain, "Your AI Product Needs Evals" (2024).**
https://hamel.dev/blog/posts/evals/

A practitioner guide to building domain-specific evaluation systems, based on real product work. It stresses looking at traces, doing error analysis, and the pros and cons of automated LLM-based evaluation. It also points out that, unlike unit tests, a 100% pass rate is not required; the acceptable pass rate is a product decision.

*Relevance:* supports making trace logging part of the system (the gateway) and treating thresholds in the CI gate as explicit, documented decisions.

## 2.2 LLM-as-a-judge

**Zheng et al., "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena" (NeurIPS 2023).**
https://arxiv.org/abs/2306.05685

The paper studies using strong LLMs to grade open-ended answers. It identifies **position bias, verbosity bias, self-enhancement bias** and limited reasoning ability. It reports that strong judges reached over 80% agreement with human preferences, which is about the level of agreement between humans.

*Relevance:* LLM judges are practical, but they have known, measurable biases. The project therefore treats the judge as a component that must be **validated against human labels** before its scores are trusted (RQ1). It does not assume the judge is correct.

## 2.3 Statistics for evaluations

**Evan Miller, "Adding Error Bars to Evals: A Statistical Approach to Language Model Evaluations" (2024).**
https://arxiv.org/abs/2411.00640

The paper notes that evals are usually reported with a "highest number is best" mindset and without statistical testing. It proposes treating eval questions as a sample from a larger population, reporting standard errors, comparing two models with **paired differences at the question level**, and using **power analysis** to decide whether an eval is large enough to detect a given effect.

*Relevance:* this is the methodological basis for the harness's comparison logic. It directly informs RQ2 (how many cases and trials are needed) and the design of a CI gate that does not fail on random noise.

## 2.4 Evaluating RAG systems

**Es et al., "Ragas: Automated Evaluation of Retrieval Augmented Generation" (2023).**
https://arxiv.org/abs/2309.15217

Proposes reference-free metrics for RAG pipelines. They evaluate separately whether retrieval finds relevant and focused context, whether the LLM uses that context **faithfully**, and the quality of the generated answer.

*Relevance:* supports evaluating the RAG system under test **per component** (retrieval vs. generation), so that a regression can be traced to the step that caused it.

**Thakur et al., "BEIR: A Heterogeneous Benchmark for Zero-shot Evaluation of Information Retrieval Models" (NeurIPS 2021 Datasets and Benchmarks).**
https://arxiv.org/abs/2104.08663

A benchmark of 18 public retrieval datasets with a common evaluation framework. The authors find BM25 to be a robust baseline across domains.

*Relevance:* provides public datasets with relevance judgments (for example SciFact) for evaluating retrieval without inventing a dataset. It also gives a standard lexical baseline to compare against.

## 2.5 Reliability of calls to external services

**Stripe Engineering, "Designing robust and predictable APIs with idempotency" (B. Leach).**
https://stripe.com/blog/idempotency

The article explains that networks are unreliable, so retries are necessary, and retries are only safe when operations are **idempotent**. It describes idempotency keys and retrying with exponential backoff and jitter, which avoids a "thundering herd" of simultaneous retries.

*Relevance:* this is the basis for the gateway's retry policy and for ensuring that a retried model call is not executed or paid for twice.

## 2.6 Semantic caching for LLMs

**Schroeder et al., "vCache: Verified Semantic Prompt Caching" (ICLR 2026).**
https://arxiv.org/abs/2502.03771

Semantic caches reuse a stored response when a new prompt's embedding is similar enough to a cached one. The paper shows that the usual **static similarity threshold** gives no correctness guarantee, leads to unexpected error rates, and produces suboptimal hit rates. It proposes learning a separate threshold for each cached entry, with a user-defined error-rate bound.

**Gill et al., "MeanCache: User-Centric Semantic Caching for LLM Web Services" (IPDPS 2025).**
https://arxiv.org/abs/2403.02694

Points out that central semantic caches raise privacy and scalability concerns, and that they produce high **false-hit** rates on queries that look similar but differ in context. Proposes a privacy-preserving, per-user cache and reports fewer false hits than GPTCache.

*Relevance:* both papers confirm that semantic caching can return wrong answers, and that a single global threshold is risky. RQ4 uses this as its starting point. Rather than proposing a new algorithm, the project measures the risk **end-to-end on a RAG system**: false hits *and* the resulting drop in answer quality, using the evaluation harness. It also enforces per-tenant isolation, which addresses the privacy concern MeanCache raises.

---

## 2.7 Summary: what the literature implies for the design

| Finding | Design consequence |
|---|---|
| Evals are a core engineering practice; check outcomes, not just text | Evaluation harness is the project's core; graders check retrieval and answer outcomes |
| LLM judges are useful but biased | Judge is validated against a human-labelled subset before use (RQ1) |
| Eval results without error bars are misleading | Paired comparisons, confidence intervals, power analysis (RQ2) |
| RAG quality has separate retrieval and generation dimensions | Per-component metrics; regressions traced to a step |
| Retries need idempotency and backoff | Gateway retry design with idempotent handling and jittered backoff |
| Static semantic-cache thresholds have unpredictable error rates; shared caches raise privacy risks | Threshold sweep with false-hit and quality measurement; per-tenant cache isolation (RQ4) |

## 2.8 Further reading (to be reviewed)

- Martin Kleppmann, *Designing Data-Intensive Applications* (O'Reilly): chapters on reliability and on handling faults in distributed systems.
- Google, *Site Reliability Engineering*, chapters on handling overload and cascading failures.
- OpenTelemetry semantic conventions for generative AI, as a possible format for gateway traces.
