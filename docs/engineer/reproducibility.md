# Reproducibility

> *Versioning the code, the data, the calibrations, the prompts, the KGs, and the reagents.*

A reviewer asks: "are you sure?" A regulator asks: "show me." A future you asks: "what did we do?" Reproducibility is the answer to all three. It is not a polish task; it is part of the platform.

## What "reproducible" means here

For an autonomous-lab platform, three reproducibility levels matter:

| Level | Promise |
| --- | --- |
| **Replay** | Given an audit-log range, the analyses can be re-run and produce the same numbers. |
| **Reanalyse** | Given raw measurements and pinned tool versions, the silver/gold layers regenerate identically. |
| **Re-acquire** | Given a logged protocol, a different team in a different lab can run the same experiment. |

All three require versioning. Different things at each level.

## The version inventory

Every result record carries pointers to:

| What | How to version | Why |
| --- | --- | --- |
| Source code | Git SHA + dirty-flag check. | Models, prompts, glue code change. |
| Container images | Content-addressed digest (`sha256:...`). | Image tags are mutable; digests aren't. |
| Conda / pip lockfile | Hash of resolved environment. | Indirect deps move silently otherwise. |
| Hardware firmware | Per-instrument firmware version. | Robots updated mid-experiment shift behaviour. |
| Calibration set | Calibration release ID. | Drives raw → silver normalisation. |
| Reagent lot | Vendor lot number + expiry. | Biology depends on it; recalls happen. |
| Cell-line / tissue ID | Authenticated cell line ID + passage number. | Mis-identification is endemic. |
| Plate-map version | Hash of the plate-map file. | Wells move; conditions shift. |
| Analysis pipeline | Container digest + parameter file hash. | Re-derivation. |
| QC config | Versioned QC rules. | What was a pass yesterday may be a flag today. |
| Planner model | Model file hash + config hash. | The "why" of the choice. |
| Prompt template | Hash + model version. | Prompts are code. |
| KG release | Release tag (immutable). | What the planner read. |
| RAG index | Index snapshot ID. | What the planner could retrieve. |
| Schedule version | Scheduler config snapshot. | When and where it ran. |

The result record is a small object; it points to all of these. The audit log is what makes the pointers resolvable later.

## The result record, in full

```json
{
  "experiment_id":        "exp-2026-06-16-0042",
  "metric":               "viability_normalized",
  "value":                0.71,
  "uncertainty":          0.04,
  "qc_status":            "pass",
  "spec_hash":            "sha256:8a3e…",
  "code_sha":             "git:9d50fe7",
  "image_digest":         "sha256:f1c2…",
  "env_lock_hash":        "sha256:0aa9…",
  "robot_firmware":       {"liquid_handler":"3.4.2","reader":"1.7.0"},
  "calibration_version":  "calib-2026-06-13",
  "reagent_lots":         {"CTG":"LOT-2026A1","cells":"SH-SY5Y-p12-batch-7"},
  "plate_map_hash":       "sha256:c4ce…",
  "analysis_version":     "v3.2.1",
  "planner_version":      "bo-v3.2.0",
  "planner_config_hash":  "sha256:5d2a…",
  "kg_release":           "KG-2026-06-10",
  "rag_index":            "rag-2026-06-12T03:00Z",
  "rationale_id":         "rat-7f0c",
  "trace_id":             "0f1d…",
  "ts":                   "2026-06-16T17:04:12Z"
}
```

A reviewer who can dereference any of these strings can answer almost every "why" question.

## Source-of-truth artifacts

The platform commits to immutability for:

- Bronze (raw) data. Object storage with object-lock; tape backups for long-term.
- Audit log. Append-only; replicated across regions.
- KG releases. Tagged; deletions disallowed without a special process.
- Planner model snapshots. Hash-named files; never overwritten.
- Prompt registry. Git-tracked; reviews required.

Mutable working areas (silver, gold) are rebuildable from immutable ones plus versioned code.

## Replay

Replay is "run the same analysis on the same raw data with the same code; verify the same numbers come out".

```bash
autolabs replay \
  --experiment exp-2026-06-16-0042 \
  --to-layer gold \
  --strict
```

`--strict` fails if the rebuilt result differs in any digit. `--tolerance 1e-9` is the polite version.

Replay catches:

- Non-determinism in analysis (CUDA, threading, random seeds).
- Hidden dependencies on system state (time, locale).
- Forgotten config that wasn't captured.

A platform that can't replay its own results from a month ago is not yet a platform.

## Reanalyse

Reanalysis is "given raw + versions, regenerate silver/gold". This needs:

- All container images still pullable.
- All software lockfiles resolvable.
- Calibration files still retrievable.

Container registries garbage-collect. Lockfile URLs rot. The platform owns durable mirrors of every artifact it depends on. Plan for a 5-year horizon at minimum.

## Re-acquire

Re-acquisition is the highest bar: another lab can reproduce the experiment from scratch.

It needs:

- A human-readable protocol with vendor-neutral details.
- A list of reagents with vendor and catalog numbers.
- A list of equipment with model and software version.
- A plate-map and timing diagram.
- A versioned protocol script (Opentrons API, SBOL, AnIML).

Most autonomous-lab papers don't publish enough for this. The serious platforms include it as artifact `protocol.tar.gz` per experiment.

## Randomness and determinism

Sources of non-determinism that ruin reproducibility:

- Unseeded random number generators.
- Float-order dependence in parallel reductions (Spark, multi-GPU).
- CUDA non-deterministic ops (set `torch.use_deterministic_algorithms(True)`).
- LLM sampling (set temperature=0 and seed; pin model version; accept that some non-determinism remains in vendor APIs).
- Network ordering for event-driven analyses.

Where impossible to make deterministic, capture the *result* as the source of truth: store the LLM output verbatim with its model version and request ID; replay by reading the stored output.

## Provenance graph

Every artifact in the platform points to its inputs. The aggregated graph — "this gold metric came from this silver run, which came from this bronze acquisition, with this calibration, with this analysis version" — is a *provenance graph*.

Tools: W3C PROV, ML Metadata (MLMD), MLflow, OpenLineage, plus your audit log. For research labs, OpenLineage + a Postgres backing store is a credible starter setup.

## KG and RAG reproducibility

The literature plane has its own reproducibility:

- KG releases are tagged and immutable.
- The pipeline that produced a KG release is captured (extractor versions, source corpus snapshot).
- RAG index snapshots are referenced by ID; old indices retained for some retention period.
- LLM prompts and responses (for batch jobs) are stored verbatim alongside the model name and version.

If a downstream paper relies on a hypothesis surfaced by RAG, the platform can re-derive that hypothesis from the stored snapshots, even if the corpus has moved on.

## Storage and retention

A practical retention policy:

| Artifact | Retention | Notes |
| --- | --- | --- |
| Bronze (raw) | 10+ years. | Often regulatory minimum. |
| Audit log | 10+ years. | Object-lock; multi-region. |
| Silver | 3 years; rebuildable from bronze. | Cheaper storage class. |
| Gold | 3 years; rebuildable. | Cheaper storage class. |
| KG releases | All retained. | Small; high re-use. |
| RAG indices | 1 year; reproducible from KG + corpus. | Storage-expensive otherwise. |
| LLM batch outputs | 5 years; high-value provenance. | Compressed. |
| Container images | 5 years; mirror your own registry. | Don't trust vendor registries to retain. |

Tie retention policies to the data governance program.

## Drills

A reproducibility drill, twice a year:

1. Pick a result from N months ago.
2. Try to replay it from the audit log.
3. Note every step that breaks.
4. File issues; pay them down.

If you cannot do this drill today, you cannot do it under pressure.

## Honest warnings

- **Mutable tags everywhere.** Vendor model IDs change behind the same name; pin model versions.
- **Untrusted system state.** Clocks, locales, file orderings — all sneak in. Test in clean containers.
- **LLM non-determinism.** Accept it; cache outputs; treat the cache as the source of truth.
- **Loss of vendor SDKs.** A vendor disappears; your pipeline breaks. Mirror what you can; document workarounds for what you can't.

## Where to next

- [Architecture](architecture.md) — where the versioned artifacts live.
- [Observability](observability.md) — what to watch to catch drift.
- [Safety & governance](safety-governance.md) — what regulators expect.
