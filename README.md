# AutoLabs

> *Autonomous Labs and Literature Synthesis — a multi-level handbook.*

### Read it online → **[https://phindagijimana.github.io/AutoLabs/](https://phindagijimana.github.io/AutoLabs/)**

[![Site](https://img.shields.io/badge/site-AutoLabs-6f42c1?logo=readthedocs&logoColor=white)](https://phindagijimana.github.io/AutoLabs/)
[![GitHub](https://img.shields.io/badge/github-phindagijimana%2FAutoLabs-181717?logo=github)](https://github.com/phindagijimana/AutoLabs)
[![Deploy docs](https://github.com/phindagijimana/AutoLabs/actions/workflows/docs.yml/badge.svg)](https://github.com/phindagijimana/AutoLabs/actions/workflows/docs.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

---

A reference for building, operating, and reasoning about systems that use AI (and sometimes robots) to **do science**: design experiments, run them, analyse them, and stitch together what the literature already knows.

This handbook is a sibling to [NeuroStack](https://phindagijimana.github.io/neuro_stack/). The structure, tone, and editorial choices follow the same playbook — layered chapters, honest warnings, neuroimaging examples (hippocampal sclerosis recurs as a worked case) — but the subject is different. NeuroStack is about *reading the brain*. AutoLabs is about *accelerating science itself*.

## Audience

Four reader levels share the same content surface. Pick the entry point that matches where you are.

| Level | Who it's for | What you'll learn |
| --- | --- | --- |
| **[Beginner](docs/beginner/index.md)** | Students, clinicians, science-curious readers. No coding required. | What autonomous labs are, what literature synthesis is, why both matter, the vocabulary. |
| **[Intermediate](docs/intermediate/index.md)** | Master's students, junior researchers, engineers entering the space. Some Python helps. | How closed-loop labs are wired, how biomedical NLP and knowledge graphs work, what tools exist. |
| **[PhD / Researcher](docs/phd/index.md)** | Doctoral students, research scientists, methods-focused engineers. | The methods underneath: Bayesian optimisation, RL, LLM scientific reasoning, relation extraction, RAG, hypothesis generation. |
| **[Senior Research Engineer](docs/engineer/index.md)** | Staff/principal engineers building or operating these systems. | Architecture, orchestration, observability, reproducibility, safety, governance, the parts that decide whether the system survives contact with reality. |

Cross-reading is encouraged. A senior engineer benefits from the beginner glossary; a master's student benefits from the senior chapter on observability before their first cluster crash.

## What's inside

```
AutoLabs/
├── README.md            ← you are here
├── mkdocs.yml           ← site config (MkDocs Material)
└── docs/
    ├── index.md
    ├── beginner/        ← plain-language overview, vocabulary
    ├── intermediate/    ← components, tools, worked examples
    ├── phd/             ← methods, math, current research
    └── engineer/        ← production systems, ops, governance
```

## Two parts in every level

Every level covers the same two halves:

- **Part A: Autonomous labs** — closed-loop experimentation, AI planners, lab robotics, feedback loops.
- **Part B: Literature synthesis** — biomedical NLP, knowledge graphs, hypothesis generation, systematic review support.

They are separated because the technical stacks differ, but they belong together: a lab that does not read the literature wastes experiments; a literature pipeline that cannot trigger an experiment is a fancy search engine.

## Reading the handbook online

The live site is at **[https://phindagijimana.github.io/AutoLabs/](https://phindagijimana.github.io/AutoLabs/)**. It is rebuilt and redeployed by the [`Deploy AutoLabs docs`](.github/workflows/docs.yml) workflow on every push to `main`.

To preview locally:

```bash
pip install mkdocs-material
mkdocs serve            # serves http://127.0.0.1:8000
mkdocs build --strict   # one-shot build into site/
```

## Status

Initial draft. The four-level structure is in place; each level has both Part A and Part B chapters. Content will be deepened — particularly the PhD case studies and the engineer-level runbook material — as the corresponding research and platform work matures.

## Related

- [NeuroStack](https://phindagijimana.github.io/neuro_stack/) — neuroimaging fundamentals, data engineering, AI/ML, computing.
- The companion DBI pipeline and `neuro_handbook` Python package, both referenced from this handbook where their patterns transfer.

## License

MIT (consistent with NeuroStack).
