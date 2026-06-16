# Senior Research Engineer

> *Designing, operating, and being responsible for autonomous-lab and literature-synthesis systems in production.*

By this level, you've built a planner. You've built a literature pipeline. They work in a notebook. Now somebody else has to run them next Tuesday while you're on holiday — and the data has to be defensible, and a regulator might ask questions. This level is about that.

## What changes at this level

The technical pieces don't go away; you keep all the BO, RL, NLP, RAG. But the centre of gravity shifts to:

| Concern | Why |
| --- | --- |
| **Architecture** | The system has many components; they must compose cleanly and survive change. |
| **Orchestration** | Real labs run for weeks; literature pipelines run forever. The scheduler is load-bearing. |
| **Observability** | When the loop stalls or drifts, you must see it within minutes, not days. |
| **Reproducibility** | Every result must be re-derivable. Reviewers and regulators ask. |
| **Safety & governance** | Biosafety, chemical safety, dual-use, patient privacy, IRB obligations. |
| **People & roadmap** | The system isn't a notebook; it's a platform that funded teams build on. |

## The chapters

- **[Architecture](architecture.md)** — services, contracts, data planes, what runs where.
- **[Orchestration](orchestration.md)** — schedulers, queues, idempotency, batching, retries.
- **[Observability](observability.md)** — metrics, logs, traces, drift detection, runbook design.
- **[Reproducibility](reproducibility.md)** — versioning of code, data, calibrations, prompts, KGs, reagents.
- **[Safety & governance](safety-governance.md)** — biosafety, dual-use, IRB, HIPAA / GDPR, audit, model risk.
- **[Portfolio & roadmap](portfolio-roadmap.md)** — a six-milestone path from "interesting demo" to "platform other teams build on".

## What this level assumes

You can read the [PhD](../phd/index.md) chapters and the [NeuroStack data-engineering](https://phindagijimana.github.io/neuro_stack/data-engineering/) chapters. You've seen a production incident. You've explained a model decision to a non-engineer.

## What this level intentionally skips

- New mathematical methods — the PhD chapters cover those.
- Specific vendor recommendations — they age out; principles age slower.
- Generic SRE content already well-covered elsewhere — pointers, not duplication.
