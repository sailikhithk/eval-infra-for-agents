# Eval Infrastructure for Agent Systems

> A field guide to building production-grade evaluation infrastructure for LLM agent systems.
> Based on real production work at Airbnb (BPI Virtual Analyst), Shell (NLP classification),
> and contributions to LangChain, LiveKit, and Ragas.

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://python.org)
[![LangChain](https://img.shields.io/badge/LangChain-0.3+-green.svg)](https://github.com/langchain-ai/langchain)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-WIP%20Ch%201--3-orange.svg)](#chapters)

## Why This Exists

Most LLM agent evaluation is ad hoc. Teams ship agents to production with manual spot-checks,
vibe-based quality assessments, and no systematic way to detect regressions when prompts,
models, or retrieval pipelines change. This handbook is a field guide for ML engineers who
need to build real evaluation infrastructure: the kind that runs 23+ agent versions against
1,690 versioned ground-truth samples, detects regressions before they reach production, and
gives the team confidence to ship.

This is not a survey paper. Every pattern in this handbook comes from production systems I
have built or operated. The code examples are simplified but structurally faithful to what
runs in production at Airbnb's Business Process Insight (BPI) Virtual Analyst and Redpen
data-labeling systems.

## What You Will Learn

- How to build a reproducible evaluation harness that runs multiple agent versions against
  versioned ground-truth samples
- How to design dual-model A/B comparison and flip-analysis to detect regressions
- How to orchestrate 30+ LLM integrations behind a unified abstraction (FacadeDriver pattern)
- How to implement PII-safe batching and streaming pipelines for eval at scale
- How to track LLM costs accurately across providers (the gap that LangChain PR #39351 fixed)
- How to score agent reliability after a call ends (the LiveKit ReliabilityObserver pattern)
- How to build a culture of test-first agent development

## Chapters

| # | Title | Status | Topic |
|---|-------|--------|-------|
| 1 | [The Eval Problem](chapters/01-the-eval-problem.md) | Draft | Why ad hoc eval fails at scale |
| 2 | [Ground Truth: Building a Versioned Sample Store](chapters/02-ground-truth.md) | Draft | 1,690 samples, versioning, PII redaction |
| 3 | [The FacadeDriver Pattern: Multi-LLM Orchestration](chapters/03-facadedriver.md) | Draft | 30+ models behind one interface |
| 4 | Dual-Model A/B and Flip Analysis | Planned | Detecting regressions across agent versions |
| 5 | Cost Tracking at Scale | Planned | LangChain #39351, litellm, per-provider token accounting |
| 6 | Reliability Scoring After Call End | Planned | LiveKit #6754, ReliabilityObserver pattern |
| 7 | PII-Safe Batching and Streaming | Planned | Microsoft Presidio, 12 entity types, 30% faster |
| 8 | Scaling from 600 to 10,000 Rows Per Run | Planned | Pipeline architecture, Airflow DAGs, consistency |
| 9 | Building an Agent Quality Program | Planned | 23+ versions, F1 0.654, project-best tracking |
| 10 | Open-Source Eval Tools: Ragas, LangSmith, Braintrust | Planned | Integration patterns and gaps |
| 11 | Test-First Agent Development | Planned | 95% coverage methodology from Southwest Airlines |
| 12 | Production Checklist: Shipping an Agent Safely | Planned | The 12-point pre-production checklist |

## Companion Notebooks

| # | Notebook | Topic |
|---|----------|-------|
| 1 | [ground_truth_store.ipynb](notebooks/ground_truth_store.ipynb) | Build a versioned ground-truth sample store |
| 2 | [facadedriver_abstraction.ipynb](notebooks/facadedriver_abstraction.ipynb) | Implement the FacadeDriver pattern |
| 3 | [dual_model_ab_flip_analysis.ipynb](notebooks/dual_model_ab_flip_analysis.ipynb) | Run A/B comparison and flip-analysis |

## Who This Is For

- ML infrastructure engineers building production LLM agent systems
- Data science teams transitioning from batch ML to agentic workloads
- Engineering managers who need to ship agents with confidence
- Open-source contributors to LangChain, LiveKit, Ragas, and similar eval tooling

## Production Provenance

This handbook is based on production work at:

| Company | System | Scale |
|---------|--------|-------|
| Airbnb | BPI Virtual Analyst | 55+ analysts, 23+ agent versions, 1,690 ground-truth samples |
| Airbnb | Redpen BigAir | 19 production configs, 11+ languages, Airflow DAGs |
| Shell | NLP drilling-loss classifier | 86% to 94% accuracy, SageMaker deployment |
| Southwest Airlines | Flight-tracking pipelines | 4M req/min, 95% test coverage, MTTR 45 to 12 min |
| Eli Lilly | AMYVID dose management | 99.9% uptime, 10-hour isotope-decay window |

## Open-Source Contributions Referenced

| PR | Repo | Fix |
|----|------|-----|
| [#39351](https://github.com/langchain-ai/langchain/pull/39351) | LangChain | Perplexity cost tracking (num_search_queries in UsageMetadata) |
| [#6754](https://github.com/livekit/agents/pull/6754) | LiveKit Agents | ReliabilityObserver for external reliability scoring |
| [#2954](https://github.com/vibrantlabsai/ragas/pull/2954) | Ragas | Deprecated top_p for Anthropic provider |
| [#1101](https://github.com/Shubhamsaboo/awesome-llm-apps/pull/1101) | awesome-llm-apps | Deprecated pathlib and PyPDF2 migration |

## Author

**Sai Likhith Kanuparthi** - Senior ML / AI Infrastructure Engineer at Airbnb
- GitHub: [sailikhithk](https://github.com/sailikhithk)
- Portfolio: [sailikhith.me](https://sailikhith.me)
- LinkedIn: [sailikhithk](https://linkedin.com/in/sailikhithk)

## License

MIT - see [LICENSE](LICENSE)

## Contributing

This is a living handbook. If you find errors, have production patterns to share, or want
to contribute a chapter, please open an issue or PR. All contributions will be attributed.
