# Observability

> *Metrics, logs, traces, drift detection, and the runbooks that make the system survivable.*

You cannot operate what you cannot see. Autonomous-lab platforms fail in slow, quiet ways — drifting calibrations, stuck planners, hallucinating RAG, growing KG noise. Observability is what turns a system from "fine until it isn't" into "fine, with caught regressions".

## What to measure

Four layers, four kinds of signal.

### Lab signals

- Per-instrument heartbeat, last successful protocol, error rate.
- Per-protocol success rate, time-to-complete distribution.
- Reagent inventory levels and shelf-life.
- Calibration delta against the previous run.
- Plate-position error rate.
- Tip pickup failure rate.

### Planner signals

- Acquisition value distribution — collapse signals a stuck planner.
- Surrogate calibration — predicted-interval coverage on new observations.
- Diversity of last-N proposals — repetition is a smell.
- Latency from `propose` to `update`.
- Rationale length / complexity (if LLM-driven).

### Literature signals

- Ingestion throughput.
- Per-source NER / RE F1 against a rolling reference set.
- KG edge growth, dropped-edge rate, conflict rate.
- RAG retrieval recall and faithfulness on a held-out eval set.
- LLM token usage; cost per query; cache hit rate.

### Platform signals

- Queue depth, lag, error rate (per queue).
- Worker CPU / GPU / memory.
- API latency P50/P95/P99 per endpoint.
- Audit-log ingestion rate.
- Storage utilisation per layer (bronze/silver/gold).

Each of these has a target SLO and an alert above which someone is paged.

## The three pillars, applied here

| Pillar | What it gives | Tools |
| --- | --- | --- |
| **Metrics** | Trends, SLO compliance, alarms. | Prometheus + Grafana; Datadog; managed Honeycomb metrics. |
| **Logs** | Per-request debugging. | Loki; Elastic; Datadog logs. |
| **Traces** | Cross-service request flow; bottlenecks. | OpenTelemetry → Tempo / Jaeger / Honeycomb. |

For an autonomous lab, *traces* matter more than they would in a generic web service: the trace for "propose → submit → run → analyse → update" tells you where the day went.

## Structured events

Every component emits structured JSON events to one bus (e.g., Kafka, NATS, or a managed event topic). Examples:

```json
{"event":"planner.propose",
 "goal_id":"g-2026-06-16-001",
 "spec_id":"s-7c3a","planner_ver":"bo-v3.2.0",
 "acq_value":0.42,
 "ts":"2026-06-16T15:22:01Z"}

{"event":"worker.protocol_step",
 "spec_id":"s-7c3a","step":"compound_addition",
 "status":"ok","ts":"..."}

{"event":"qc.fail",
 "spec_id":"s-7c3a","reason":"positive_control_below_threshold",
 "ts":"..."}

{"event":"rag.query",
 "query_id":"q-1","cited_passages":["P3","P7","P12"],
 "faithfulness_score":0.91,"ts":"..."}
```

Structured events are the substrate for metrics, dashboards, and alerts. They are also the audit log (see [reproducibility](reproducibility.md)).

## Drift detection

The single most important observability investment for AutoLabs is *drift detection*.

### Calibration drift

Compare per-instrument calibration over rolling windows. Alert when control-well measurements move more than $k\sigma$ from the historical distribution.

```python
def calibration_alert(latest, baseline):
    mu, sd = baseline.mean(), baseline.std()
    if abs(latest - mu) > 3 * sd:
        return "drift_alert"
```

Pair with weekly phantom / sentinel runs — known inputs whose outputs should not move.

### Surrogate drift

Track the surrogate's predictive intervals against new observations. If a 95% interval captures only 70%, the surrogate is overconfident; planner decisions are unsafe.

### Extractor drift

The NLP extractors (NER, RE) are evaluated on a frozen rolling reference set, monthly. A regression above a small threshold blocks the KG release.

### RAG faithfulness drift

Run a small evaluation set through the RAG pipeline daily. Track citation-supported-claim rate. Below threshold → page.

### Concept drift in the lab

New reagent lot, new cell batch, new operator. These produce silent regime changes. Periodic sentinel experiments — known conditions, known expected outputs — catch them before the planner does.

## Dashboards

A working autonomous-lab platform has at least four dashboards:

1. **Loop health.** Last successful loop, queue depths, latency, recent failures.
2. **Lab inventory and calibration.** Reagent stocks, last calibration per instrument, drift status.
3. **Planner cockpit.** Goal-by-goal acquisition values, predicted-vs-actual scatter, batch diversity.
4. **Literature pipeline.** Ingestion volume, extractor scores, KG release diff, RAG faithfulness.

Plus on-call pages summarising what's red.

## Alerts that earn their keep

Avoid alert fatigue. The alerts that survive a year on call:

- Loop has not produced a result in $T$ hours (per goal).
- Queue depth above threshold for $T$ minutes.
- Calibration drift threshold breached.
- Surrogate calibration falls below threshold.
- KG conflict rate above threshold.
- RAG faithfulness below threshold.
- LLM spend trajectory exceeds budget.
- Audit-log write failure (catastrophic).

Each alert has an associated runbook.

## Runbooks

A runbook is a markdown file in the repo for each common incident. Template:

```markdown
# Title: <Symptom>
## Signal
What the alert looks like.

## Triage
Five-minute checks to confirm the symptom.

## Likely causes
Top 3, with how to differentiate.

## Mitigations
Specific actions. Each action labelled `[reversible]` or `[needs approval]`.

## Resolution
What "fixed" looks like.

## Post-mortem
Link to incident review.
```

Runbooks are kept under version control next to the system code. They are reviewed in incident retros.

## A worked alert: "loop has stalled"

Signal: no `analysis.completed` event for goal G in 6 hours.

Triage:

1. Is the planner emitting `spec.submitted`? If no, planner side; if yes, downstream.
2. Are workers picking up specs? Check queue depth.
3. Is the worker reporting `protocol_step` events? If no, robot side.
4. Is the analysis service receiving raw measurements? If no, transport problem.
5. Is the analysis service running but slow? Backlog.

Mitigations:

- Restart worker pool [reversible].
- Drain stuck spec [needs approval].
- Pause goal G [reversible].
- Page lab operations if root cause is hardware.

This runbook turns an unbounded incident into a 30-minute mitigation.

## SLOs

A small set of SLOs keeps engineering honest:

- **Loop completion SLO.** 95% of submitted specs complete the loop within $T$ hours.
- **Calibration freshness SLO.** All instruments calibrated within the past $D$ days.
- **KG release SLO.** No more than $T$ days of literature backlog at release time.
- **RAG availability SLO.** 99.5% of queries in $L$ seconds.
- **Audit-log durability SLO.** Zero loss; replication lag below threshold.

SLOs feed error budgets; error budgets feed prioritisation.

## Logging hygiene

- Structured JSON, with a stable schema.
- Trace ID on every event so cross-service traces stitch.
- No PHI / PII in logs. Apply scrubbing at the edge.
- Sampled at high volume; un-sampled for errors.

## Cost observability

LLM calls and GPU minutes are the hidden costs. Track per-goal:

- LLM tokens (input/output) by model.
- GPU minutes by job type.
- Storage bytes accumulated this month.

Surface as a dashboard at the team level. Cost regressions are a smell that a model swap or prompt change introduced waste.

## Honest warnings

- **You will under-instrument.** Reserve a regular slice of platform-engineering time to add instruments to whatever you most recently debugged.
- **Logs without traces are noise.** Always emit a trace ID.
- **Dashboards rot.** Audit them quarterly; delete dead panels.
- **Alerts without runbooks are alarms-only.** Refuse to merge a new alert that lacks a runbook entry.

## Where to next

- [Reproducibility](reproducibility.md) — versioning so that a captured event is interpretable later.
- [Safety & governance](safety-governance.md) — what observability owes regulators.
- [Orchestration](orchestration.md) — the events that the scheduler emits.
