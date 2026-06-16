# What is literature synthesis?

> *Using AI to read, extract from, and connect what is already known.*

Biomedical research publishes thousands of papers a day. Plus guidelines, clinical-trial records, drug-label updates, omics datasets. No human can keep up. **Literature synthesis** is the use of AI tools to do the reading, the extraction, and the connecting on a scientist's behalf.

It is not just better search. Search finds papers. Synthesis pulls *findings* out of papers, links them across sources, and surfaces patterns or contradictions.

## The four classic tasks

| Task | What it produces | Example |
| --- | --- | --- |
| **Biomedical NLP** | Structured facts pulled from free text. | "Paper X reports that drug A reduced tumour size by 32% in mouse model Y." |
| **Knowledge graphs** | A network of concepts and how they relate. | Drug A → inhibits → Protein B → part of → Pathway C → linked to → Disease D. |
| **Hypothesis generation** | Plausible new ideas worth testing. | "Drug A might also help disease E because they share pathway C." |
| **Systematic-review support** | Faster, more thorough literature reviews. | Screen 5,000 abstracts down to 80 relevant papers, extract the outcomes table, surface contradictions. |

This level explains each in plain language. The intermediate level shows the tools. The PhD level shows the methods.

## Biomedical NLP

NLP stands for *natural language processing*. **Biomedical NLP** is NLP trained on biomedical text — papers, clinical notes, drug labels.

It can extract:

- **Diseases, genes, proteins, drugs, chemicals.** ("Phenytoin", "BRCA1", "hippocampal sclerosis".)
- **Symptoms and outcomes.** ("Seizure freedom at 12 months", "30% reduction in tumour volume".)
- **Methods and populations.** ("Randomised controlled trial in 240 paediatric patients".)
- **Treatment effects.** ("Drug X reduced relapse rate vs placebo, p<0.01".)

These extracted facts become the raw material for everything else.

## Knowledge graphs

A **knowledge graph** is a network. Each node is a concept (a disease, a gene, a drug). Each edge is a relationship (*inhibits*, *causes*, *is part of*, *is associated with*).

Once thousands of papers are summarised this way, a researcher can ask questions you cannot ask a search engine:

- *What proteins are linked to hippocampal sclerosis through at least two intermediate genes?*
- *Which drugs target a pathway involved in both epilepsy and Alzheimer's disease?*
- *Are there contradictory reports about a particular biomarker?*

The graph is only as good as the extraction that built it. Garbage edges are the most common failure mode.

## Hypothesis generation

If you can see the graph, you can ask, "what's missing?" A drug that targets a protein in pathway C, and a disease also linked to pathway C, suggest a *hypothesis*: maybe the drug helps that disease.

AI does not *prove* the hypothesis. It points to ones worth testing. Most are wrong. The good ones save months of literature wandering.

Famous early example: a 1980s computer-aided literature analysis suggested fish oil might help Raynaud's disease because both connected to blood-viscosity papers, even though no single paper linked them directly. The hypothesis was later confirmed clinically.

Modern systems do the same idea at a vastly larger scale.

## Systematic-review support

A **systematic review** is a formal, exhaustive summary of evidence on a clinical question. Done by hand, it takes a team six to eighteen months: thousands of abstracts to screen, dozens of full papers to read, careful tables of outcomes to extract.

AI helps at every step:

- Pull all potentially relevant papers from multiple databases.
- Drop duplicates.
- Screen abstracts for relevance.
- Extract numerical outcomes into a table.
- Summarise the body of evidence.
- Flag contradictions and gaps.

A well-built pipeline can cut human time by 50–70%. It does not replace the human judgement step; it makes that step land on better-prepared material.

## A worked feeling for it: hippocampal sclerosis

Imagine you are starting a project on hippocampal sclerosis (HS), a hippocampal scarring pattern often found in drug-resistant epilepsy.

A literature-synthesis pipeline could:

1. Pull every paper, abstract, and clinical-trial record mentioning HS over the last twenty years.
2. Extract imaging biomarkers reported (T2 hyperintensity, volume loss, FLAIR asymmetry).
3. Extract methods (MRI sequences, scanner field strengths, segmentation pipelines).
4. Extract outcomes (seizure freedom at 1 year, 2 years, 5 years; complication rates).
5. Build a graph linking imaging features → histopathological grade → seizure laterality → surgical outcome.
6. Surface gaps — for instance, asymmetry thresholds that vary widely across studies, or paediatric cohorts that are under-represented.

You still write the paper. The synthesis gives you the starting map.

## Honest warnings

- **NLP can be confidently wrong.** A model that says "Drug A reduces X by 32%" might be mis-reading a sentence about Drug B. Always allow human verification on consequential edges.
- **Coverage is uneven.** PubMed is well-covered. Preprint servers, grey literature, and non-English papers are not.
- **The graph is a *summary*, not the science.** Use it to navigate. Read the original paper before you act on a finding.

## What this level intentionally skips

- The model architectures behind biomedical NLP — see [PhD: Relation extraction](../phd/relation-extraction.md).
- How to build a knowledge graph in practice — see [PhD: KG construction](../phd/kg-construction.md).
- How to evaluate hypotheses at scale — see [PhD: LLM scientific reasoning](../phd/llm-scientific-reasoning.md).

## Next

- [Glossary](glossary.md)
- Or jump to the [intermediate](../intermediate/index.md) level for components and tools.
