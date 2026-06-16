# Portfolio & roadmap

> *A six-milestone path from "interesting demo" to "platform other teams build on".*

A senior research engineer's promotion artifact is not "I trained a model"; it is "the platform I built supports five teams and three publications". This chapter is a milestone roadmap — both for organising the work and for portfolio framing.

It echoes the [NeuroStack portfolio roadmap](https://phindagijimana.github.io/neuro_stack/data-engineering/portfolio-roadmap/) deliberately: same shape, different domain.

## Milestone 1 — Working notebook (~1 week)

Goal: a single notebook that runs the closed loop end-to-end on a toy problem.

- One Python notebook.
- A toy `f(x)` (analytic) standing in for the lab.
- BoTorch BO loop.
- 10 iterations, plots of acquisition and best-so-far.

What it teaches: the loop's anatomy, the surrogate's behaviour, where uncertainty matters.

Anti-goal: do not productionise. The point is end-to-end shape, not robustness.

Output for portfolio: a public notebook with a clear write-up.

## Milestone 2 — Decoupled services (~3–4 weeks)

Goal: split the notebook into a planner service, a "worker" service (still toy `f`), and a history service. Talk over an event bus.

- Planner: FastAPI service exposing `propose`, `update`.
- Worker: subscribes to a queue, "runs" the toy `f`, posts results.
- History: Postgres with append-only schema; result records as in [reproducibility](reproducibility.md).
- A simple CLI (Typer) for proposing goals and inspecting history.

What it teaches: service contracts, idempotency, queues, replay.

Anti-goal: do not add hardware yet.

Output: a small docker-compose stack with a README showing the loop running, plus a short architecture diagram. Add this to the portfolio.

## Milestone 3 — First real plumbing into the lab (~6–10 weeks)

Goal: replace the toy `f` with a real (or simulated) instrument.

Pick one:

- An Opentrons OT-2 doing a dilution series and a plate read.
- A simulated DWI pipeline using QSIPrep on a single subject (the [NeuroStack DWI case study](https://phindagijimana.github.io/neuro_stack/data-engineering/dwi-case-study/) shape).
- A simulated lab using a digital twin from a vendor or open-source kit (e.g., PyLabRobot's simulator).

Add: a QC service, a calibration service, structured event logging, basic Grafana dashboards.

What it teaches: real failure modes, the cost of every silent assumption, what observability earns its keep on.

Anti-goal: don't yet integrate the literature plane.

Output: an incident retrospective (yes, you will have one). Drop into portfolio with anonymised numbers — "I caught calibration drift before it corrupted N runs".

## Milestone 4 — Literature plane minimum-viable (~6–8 weeks)

Goal: a working literature pipeline that produces a useful KG slice.

- PubMed ingestion (E-utilities; rate-limited).
- NER with scispacy + a fine-tuned PubMedBERT NER head on a small in-domain set.
- Normalisation against UMLS or Mondo / HGNC / DrugBank.
- Relation extraction with a fine-tuned classifier on BC5CDR or DrugProt.
- Edge aggregation with calibrated confidence.
- BioCypher-built KG in Neo4j.
- Snapshot release tagging.

What it teaches: NLP engineering rigour, ontology grounding, aggregate confidence, KG ops.

Output: a published KG release with a documented release-notes diff. Notebook examples querying the graph.

## Milestone 5 — Integration: literature shapes the loop (~4–6 weeks)

Goal: the planner's search space and prior are conditioned on the KG.

- Hypothesis generator surfaces candidate experiments from the KG.
- Goal-spec validator checks proposals against KG-derived feasibility hints.
- RAG service serves citations alongside the planner rationale.
- The systematic-review pipeline produces a PRISMA flow that's automatically updated each release.

What it teaches: the platform now has both halves; the interactions are what generate the wins.

Output: a paper figure showing one experiment whose choice was traceable end-to-end from a KG edge to a wet-lab measurement.

## Milestone 6 — Other teams use it (~3–6 months)

Goal: a second team uses the platform without your daily involvement.

You will know you have arrived here when:

- A new goal can be onboarded in under a day.
- The second team writes their own runbooks.
- Incidents are handled by the second team (with you in retro).
- The roadmap is shaped by their requests, not yours.

What it teaches: documentation, support, prioritisation under multi-tenant pressure.

Output for portfolio: testimonials, internal blog post, talk at internal demo day. A successful platform supports more than its author.

## What the senior engineer does, month by month

A rough rhythm once the platform exists:

| Cadence | Activity |
| --- | --- |
| Daily | On-call rotation; triage alerts; review one extracted KG edge audit sample. |
| Weekly | Goal-onboarding sync; results review with PIs; pay down one observability gap. |
| Monthly | Calibration audit; KG release; RAG faithfulness eval. |
| Quarterly | Architecture review; cost review; safety tabletop. |
| Annually | Reproducibility drill; restore drill; red-team of hypothesis generator. |

This rhythm is what keeps a research platform from becoming research debt.

## Portfolio artifacts that work

- **End-to-end demo videos.** A 3-minute screen recording of a goal-to-result run with commentary.
- **Architecture decision records (ADRs).** Why you picked Prefect over Airflow, Neo4j over RDF, etc.
- **Incident retros.** Specific, anonymised; what you learned.
- **Cost / performance reports.** A graph of "cost per experiment" or "median loop latency" trending down.
- **Talks.** A 20-minute conference talk on a specific failure mode you solved.

What does *not* land in portfolios: code-only repos with no narrative; novel methods with no users; dashboards no one reads.

## Anti-patterns

- **Architecture astronauting milestone 1.** A microservices mesh before you have a result is wasted effort.
- **Literature plane before the lab plane works.** Easier to demo, but the integration in milestone 5 is the leverage.
- **Skipping milestone 6.** Solo platforms decay. Aim for users.
- **Choosing tools by trend.** Pick what you can operate; trends rotate.
- **Avoiding incident retros.** They are the most useful artifact you will produce.

## What "senior research engineer" looks like at the end

You ship platforms that other people build their careers on. You are responsible for safety and reproducibility decisions, not just for code. You can write a postmortem that a regulator finds reassuring. You can explain a planner's behaviour to a PI in two sentences. You can read a paper, identify the closest existing platform pattern, and propose an extension that costs your team a week, not a quarter.

That is the destination. The milestones above are how you get there.

## Where to next

- Re-read whichever earlier chapter you have least confidence in.
- For methods depth, [PhD chapters](../phd/index.md).
- For the broader handbook ecosystem, [NeuroStack](https://phindagijimana.github.io/neuro_stack/).
