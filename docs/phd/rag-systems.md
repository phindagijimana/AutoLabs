# Retrieval-augmented generation

> *Retrieval + LLM, in patterns that actually survive contact with biomedical literature.*

RAG is the dominant pattern for biomedical literature tools that produce text. Done well, it grounds an LLM in real sources. Done badly, it papers over hallucinations with fake citations. This chapter is about the engineering that decides which.

## The architecture

```mermaid
flowchart LR
  q[Question / goal] --> exp[Query expansion]
  exp --> emb[Embedding]
  emb --> idx[(Vector index)]
  idx --> top[Top-K passages]
  top --> rer[Reranker]
  rer --> ctx[Context window]
  ctx --> llm[LLM]
  llm --> cite[Cite-and-answer]
  cite --> ver[Citation verifier]
  ver --> ans[Response]
```

Each box is its own engineering decision.

## Chunking

The unit you embed and retrieve.

| Granularity | Trade-off |
| --- | --- |
| Sentence | High precision; loses context. |
| Paragraph (~256 tokens) | Good default. |
| Section | Better recall; weaker grounding. |
| Whole-paper | Only for long-context models; expensive. |

A common scheme: split into ~512-token chunks with 20% overlap; preserve section headings as metadata; embed each chunk plus an enrichment string ("This passage is from the Methods section of [paper]").

For biomedical content, sentence-level for the *grounding facts* and paragraph-level for the *retrieval index* is a strong combination.

## Embeddings

| Model | When |
| --- | --- |
| **SPECTER / SPECTER2** | Paper-level embeddings; AllenAI. |
| **PubMedBERT / BioMedBERT-CLS pooled** | Biomedical text. |
| **BioMedRoBERTa** | Larger biomedical encoder. |
| **OpenAI text-embedding-3** / **Voyage 2 / 3** / **Cohere Embed** | General-purpose; competitive on biomedical. |
| **e5-large, BGE-large** | Strong open embeddings; cheap to host. |

Practical move: benchmark 2–3 embedders on a *small in-domain query set* (e.g., 200 biomedical questions where you know the right passages). Don't trust general MTEB rankings; biomedical is different.

## Indexing

For modest corpora (< 100M chunks), an in-process index (FAISS, Annoy, HNSW) is fine. For larger or shared access:

| System | Notes |
| --- | --- |
| **FAISS** | Standard library; HNSW + IVF-PQ; in-process. |
| **Milvus / Weaviate / Qdrant** | Vector DBs with metadata filtering, multi-tenancy. |
| **pgvector** | Postgres extension; good if you already run Postgres. |
| **OpenSearch / Elasticsearch with kNN** | If you already operate them. |

For biomedical, metadata filtering (year, journal, study type) is as important as vector similarity. The index must support hybrid (vector + keyword + structured filter) queries.

## Hybrid retrieval

Pure vector retrieval misses exact-term matches (a specific gene name, drug name, MeSH heading). Hybrid retrieval combines:

- **Dense** retrieval over embeddings.
- **Sparse** retrieval (BM25 or SPLADE) over tokens.

Then *reciprocal rank fusion* (RRF) or learned weighted combination produces the top-K.

For biomedical text, hybrid retrieval is rarely optional. Single-term queries (a CUI, a gene symbol) are common, and sparse retrieval dominates them.

## Reranking

A small cross-encoder reranks the top-K candidates by reading the question and each passage jointly.

| Reranker | Notes |
| --- | --- |
| **MiniLM cross-encoder** | Cheap, fast, surprisingly good. |
| **Cohere Rerank**, others (hosted) | Strong but API-dependent. |
| **Domain-tuned reranker** (e.g. RankGPT-bio) | Best when you can label a few thousand pairs. |

Reranking turns top-50 retrieval into top-5 *useful*. Without it, the LLM sees too many irrelevant passages and starts inventing.

## The prompt

For biomedical RAG, the prompt should:

1. Give the question.
2. Give the retrieved passages, each with an explicit ID.
3. Demand bracket-citations to those IDs for every fact.
4. Allow "I don't know" as an output.

A working template:

```
You are answering a biomedical question using only the passages below.

Question: {question}

Passages:
[P1] {chunk_1}
[P2] {chunk_2}
...

Rules:
- Answer using only the passages.
- Every factual statement must end with one or more bracket citations, e.g. [P3].
- If the passages don't answer the question, say "Insufficient evidence."
- Do not invent citations.

Answer:
```

This pattern, with a strict citation verifier downstream, is the difference between a useful tool and a confident liar.

## Citation verification

Post-generation, parse the output, check each `[Pk]` against the actual passage, and reject claims whose cited passage doesn't support them.

A pragmatic verifier:

1. Parse claims and their bracket-citations from the output.
2. For each claim, retrieve the cited passage.
3. Score "does the passage entail the claim?" with a small NLI model (e.g., DeBERTa-v3 NLI).
4. Reject claims below threshold.
5. Either return the verified-only answer, or re-prompt the LLM with the rejected claims removed.

This is the closest thing to a hallucination fix.

## Long documents

Some questions need a whole paper, not chunks. Options:

- **Map-reduce** — summarise each chunk; combine summaries; answer over combined summary.
- **Sliding-window summarisation** — produce a running summary; use it as context.
- **Long-context model** — Claude 200k+, GPT 128k, etc.

Each has costs. For systematic-review-style synthesis, map-reduce is most defensible.

## Multi-hop retrieval

Some questions need to chain retrievals: answer A, use A to retrieve more, answer B. Pattern:

```python
def multihop(query, k=5, max_hops=3):
    context = []
    for hop in range(max_hops):
        new = retrieve(query, context, k)
        context.extend(new)
        next_q = generate_followup(query, context)
        if next_q is None:
            break
        query = next_q
    return answer(query, context)
```

Verify each hop's grounding; multi-hop is where hallucination compounds.

## Evaluating RAG

| Axis | Metric |
| --- | --- |
| **Retrieval recall@K** | Of relevant passages, what fraction in top-K. |
| **Reranking nDCG@K** | Quality of ordering. |
| **Faithfulness** | Fraction of claims supported by cited passage. |
| **Answer relevance** | Does the answer address the question? |
| **Answer correctness** | Does it match a gold answer? |
| **End-to-end task success** | For agent settings — did the user's goal happen? |

Frameworks: RAGAS, ARES, TruLens. For biomedical: build a small in-domain eval set with annotated passages; rely on it more than public benchmarks.

## Honest warnings

- **Citations the LLM produces are not citations.** They're hints. Always verify against retrieved IDs.
- **Reranker errors are silent.** Without periodic relevance audits, retrieval quietly degrades.
- **Long-context models are not the answer to bad retrieval.** They tolerate noise but still get confused.
- **PDF parsing breaks RAG.** A bad parser produces garbage chunks; the LLM faithfully cites them. Audit the parser.
- **Privacy.** If you index clinical notes, retrieval can re-expose patient identifiers. De-identify before indexing.
- **Cost growth.** Each query embeds the question, retrieves, reranks, calls the LLM. At scale this is real money; cache aggressively.

## References

- Lewis P, et al. Retrieval-augmented generation for knowledge-intensive NLP tasks. *NeurIPS.* 2020.
- Formal T, et al. SPLADE: sparse lexical and expansion model for first-stage retrieval. *SIGIR.* 2021.
- Es S, et al. RAGAS: automated evaluation of retrieval-augmented generation. *EACL system demos.* 2024.
- Xiong G, et al. Benchmarking retrieval-augmented generation for medicine. *arXiv:2402.13178.* 2024.

## Where to next

- [LLM scientific reasoning](llm-scientific-reasoning.md) — RAG inside a tool-using agent.
- [Systematic-review support](../intermediate/systematic-review.md) — RAG inside PRISMA pipelines.
- [Engineer: observability](../engineer/observability.md) — measuring and alarming on faithfulness drift.
