# Orchestration

> *Schedulers, queues, idempotency, batching, retries — what turns a Python script into a platform.*

The planner picks experiments. The scheduler turns picks into runs. A research engineer underweights the scheduler at their peril: it is the component that fails most often, fails most visibly, and is hardest to add later.

## Workloads in this platform

Three flavours coexist:

| Workload | Cadence | Latency tolerance |
| --- | --- | --- |
| **Lab experiments** | Seconds–weeks per cycle. | Hours of delay are bad; days of delay are fatal. |
| **Analyses** | Minutes–hours per job; bursty. | Best-effort but mustn't block the loop. |
| **Literature pipelines** | Daily or weekly batch. | Backlog-tolerant. |

A single orchestrator can drive all three, but they have different SLOs and need separate queues, separate workers, and separate retries.

## The scheduler's contract

What every scheduler in this platform owes:

1. **Durable enqueue.** A spec accepted by the scheduler is recoverable if a node dies.
2. **Idempotent execution.** A retried run produces the same result (or a clear "already done").
3. **At-least-once delivery.** Better duplicates than losses; the worker dedupes.
4. **Observable.** Queue depth, lag, success/failure rate are first-class metrics.
5. **Cancellable.** A scientist can revoke a queued or in-flight run cleanly.
6. **Prioritisable.** Critical runs preempt long ones; long ones don't starve.
7. **Calendar-aware.** Maintenance windows, calibration windows, day/night windows.

## Tool choices

| Tool | Notes |
| --- | --- |
| **Airflow** | Mature, batch-oriented. Reasonable for literature pipelines; clumsy for tight planner-robot loops. |
| **Prefect 2/3** | Pythonic, hybrid execution; good for mixed workloads. |
| **Dagster** | Asset-oriented; strong for data plane and literature. |
| **Argo Workflows** | Kubernetes-native; sturdy on bursty analyses. |
| **Temporal** | Durable execution for long-running orchestrations; great for the planner→robot path. |
| **Celery / RQ** | Simple distributed task queues; adequate for many labs. |
| **HPC schedulers** (SLURM, PBS) | Necessary if your analyses live on a cluster; integrates as a worker, not a substitute. |
| **Vendor lab schedulers** (Cellario, Biosero) | Sit inside the lab; the engineering scheduler integrates with them via an adapter. |

For most platforms: Prefect or Temporal for the planner-robot path; Dagster or Airflow for the literature plane. The HPC scheduler is an executor target, not a replacement.

## Queues and priority

A practical queue layout:

| Queue | Workers | Priority |
| --- | --- | --- |
| `lab.run.critical` | Robot drivers. | Highest. |
| `lab.run.standard` | Robot drivers. | Default. |
| `lab.calibration` | Robot drivers. | High; preempts standard. |
| `analysis.fast` | CPU workers. | Default. |
| `analysis.gpu` | GPU workers. | Default; long-running. |
| `lit.ingest` | Cloud workers. | Low. |
| `lit.refresh` | Cloud workers. | Background. |

Workers register against specific queues; the scheduler routes accordingly. Don't share a queue across drastically different SLAs — head-of-line blocking will haunt you.

## Idempotency

Reality: networks blink, vendor SDKs throw, workers OOM. Without idempotency, retries duplicate experiments — which can mean wasted reagents, double-counted measurements, or contaminated wells.

Three layers:

- **Spec idempotency key.** The planner emits a spec with a stable hash. The scheduler refuses to enqueue if the key is already in flight or recently completed.
- **Worker-side dedupe.** Even with a duplicate enqueue, the worker checks "did I already do this?" before acting.
- **Physical-action idempotency** where possible. Some lab actions are physically irreversible; the worker must distinguish "already started" from "not started" — pre-flight checks and per-step idempotency tokens help.

```python
def submit(spec):
    key = hash_spec(spec)
    if jobs.recently_completed(key, within="7d"):
        return {"job_id": jobs.lookup(key),
                "status": "deduplicated"}
    if jobs.in_flight(key):
        return {"job_id": jobs.lookup(key),
                "status": "already_running"}
    return jobs.enqueue(spec, idempotency_key=key)
```

## Retries

Retries are dangerous in a wet lab. The rule of thumb: *retry only what you can prove is safe to retry.*

| Failure | Retry? |
| --- | --- |
| Network blip before any physical action | Yes, automatic. |
| Robot heartbeat lost mid-protocol | No, page a human. |
| Liquid handler reports protocol complete but no measurement returned | Manual review; do not auto-retry the measurement step. |
| Analysis OOM | Yes, with larger memory; bounded retries. |
| RAG call timeout | Yes, with backoff. |
| KG-ingest transaction conflict | Yes, with exponential backoff. |
| LLM 5xx | Yes, with budget cap. |

Implement: exponential backoff with jitter, capped retry count, per-failure-class policy.

## Batching

For literature pipelines, batching saves order-of-magnitude cost and time. Group hundreds of paper extractions, hundreds of KG ingests, hundreds of RAG queries.

For lab pipelines, batching matters in two places:

- **Planner emits a batch** — q-acquisition across q experiments at once.
- **Scheduler enforces resource batching** — e.g., one plate-reader read per plate, not per well.

Batching also matters for cost-aware LLM calls — batch where possible; use OpenAI / Anthropic batch APIs for non-urgent jobs.

## Long-running workflows

Some workflows last days (incubation, tissue prep) or weeks (animal acclimatisation). Durable execution engines (Temporal, AWS Step Functions, Argo) preserve workflow state across worker restarts. The planner's "wait for result" is a sleep in workflow code, not a process holding open a socket.

```python
@workflow.defn
class ExperimentWorkflow:
    @workflow.run
    async def run(self, spec):
        run_id = await workflow.execute_activity(submit_to_robot, spec, ...)
        raw    = await workflow.execute_activity(wait_for_result, run_id,
                                                 start_to_close_timeout=timedelta(days=3))
        clean  = await workflow.execute_activity(qc_and_normalise, raw, spec)
        result = await workflow.execute_activity(analyze, clean, spec)
        await workflow.execute_activity(report_to_planner, result, spec)
        return result
```

The workflow runs across worker restarts, scheduler upgrades, and weekends.

## Quotas, throttling, budgets

A platform without quotas eventually has runaway behaviour.

- **Per-goal quota.** Maximum experiments per week per goal.
- **Per-user / per-team quota.** Stops a runaway agent from monopolising the lab.
- **LLM token budget.** Per goal, per day; hard stop with notification.
- **Cloud spend budget.** Per project; alerts at 50%, 80%, 100%.
- **Hardware time fairness.** Round-robin among teams when multiple goals queue.

The scheduler is the natural enforcement point.

## Coordination with the lab orchestrator

If a vendor scheduler (Cellario, Biosero) sits inside the lab, the engineering scheduler doesn't try to replace it — it integrates as a client:

```mermaid
flowchart LR
  planner --> engsched[Engineering scheduler]
  engsched -->|protocol| labsched[Lab orchestrator]
  labsched -->|commands| robots
  robots -->|measurements| labsched
  labsched -->|raw + audit| engsched
  engsched -->|result event| planner
```

The engineering scheduler owns the queue, prioritisation, and idempotency. The lab orchestrator owns the on-deck choreography.

## Observability hooks

Every scheduler operation emits structured events:

- `spec.submitted`, `spec.deduplicated`, `spec.cancelled`.
- `job.started`, `job.completed`, `job.failed`, `job.timed_out`.
- `queue.depth`, `queue.lag`.
- `retry.attempted`, `retry.exhausted`.

These feed [observability](observability.md). Without them, you find out about a stalled queue from a confused scientist.

## Disaster recovery

- The queue is durable and replicated.
- The audit log of submissions/completions is independently backed up.
- Recovery procedure documented as a runbook.
- Quarterly recovery drills.

A scheduler that has never been restored from backup will not survive its first real outage.

## Honest warnings

- **Auto-retry will burn reagents.** Tighten retry policy *before* the first robot incident, not after.
- **Queues hide behaviour.** A consistent 30-minute backlog is invisible until somebody asks "why did this take so long?".
- **Calendar awareness is hard to retrofit.** Build it in from day one.
- **Vendor lab schedulers fight you.** Their assumptions differ from yours; the adapter is more work than expected.

## Where to next

- [Observability](observability.md) — what you measure across all this.
- [Reproducibility](reproducibility.md) — pinning code, calibration, prompts across scheduler restarts.
- [Architecture](architecture.md) — where the scheduler fits.
