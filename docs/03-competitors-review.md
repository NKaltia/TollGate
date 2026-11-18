# 3. Competitors Review

*Draft v1 · November 2026. Feature descriptions come from public documentation and may change. This is not an exhaustive feature comparison.*

The market has two mature categories, **evaluation / observability tools** and **LLM gateways**, plus a newer group of commercial platforms that combine them. Tollgate sits where the two meet.

## 3.1 Evaluation and observability tools

| Tool | What it is | Notes |
|---|---|---|
| **promptfoo** | Open-source CLI and library for evaluating and red-teaming LLM apps | Declarative test cases with assertions; side-by-side model comparison; CI/CD integration with quality-threshold gates; caching and concurrency for runs; strong security / red-teaming focus |
| **Langfuse** | Open-source (MIT) LLM engineering platform | Tracing as its core; datasets and experiments; LLM-as-a-judge, manual labelling and user feedback; prompt management; self-hostable |
| **LangSmith** | Hosted tracing and evaluation platform from the LangChain team | Tracing, datasets and evaluation, closely tied to the LangChain ecosystem |
| **Braintrust** | Hosted evaluation and observability platform | Experiments, datasets, scoring, production logging |
| **Ragas** | Open-source library and metrics for RAG evaluation | Metrics such as faithfulness and answer relevance; a library, not a platform |

## 3.2 LLM gateways

| Tool | What it is | Notes |
|---|---|---|
| **LiteLLM** | Open-source proxy with an OpenAI-compatible API in front of many providers | Retries, fallbacks, budgets per key / team / user, rate limits, Redis-based caching including semantic caching, spend tracking |
| **Portkey** | AI gateway with an open-source core | Unified API, retries, fallbacks, load balancing, simple and semantic caching via configs |
| **Helicone** | Open-source observability proxy and gateway | Request logging, cost and latency tracking, caching, rate limiting, fallbacks |
| **Cloudflare AI Gateway** | Managed gateway on Cloudflare's network | Caching, rate limiting, analytics, logging |

## 3.3 Integrated platforms (gateway + evaluation)

A newer group of commercial platforms combines both categories:

| Tool | What it is | Notes |
|---|---|---|
| **Braintrust** (with Braintrust Gateway) | Evaluation platform with its own gateway | GitHub Action runs eval suites on PRs and blocks merges below thresholds; publishes guidance on evaluation-driven routing using cost / quality / latency Pareto frontiers |
| **Maxim + Bifrost** | Bifrost gateway (open source) with Maxim's evaluation and observability platform | Positioned explicitly around cost-versus-quality visibility |
| **Respan** | Platform combining gateway, observability, evaluations and prompt management | Open-source gateway plus hosted platform |

**Conclusion:** combining a gateway with evaluations is an active commercial direction in 2026. This **validates the problem**: companies pay to answer "what does this optimization cost us in quality?". It also means the project's value is not product novelty.

## 3.4 Tollgate's position

The project does not try to compete with these products on features. Its contribution is a **transparent, measured reference implementation and study** of one specific engineering loop:

1. **Gateway decisions come with quality evidence.** Every gateway policy (cache, similarity threshold, cheaper-model route, fallback) can be run through the evaluation harness, which produces a cost / latency / quality trade-off report. Gateways report savings; Tollgate reports savings *and their quality price*.
2. **Statistical comparison by default.** Comparisons are paired at the test-case level, with confidence intervals and power analysis (following Miller, 2024). The CI gate fails on statistically meaningful drops, not on single-run noise.
3. **The judge is validated before it is trusted.** Agreement of the LLM judge with a human-labelled subset is a required, reported step, not an optional feature.
4. **Measured safety of semantic caching (main research focus).** Gateways such as LiteLLM and Portkey offer semantic caching, and recent research (vCache, MeanCache; see literature review §2.6) shows that static thresholds produce unpredictable false hits. The project measures this end-to-end on a RAG system (savings vs. false hits vs. answer quality across thresholds) and applies a safe policy: caching per request type, isolated per tenant, with conversational replies not cached by default.
5. **Open and reproducible.** Datasets, grader validation, statistics and negative results are all published, so every number can be checked. Commercial platforms typically do not expose their methodology at this level of detail.

In short, the existing tools show that the problem matters. This project contributes an open study of **how** to measure it correctly and what the numbers look like on a real RAG system.

## 3.5 Honest limitations compared with existing tools

- Supports far fewer providers, integrations and UI features than mature products.
- Designed and evaluated on one system under test (a RAG assistant), not a general platform.
- The goal is a well-measured reference system with documented trade-offs, suitable for an academic project, not a commercial product.

## 3.6 Sources

- promptfoo documentation: https://promptfoo.dev/docs/intro/ and https://promptfoo.dev/docs/integrations/ci-cd/
- Langfuse documentation: https://langfuse.com/docs
- LiteLLM proxy documentation: https://docs.litellm.ai/docs/proxy/config_settings
- Portkey feature overview: https://portkey.ai/docs/introduction/feature-overview
- Helicone: https://helicone.ai
- Cloudflare AI Gateway: https://developers.cloudflare.com/ai-gateway/
- LangSmith: https://www.langchain.com/langsmith
- Braintrust: https://www.braintrust.dev
- Ragas: https://arxiv.org/abs/2309.15217
- Braintrust, evaluation-driven model routing: https://www.braintrust.dev/articles/evaluation-driven-model-routing-pareto-frontier
- Braintrust, gateways and observability overview: https://www.braintrust.dev/articles/best-llm-gateways-observability-2026
- Maxim / Bifrost: https://www.getmaxim.ai/bifrost
- Respan: https://respan.ai/articles/best-llm-gateways
