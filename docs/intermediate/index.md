# Intermediate

> *Components, tools, and worked examples. Some Python helps.*

This level shows you the actual moving parts. After it you should be able to read a paper or a vendor brochure about an autonomous lab or a literature-synthesis pipeline and identify which pieces it has, which it skips, and where the risk lives.

## Part A — Autonomous labs

- **[Closed-loop systems](closed-loop.md)** — the loop in detail; what "closed" really requires.
- **[AI planners](ai-planner.md)** — the planner's job, the families of methods (BO, RL, LLM), how they trade off.
- **[Lab robotics](robotic-equipment.md)** — what real hardware does, what it can't do, glue layers.
- **[Experiment data analysis](data-analysis.md)** — turning measurements into the numbers the planner consumes.

## Part B — Literature synthesis

- **[Biomedical NLP](biomedical-nlp.md)** — entity recognition, normalisation, relation extraction in practice.
- **[Knowledge graphs](knowledge-graphs.md)** — building, querying, and not lying with a KG.
- **[Hypothesis generation](hypothesis-generation.md)** — link prediction, abductive reasoning, LLM brainstorming.
- **[Systematic-review support](systematic-review.md)** — PRISMA-compatible pipelines, living reviews.

## How to read this level

Read each part in order. Each chapter ends with a "where to next" section pointing into the [PhD](../phd/index.md) level if you want the math, or the [engineer](../engineer/index.md) level if you want the production patterns.

## What "intermediate" assumes

- You can read Python and call libraries.
- You know what an embedding is, even if you couldn't derive one.
- You can spot a workflow DAG.
- You are willing to install MkDocs, run a notebook, or wire a small CLI.

If any of that is unfamiliar, start with the [beginner](../beginner/index.md) section or skim [NeuroStack's foundations chapter](https://phindagijimana.github.io/neuro_stack/fundamentals/foundations/python/).

## What this level intentionally skips

- **Most of the math.** Acquisition functions, GP kernels, transformer training — those live in the [PhD](../phd/index.md) chapters.
- **Most of the platform work.** Schedulers, observability, runbooks, governance — those live in the [engineer](../engineer/index.md) chapters.
