# Glossary

> *Every recurring term in this handbook, in one place.*

Terms are grouped by the chapter where they first matter. If a term has its own chapter elsewhere, the entry is short and the chapter is linked.

## Part A — autonomous labs

**Autonomous lab.** A laboratory where AI plans experiments, robots run them, and software analyses results in a closed loop. Also called a *self-driving lab*.

**Closed-loop system.** Any system where the output of a step is fed back to inform the next step. In autonomous labs, the loop is: plan → run → measure → learn → plan again.

**AI planner.** The software component that picks the next experiment. May use [Bayesian optimisation](../phd/bayesian-optimization.md), [reinforcement learning](../phd/reinforcement-learning.md), [LLM-based reasoning](../phd/llm-scientific-reasoning.md), or simpler optimisation methods.

**Acquisition function.** In Bayesian optimisation, a small math object that scores how *informative* or *rewarding* an experiment would be. The planner picks the experiment with the best score.

**Active learning.** A general idea: instead of labelling random data, ask the model where it is most uncertain and label *there*. Autonomous labs are a physical version of active learning.

**Bayesian optimisation.** A way to find the maximum of a function you can only sample at cost, by maintaining a probabilistic guess at the function and choosing the next sample to either explore or exploit.

**Liquid handler.** A robot that moves precise volumes of liquid between containers. Workhorse of biology labs.

**High-throughput screening (HTS).** Running thousands of experiments in parallel, each at small scale (often in 96-, 384-, or 1536-well plates).

**Assay.** A measurement procedure that produces a number for a biological property. Cell-viability assay, kinase activity assay, etc.

**Plate.** A flat tray with a grid of small wells, used to run many assays in parallel.

**Closed-loop optimisation.** Closed-loop system + an objective the planner is trying to maximise (potency, yield, signal-to-noise).

**Feedback loop.** The arrows in the closed-loop diagram that send results back to the planner. The whole field's leverage lives here.

**Reinforcement learning (RL).** Training a policy that picks actions to maximise long-term reward. In autonomous labs, the actions are experiments and the reward is closeness to the goal.

**Reward function.** A formula that turns experiment outcomes into a single number the planner tries to make big.

**Cost-aware optimisation.** Optimisation that knows experiments differ in time, reagents, or risk, and trades expected gain against cost.

## Part B — literature synthesis

**Biomedical NLP.** Natural-language processing trained on biomedical text — papers, clinical notes, drug labels.

**Named-entity recognition (NER).** The NLP task of finding spans of text that name a real-world thing (a gene, a drug, a disease).

**Relation extraction.** Given two entities in a sentence, deciding whether they are related and how (*A inhibits B*, *A causes C*).

**Knowledge graph (KG).** A network of typed nodes (entities) and typed edges (relations).

**Ontology.** A formal vocabulary that defines what nodes and edges can exist. UMLS, MeSH, Gene Ontology, SNOMED CT are common biomedical ontologies.

**UMLS.** Unified Medical Language System — a giant integration of biomedical vocabularies maintained by the U.S. National Library of Medicine.

**MeSH.** Medical Subject Headings — the controlled vocabulary used to index PubMed.

**SNOMED CT.** A clinical terminology used widely in electronic health records.

**Embedding.** A numeric vector representing a word, sentence, or document, learned so that similar items have similar vectors.

**Retrieval-augmented generation (RAG).** A pattern: when a language model needs to answer something, *retrieve* relevant documents first, then *generate* using those documents as context. Reduces hallucination and grounds outputs in sources.

**Vector database.** A database optimised for nearest-neighbour search over embeddings. Used by RAG to retrieve relevant passages.

**Hypothesis generation.** Suggesting plausible scientific ideas worth testing, based on patterns in literature and data. Generation, not proof.

**Systematic review.** A formal, exhaustive summary of evidence on a clinical question. AI tools support screening, extraction, and synthesis.

**PRISMA.** The reporting standard for systematic reviews. AI tooling that supports systematic reviews is typically evaluated against PRISMA compliance.

**Living review.** A systematic review kept up to date continuously as new papers appear. Practical only with AI assistance.

**Evidence synthesis.** Umbrella term for combining findings across studies — systematic reviews, meta-analyses, scoping reviews.

**Hallucination.** When a generative model produces a confident-looking statement that is not supported by evidence. The single biggest failure mode of LLM-based literature tools.

## Cross-cutting

**Reproducibility.** The ability for someone else to repeat your result. Both halves of the handbook treat this as a primary requirement, not a nice-to-have.

**Provenance.** A record of where a fact, decision, or output came from. In an autonomous lab: which experiment, which calibration, which version of the planner. In literature synthesis: which paper, which extraction model, which prompt.

**Observability.** The ability to see what a running system is doing — logs, metrics, traces. Critical for systems that run for days unattended.

**Determinism.** A run that produces the same output every time given the same input. Hard to achieve when planners use sampling or LLMs.

**Closed loop with human override.** The realistic default: the loop runs autonomously, but a human can pause, inspect, or veto any step.

**Reference data / ground truth.** Trusted data used to evaluate a system. In autonomous labs: control experiments. In literature synthesis: curated benchmarks like BioCreative or BC5CDR.

**Benchmark.** A standard dataset and metric used to compare methods. The PhD chapters cite several common ones.
