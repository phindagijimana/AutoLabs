# Hypothesis generation

> *Using literature, graphs, and language models to suggest scientific ideas worth testing.*

A hypothesis generator does not prove anything. It produces a *ranked list of plausible candidates* — drugs that might be repurposed, genes that might explain a phenotype, biomarkers worth measuring. The scientist still tests, but the search starts somewhere better than random.

## Three families of approach

| Approach | Idea | Best when |
| --- | --- | --- |
| **Literature-based discovery (LBD)** | Connect what's known via shared intermediates (Swanson's ABC pattern). | Vast literature, sparse direct evidence. |
| **Graph link prediction** | Score *missing* edges in a KG using embeddings. | You have a clean, large KG. |
| **LLM brainstorming with retrieval** | Ask an LLM, ground it in retrieved passages. | Search space is fuzzy; you want a rationale alongside. |

Real systems combine them. The LBD edge gives a candidate; the link prediction ranks it; the LLM writes the rationale that the scientist actually reads.

## ABC: literature-based discovery

The original idea (Don Swanson, 1980s): if many papers say A relates to B, and many say B relates to C, but few or none say A relates to C, then *A → C* is a candidate hypothesis.

Swanson used this to suggest fish-oil for Raynaud's syndrome from non-overlapping clusters of papers. The hypothesis was later supported by clinical data.

Modern LBD systems do this at scale:

```python
def abc_candidates(kg, target_C, min_support_AB=3, min_support_BC=3):
    candidates = {}
    for b in kg.neighbours(target_C):
        if kg.edge_count(b, target_C) < min_support_BC:
            continue
        for a in kg.neighbours(b):
            if a == target_C:
                continue
            if kg.edge_count(a, b) < min_support_AB:
                continue
            if kg.has_edge(a, target_C):
                continue  # already known
            candidates.setdefault(a, []).append(b)
    # rank by number of intermediates and average evidence weight
    return sorted(candidates.items(),
                  key=lambda kv: (-len(kv[1]), -sum(kg.edge_weight(a, b)
                                                    for b in kv[1])))
```

The key tuning parameters: minimum-evidence thresholds (otherwise you drown in noise) and edge weighting (more recent / higher-confidence edges weigh more).

## Link prediction with KG embeddings

If you've embedded the KG (last chapter), every node has a vector. Score a missing edge with a tensor product or a relation-specific function:

```python
def score(head, relation, tail, embeddings):
    h, r, t = embeddings[head], embeddings[relation], embeddings[tail]
    return -np.linalg.norm(h + r - t)   # TransE style; higher = better
```

You can now rank, for a given disease, every drug by predicted `TREATS` score. The top ranks are hypotheses.

In practice:

- Use `PyKEEN` to train embeddings on a biomedical KG.
- Hold out a slice of known edges; check whether the model recovers them.
- Calibrate the score — raw scores aren't probabilities.
- Look at the *neighbourhoods* of the top candidates. If they're all reasonable, the model has signal; if they're nonsense, the embedding has collapsed.

## LLM brainstorming with retrieval (RAG)

A retrieval-augmented LLM can produce candidate hypotheses with rationales. The pattern:

```mermaid
flowchart LR
  q[Scientific question] --> retr[Retriever]
  retr -->|relevant passages| llm[LLM]
  llm -->|hypothesis + rationale + citations| reviewer[Scientist review]
```

Concrete prompt skeleton (for an LLM with a retrieval tool wired in):

```
You are a biomedical research assistant. Given the question:
"What signalling pathways might link hippocampal sclerosis to early childhood febrile seizures?"

1. Retrieve the most relevant passages from PubMed.
2. Propose up to 5 candidate pathways.
3. For each, cite the passages that support it, and the passages that contradict it.
4. Mark any candidate where evidence is weak.
```

This pattern is increasingly used in drug-discovery groups and clinical-research groups. The honest qualifier: the model will sometimes fabricate citations. Always link to the actual PMID and verify before acting.

See [PhD: retrieval-augmented generation](../phd/rag-systems.md) for the implementation.

## Evaluating hypothesis generators

Evaluation is the hard part. You can't easily run a randomised trial of every candidate, so most groups use one of:

| Eval method | Description |
| --- | --- |
| **Temporal hold-out** | Train on edges up to year T; ask the system to predict edges that appeared in year T+1. |
| **Known-positives recall** | Inject known drug-repurposing successes; check whether the system would have suggested them. |
| **Expert review** | Domain experts rate top-K candidates as "worth testing", "plausible but known", "implausible". |
| **Downstream wet-lab validation** | The gold standard; expensive; only for top candidates. |

Temporal hold-out is the most common and the most defensible.

## Ranking that doesn't lie

A hypothesis generator that ranks *everything* as plausible is useless. Two practical patterns:

- **Score calibration.** Use isotonic regression on temporal hold-out so that "score 0.9" actually means ~90% likely-to-be-supported.
- **Diverse top-K.** Penalise candidates similar to ones already in the list. The scientist wants a *portfolio*, not five variants of the same idea.

## Where this lives in an autonomous lab

```mermaid
flowchart LR
  lit[Literature KG] --> gen[Hypothesis generator]
  gen --> rank[Ranked candidates]
  rank --> review[Scientist review + selection]
  review --> planner[AI planner]
  planner --> robot[Robot]
  robot -->|results| lit
  robot -->|results| planner
```

The hypothesis generator narrows the search space *before* the planner starts. Without it, the planner explores blindly. Without the planner, the hypothesis generator's ideas never get tested. They are complementary.

## Honest warnings

- **Confirmation bias.** A list of "AI-suggested" candidates can become a self-fulfilling prophecy. Run negative controls — candidates the system rated low — at least occasionally.
- **Hallucinated citations.** LLMs invent PMIDs. Always link-check.
- **Publication-bias amplification.** Hypotheses become candidates because *something was published*. Diseases with little funding are under-represented.
- **Ethical scope.** Hypothesis generators can suggest experiments that are unsafe or unethical (gain-of-function, dual-use). Filter early.

## Where to next

- [Systematic-review support](systematic-review.md) — the structured-evidence counterpart.
- [PhD: LLM scientific reasoning](../phd/llm-scientific-reasoning.md) — what's possible and what isn't.
- [Engineer: safety & governance](../engineer/safety-governance.md) — keeping a hypothesis generator inside the guardrails.
