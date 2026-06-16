# PhD / Researcher

> *The methods underneath, with the math.*

This level assumes you can read a research paper, understand likelihoods and posteriors, train a transformer, and follow optimisation theory. It is written for doctoral students, research scientists, and methods-focused engineers who need more than "use scikit-optimize" but less than reading every paper from scratch.

## Part A — Autonomous-lab methods

- **[Bayesian optimisation](bayesian-optimization.md)** — surrogate models, acquisition functions, batch and multi-objective extensions.
- **[Reinforcement learning planners](reinforcement-learning.md)** — when a sequential MDP framing fits an autonomous lab.
- **[LLM scientific reasoning](llm-scientific-reasoning.md)** — language models as planners, with retrieval, tools, and grounding.
- **[Active learning under constraints](active-learning.md)** — the unifying view; what changes when experiments are expensive, batched, and cost-asymmetric.

## Part B — Literature-synthesis methods

- **[Relation extraction](relation-extraction.md)** — sentence- and document-level architectures.
- **[Knowledge-graph construction](kg-construction.md)** — schema design, distant supervision, harmonisation.
- **[Retrieval-augmented generation](rag-systems.md)** — RAG for biomedical search and summarisation, including the failure modes.

## A worked case

- **[Case study — hippocampal sclerosis](case-study-hs.md)** — both halves applied end-to-end to a real neuroscience question, from literature graph to MRI biomarker pipeline to next-experiment selection.

## What this level intentionally skips

- **Production engineering** — see the [engineer](../engineer/index.md) level.
- **First-principles introductions** — see [intermediate](../intermediate/index.md) for those.

## Expectations

Each chapter cites the seminal and modern references. The reading lists are deliberately short; the goal is direction, not completeness. If a method has been benchmarked recently on biomedical data, the chapter mentions which benchmark and what to look for.
