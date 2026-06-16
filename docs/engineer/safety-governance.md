# Safety & governance

> *Biosafety, chemical safety, dual-use risk, patient privacy, IRB obligations, model risk.*

The technical capacity to plan and run experiments at AI speed has outrun most institutional processes. The senior research engineer is responsible for keeping the platform inside the rails, often without those rails being written down. This chapter is about the responsibilities, not just the controls.

## What can go wrong

The honest list, in rough decreasing frequency:

1. Wasted reagents or animal subjects from a planner that didn't know its constraints.
2. A wet-lab safety incident from automation outpacing operator awareness.
3. A patient-data leak from a literature pipeline that ingested clinical notes.
4. An IRB violation from a study modification not re-approved.
5. A regulatory finding from missing or incomplete audit trail.
6. A model-bias or equity harm from a planner that over-sampled accessible subjects.
7. An unintended dual-use risk from a hypothesis generator suggesting harmful syntheses.
8. A reputational incident from a hallucinated systematic-review finding being relied on.

Each has a different mitigation. None is hypothetical; each has happened to *somebody*.

## Biosafety

Standard biosafety (BSL-1 → BSL-4) levels apply regardless of automation. Additional concerns when robots are in the loop:

- **Accidental escalation.** A planner asks for higher concentrations of a regulated pathogen than the lab's biosafety level permits. A whitelist enforcement layer rejects.
- **Containment failures.** Robot-induced aerosol from rapid pipetting. Engineering controls and operator training.
- **Operator awareness.** Humans in the lab while robots are running need to know the program. Lights, alarms, and physical-interlock interfaces.
- **Decontamination cycles.** Automated decon between experiments; verifiable, logged.

The platform encodes the biosafety policy as a *machine-checkable constraint* on every spec the planner proposes.

```python
def biosafety_filter(spec, policy):
    if spec.pathogen.bsl > policy.max_bsl:
        return Rejection("BSL ceiling exceeded")
    if any(c.qty > policy.qty_ceiling[c.id]
           for c in spec.consumables):
        return Rejection("quantity ceiling exceeded")
    return Allow()
```

Two-key approval (planner + human) for any action above a defined risk threshold.

## Chemical safety

Similar shape:

- Reagent compatibility checks (peroxides + reducers, etc.).
- Volume / concentration ceilings.
- Waste-stream segregation rules encoded.
- SDS lookups automated; expired-SDS-blocking.

A chemistry-side autonomous lab without a *constraint-aware planner* is a regulatory exposure.

## Dual-use research of concern (DURC)

Some lab capabilities and some hypothesis suggestions touch dual-use territory: gain-of-function pathogen work, broadly applicable toxin design, restricted chemistry. Institutional DURC review precedes any platform that could enable such work.

In practice:

- The hypothesis generator filters DURC-adjacent categories at the *prompt* layer; if a candidate hypothesis falls in restricted categories, it surfaces to a designated review group rather than the user.
- The planner's allowed search space excludes DURC categories by configuration; changes to the configuration are reviewed.
- The audit log preserves every flagged and rejected request.

For US researchers the relevant policy is NIH's DURC and Potential Pandemic Pathogen Care and Oversight (P3CO) policies; the lab's IBC owns the case-by-case decisions.

## Patient data — HIPAA, GDPR, and friends

If the literature plane ingests any clinical records, electronic health records, or radiology archives, you are now in the patient-data regime.

A minimal set of controls:

- **De-identification at ingestion.** Strip HIPAA Safe Harbor identifiers or perform expert determination before indexing. For GDPR, pseudonymisation with separated key management.
- **Record-level access control.** Not just per-table; per-document. The RAG retriever respects the requester's role.
- **No PHI in logs.** Edge scrubbers remove identifiers from observability streams.
- **Data residency.** Some institutions and jurisdictions require the data physically stay in-region. The architecture must support split deployments.
- **Right-to-erasure.** GDPR requires deletion paths; the KG must support edge-level deletion driven by a deletion event from the source-of-truth data.

If a literature pipeline can produce text containing patient identifiers, it has failed.

## IRB and clinical-research governance

Loops that drive any human-subjects research, including biomarker discovery on existing imaging cohorts, are subject to IRB / ethics-board oversight.

Engineering implications:

- **Approved protocol = artifact.** The planner's allowed action space is the approved protocol; out-of-protocol actions are rejected.
- **Protocol versions.** Every modification needs IRB amendment; the platform tracks which spec ran under which protocol version.
- **Consent management.** Subject consent attributes (re-contact, broad consent, data sharing) propagate into access controls.
- **Adverse-event reporting.** Triggered automatically when QC or downstream signals indicate a safety event; surfaced to the responsible investigator.

For paediatric or vulnerable cohorts, additional safeguards apply; the platform's equity-weighted planner does not relax oversight.

## Animal-subjects governance

For animal work, IACUC approval and Animal Use Protocol (AUP) versioning apply. The planner's action space must align with the approved AUP. The platform records:

- Subjects (animal IDs).
- Procedures and durations.
- Welfare scores (humane endpoints).
- Drug, dose, duration.

Welfare endpoints are a hard constraint, not a soft one.

## Model risk management

When models drive decisions, model-risk practices borrowed from finance and clinical-decision-support apply:

- **Inventory.** Every model in the platform listed, with purpose, owner, version, training data, and validation evidence.
- **Validation.** Pre-deployment validation against a held-out cohort; documented.
- **Monitoring.** Drift detection (see [observability](observability.md)) tied to a re-validation trigger.
- **Change management.** Model swaps go through review; not a silent `pip install --upgrade`.
- **Decommissioning.** A retired model is archived, not deleted; results made from it stay traceable.

For clinical-decision-support tools (US FDA), the SaMD framework and Good Machine Learning Practice principles apply.

## LLM-specific risks

LLMs concentrate several risks:

- **Hallucination.** Verified citations only (see [RAG](../phd/rag-systems.md)).
- **Prompt injection.** A paper or KG entry contains text designed to manipulate the model. Sanitise retrieved content; never let retrieved instructions override system prompts.
- **Data leakage.** Sending PHI to an external API is a regulatory event. Use private deployments or on-prem inference for sensitive data.
- **Model swap surprises.** A vendor model update changes behaviour. Pin versions; re-validate before adoption.
- **Cost runaway.** Agent loops can burn dollars; budget caps and circuit breakers.
- **Over-trust.** Operators defer to model output. Counter with mandatory citation review for consequential decisions.

## Equity and fairness

A planner can quietly amplify access biases:

- Choose subjects from convenient cohorts; under-sample others.
- Choose conditions overstudied in well-funded diseases; under-sample neglected ones.
- Rank papers from high-impact journals; underweight regional research.

Mitigations the engineer can build:

- Equity-weighted acquisitions (up-weight under-represented strata).
- Fairness dashboards (track demographic mix vs. target).
- Stakeholder review of goal definitions.

These are an engineering concern *and* a governance concern. The platform must surface the fairness picture, not hide it.

## Access control

A useful minimum:

- Role-based access (read, write, run, configure).
- Per-cohort / per-project scoping; cross-project access is the exception.
- Secrets in a vault; rotated.
- API tokens scoped and expiring.
- Audit-log access requires its own role distinct from admin.
- "Break-glass" emergency access logged and reviewed.

## Audit and compliance

The audit log (see [reproducibility](reproducibility.md)) is the single most important compliance artifact.

It records: who proposed what, when, with which version, who approved, who ran, with which calibration, what result, who saw the result. For a clinical trial this is non-negotiable; for non-clinical research it is best practice; for an internal R&D lab it is what keeps the team honest.

## Drills and exercises

- **Incident drill.** Pick a scenario (PHI leak, runaway planner, audit-log corruption); walk through detection, escalation, mitigation, comms.
- **Tabletop with stakeholders.** PI, IRB liaison, biosafety officer, IT security, data-protection officer. Annual.
- **Restore drill.** Restore the system from backups in a test environment.
- **Red-team the hypothesis generator.** Have a security-minded reviewer try to elicit DURC-adjacent outputs; close the gaps found.

## When to say no

Some experiments should not be run by an autonomous loop. The judgement is institutional, not algorithmic. The platform engineer's job is to:

- Make "no" possible: hard constraints, human-in-the-loop gates.
- Make "no" cheap: out-of-bound experiments default to "ask a human", not "run anyway".
- Make "no" visible: dashboards of rejected proposals, with reasons.

A platform that has never said "no" to its planner is one rejection away from a regrettable headline.

## References (non-exhaustive)

- US NIH Office of Science Policy — DURC and P3CO policies.
- US HHS Common Rule (45 CFR 46) — human subjects research.
- HIPAA Privacy Rule, Safe Harbor de-identification.
- EU GDPR; UK DPA 2018.
- ISO/IEC 27001 — information security.
- ISO 14971, IEC 62304 — medical-device software.
- FDA Good Machine Learning Practice for Medical Device Development (joint principles, 2021+).
- CIOMS Guidelines on ethics of research involving humans.
- Public Health Service Policy on Humane Care and Use of Laboratory Animals (PHS).

These are not optional reading for any engineer responsible for a platform that touches biomedical research.

## Where to next

- [Reproducibility](reproducibility.md) — the audit trail that backs every claim.
- [Observability](observability.md) — the signals that surface drift before it's a finding.
- [Portfolio & roadmap](portfolio-roadmap.md) — staging the platform so safety is built in, not bolted on.
