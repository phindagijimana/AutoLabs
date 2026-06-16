# Knowledge graphs

> *A network of biomedical concepts and how they relate. Building one, querying one, and not lying with one.*

A knowledge graph (KG) turns a pile of extracted facts into a queryable structure. Once you have one, you can ask questions like "what proteins connect drug A to disease D through at most two intermediate nodes?" — questions a search engine cannot answer.

## Nodes and edges, concretely

```
:Drug{name: "Phenytoin", id: "DB00252"}
:Protein{name: "SCN1A", id: "HGNC:10585"}
:Disease{name: "Hippocampal sclerosis", id: "MONDO:0008029"}

(:Drug)-[:INHIBITS]->(:Protein)
(:Protein)-[:ASSOCIATED_WITH]->(:Disease)
```

Each node has a *type* (Drug, Protein, Disease, etc.). Each edge has a *relation* (`INHIBITS`, `ASSOCIATED_WITH`). Both should resolve to controlled-vocabulary IDs, not to free strings.

## Two flavours of graph

| Flavour | Storage | Strength |
| --- | --- | --- |
| **Property graph** (Neo4j, Memgraph, KuzuDB) | Nodes and edges with key-value properties. Cypher query language. | Easy to learn, great for traversal and pattern queries. |
| **RDF triple store** (Blazegraph, GraphDB, RDF4J) | Subject-predicate-object triples. SPARQL query language. | Standards-based; integrates with biomedical ontologies (OWL). |

For research and most application use, property graphs are easier. For interoperability with Bioportal / OBO ontologies, RDF wins. Many large groups run both, with periodic dumps between them.

## Building one

A pragmatic flow:

```mermaid
flowchart LR
  papers[Papers + databases] --> ner[Biomedical NLP]
  ner --> norm[Normalisation]
  norm --> rel[Relations + assertions]
  rel --> stage[(Staged facts)]
  stage --> dedupe[Dedup + provenance]
  dedupe --> graph[(Knowledge graph)]
```

### Sources

A serious biomedical KG never lives off one source. Common ingestion sources:

| Source | What it gives |
| --- | --- |
| PubMed | Titles + abstracts; >35M records. |
| PubMed Central (PMC OA) | Full text for open-access papers. |
| ClinicalTrials.gov | Registered trials, populations, outcomes. |
| DrugBank, ChEMBL, RxNorm | Drug data. |
| OMIM, Mondo, Orphanet | Disease data. |
| Gene Ontology, HGNC | Gene/function data. |
| Reactome, KEGG, WikiPathways | Pathway data. |
| UMLS | Cross-vocabulary glue. |
| MeSH | PubMed indexing. |

Roughly half the edges in a useful biomedical KG come from databases, not text. The text extractions add the unique recent findings.

### Schema

You need a schema. Without one, you'll end up with five different edge types that mean the same thing.

A simple starter schema for a neuro-focused KG:

```
Nodes:    Disease, Drug, Gene, Protein, Pathway, Brain_Region, Imaging_Feature,
          Symptom, Trial, Patient_Cohort
Edges:    INHIBITS, ACTIVATES, ASSOCIATED_WITH, PART_OF, MEASURED_BY,
          REPORTED_IN, TREATS, CAUSED_BY, RISK_FACTOR_FOR
```

Document the schema. Refer to existing ontologies (Mondo, Gene Ontology, NeuroFMA, NIDM) when their categories fit; copy their IDs.

### Provenance

Every edge carries:

- **Source(s)** — paper PMID, database release.
- **Extractor** — which NLP model, which version.
- **Confidence** — model probability or rule-based score.
- **Assertion type** — *positive*, *negated*, *speculated*.

Without provenance, the graph is unusable for science: you cannot answer "where did this come from?".

## Querying

A small Cypher example, given the graph above:

```cypher
MATCH (d:Drug)-[:INHIBITS]->(p:Protein)
              -[:ASSOCIATED_WITH]->(dis:Disease {name: "Hippocampal sclerosis"})
RETURN d.name, p.name
ORDER BY d.name
```

"What drugs inhibit a protein that is associated with hippocampal sclerosis?"

Two-hop queries like this are where KGs earn their keep. Three-hop and above is where they start to *suggest* (hypothesis generation, next chapter).

## Quality

Three failure modes haunt every biomedical KG:

1. **False edges.** The NLP got the relation backwards or missed negation. Mitigation: confidence thresholds; multi-source agreement requirement.
2. **Stale edges.** A paper has been retracted; a finding has been overturned. Mitigation: link to current paper status; re-run extraction periodically.
3. **Missing edges.** Important relations weren't extracted because the model didn't know the type. Mitigation: gap analysis against curated databases.

Periodic *manual* audits — a domain expert reviews a sample of edges — are the only reliable way to keep quality from drifting.

## Embedding the graph

For machine-learning use, KGs are often embedded:

| Method | Idea |
| --- | --- |
| TransE, ComplEx, RotatE | Learn embeddings such that `head + relation ≈ tail`. |
| Node2Vec, DeepWalk | Random-walk-based; cheap. |
| GraphSAGE, GAT, RGCN | Graph neural networks; aggregate neighbour information. |
| Knowledge-graph + text co-embeddings | Combine the structure with text from the source papers. |

Embeddings power:

- **Link prediction** ("are these two nodes likely to be connected?").
- **Hypothesis generation** (next chapter).
- **Drug repurposing** (the canonical industry use case).

## Honest warnings

- **A KG without provenance is a rumour engine.** Always trace edges back to sources.
- **A KG without versioning is unreproducible.** Tag releases; pin them in downstream pipelines.
- **A KG that ingests everything will drown in noise.** Curate sources; don't just hoover.
- **Schemas drift.** Plan for migrations; never silently rename node types.

## Tooling landscape

| Layer | Tools |
| --- | --- |
| Storage | Neo4j, Memgraph, KuzuDB, Blazegraph. |
| ETL | dbt (for the staging tables), Airflow / Prefect / Dagster for orchestration. |
| Schema / ontology | OBO toolkit, Robot, PyOBO. |
| Embeddings | PyKEEN, DGL, PyTorch Geometric. |
| Visualisation | Neo4j Bloom, Cytoscape, custom React apps. |
| Biomedical-specific platforms | Hetionet, BioCypher, IndraDB, ROBOKOP. |

`BioCypher` is worth knowing: a Python framework for building biomedical KGs in a reproducible way with ontology grounding.

## Where to next

- [Hypothesis generation](hypothesis-generation.md) — what you can ask the graph that you can't ask the literature.
- [PhD: KG construction](../phd/kg-construction.md) — schema design, distant supervision, harmonisation.
- [Engineer: architecture](../engineer/architecture.md) — running a KG as a production system.
