# Active learning under constraints

> *The unifying view of every planner: pick the experiment that most reduces uncertainty about the thing you care about, subject to what you can afford.*

BO and RL are special cases of *active learning*: a learner choosing what data to acquire next. Framing the lab as active learning, with real constraints, is the cleanest way to think about planner design.

## The pure objective

You want a posterior over the quantity of interest $Q$ — could be the maximiser, a parameter, a classifier, or a hypothesis. Active learning picks $x$ to maximise expected information about $Q$:

$$
\alpha(x) = \mathbb{E}_{y \sim p(y \mid x)}\big[\mathrm{H}(Q \mid \mathcal{D}_n) - \mathrm{H}(Q \mid \mathcal{D}_n \cup \{(x, y)\})\big].
$$

For function maximisation, $Q$ is "the maximiser". For classification, $Q$ is "the classifier parameters". For hypothesis testing, $Q$ is "which of these candidates is true".

Real labs do not optimise this pure objective; they optimise it under constraints.

## The constraints that matter

| Constraint | What it forces |
| --- | --- |
| **Cost asymmetry** | Some $x$ are cheap, some expensive. Acquisition / cost matters more than acquisition alone. |
| **Batching** | You run $q$ experiments per round. Diversity of the batch becomes part of the acquisition. |
| **Latency** | Results come back after $\tau$ time units. The planner must operate without immediate feedback. |
| **Feasibility** | Some $x$ are infeasible. Reject or penalise. |
| **Safety** | Some $x$ are unsafe. Hard constraint with high confidence. |
| **Ethics / regulation** | Some $x$ require approval. Out-of-loop until approved. |
| **Bias / equity** | Sampling decisions affect downstream model fairness. |
| **Reproducibility** | The sequence must be auditable; stochastic acquisition must be seeded. |

A planner that ignores any of these in production will fail in a way the team has to clean up.

## Latency and asynchronous loops

The clean diagram has a queue between the planner and the lab. While the planner waits for the result of experiment $t$, it can propose experiments $t+1, t+2, \dots$ using its *current* posterior — but those proposals are made *without seeing* the in-flight results.

Two patterns:

- **Fantasised observations.** Treat in-flight experiments as if they have expected outcomes; update the surrogate optimistically; propose the next experiment. Correct the surrogate when the real result arrives.
- **Asynchronous Thompson sampling.** Each in-flight worker draws its own posterior sample and proposes from that. No coordination needed between workers.

Both are well-understood; both are slightly less efficient than synchronous BO; both are nearly always worth it for real labs.

## Batch diversity

When a planner emits $q$ experiments at once, repetition is the enemy. Three working approaches:

- **Local penalisation.** After picking $x_1$, multiply the acquisition by $1 - \exp(-\|x - x_1\|^2 / \ell^2)$. Repeat.
- **q-acquisition** (q-EI, q-EHVI, q-KG). Maximise the joint acquisition over the batch.
- **Determinantal Point Processes** over the acquisition surface.

For categorical / discrete spaces (compound libraries), submodular set functions give clean diversity guarantees.

## Cost-aware acquisitions

Two equivalent framings:

- Divide acquisition by cost: $\alpha(x) / c(x)$.
- Constrain total cost: $\sum c(x_i) \le B$; solve for the acquisition-maximising sequence.

For multi-fidelity settings, cost is a per-fidelity table. The MES / MF-MES acquisitions handle this jointly.

## Safe active learning

The lab cannot try things that could harm cells, contaminate the deck, or violate biosafety rules. The planner must avoid them with high confidence even before any data shows them to be safe.

- **Safe-BO (SafeOpt).** Restrict exploration to a "safe set" of $x$ where the surrogate predicts safety with high probability; expand the set conservatively.
- **Gaussian-process constraints with chance bounds.** Acquisition multiplied by $\Pr(\text{safe})$ raised to a high power.
- **Hard veto layer.** A separate model (or rule set) approves every proposed $x$ before the lab executes it. Non-negotiable in regulated contexts.

## Robust active learning

The surrogate is wrong. The planner that *knows* the surrogate is wrong does better.

- **Risk-averse acquisitions.** Optimise CVaR or worst-case over surrogate ensembles.
- **Distributionally robust active learning.** Acquisition under an adversarial choice of the data-generating distribution within an ambiguity set.
- **Calibration audits.** Periodically check that the surrogate's predictive intervals contain the right fraction of new observations.

In labs with non-stationary state (cell culture drift, reagent variability), surrogate audits are essential — without them, the planner re-learns ghosts.

## Active learning beyond function maximisation

Function maximisation is only one mode. Others matter as much in labs:

| Goal | Acquisition family |
| --- | --- |
| **Level-set estimation** (find the dose where effect = 50%) | Straddle, LSE-EI, entropy-based level acquisitions. |
| **Pareto-front identification** | Multi-objective BO. |
| **Hypothesis testing** | Sequential information gain on a discrete hypothesis space. |
| **Model calibration** | Acquisition that maximises surprise vs. prior. |
| **Coverage** | Maximise minimum distance to existing samples; fill the space. |

For instance, a level-set acquisition is appropriate for "find the dose at which 50% of cells die" — quite different from "find the most lethal dose".

## Where it ties to BO and RL

BO ≈ active learning with $Q$ = maximiser, one experiment at a time, fixed environment, function-only feedback.

RL ≈ active learning with $Q$ = optimal policy, sequential decisions, state-dependent environment, reward-only feedback.

Most real labs need *active learning* — with cost, batches, latency, safety, and sometimes sequential effects — not pure BO or pure RL. Picking the right framing is the planner-design question.

## A more general loop sketch

```python
while not goal.satisfied(state):
    candidates = sample_candidates(state.search_space, k=2048)
    candidates = filter_feasible(candidates, state)
    candidates = filter_safe(candidates, state, conf=0.99)

    scores = [acquisition(x, state) / cost(x, state) for x in candidates]
    batch  = diversify(top_k(candidates, scores, k=q), state)

    for x in batch:
        scheduler.submit(x)

    completed = scheduler.collect(timeout="next_round")
    for x, y in completed:
        state.update(x, y)
        state.audit_calibration()      # surrogate health
    state.save()
```

Each line corresponds to a chapter or section of this handbook.

## Honest warnings

- **The pure-information objective is rarely what you want.** You almost always care about a downstream decision; choose an acquisition that reflects *that* decision.
- **Asynchrony breaks naive bounds.** The convergence theorems for BO assume synchronous observation; in practice, audit empirically.
- **Constraints can dominate.** A planner spending all its time waiting for safety approval is the bottleneck, not the planner.
- **Stop-criteria are under-discussed.** "Run for 100 cycles" is almost never the right answer. Define when to stop in terms of the decision the loop is supposed to inform.

## References

- Sui Y, et al. Safe exploration for optimization with Gaussian processes. *ICML.* 2015.
- González J, et al. Batch Bayesian optimization via local penalization. *AISTATS.* 2016.
- Bryan B, et al. Active learning for identifying function threshold boundaries. *NeurIPS.* 2005.
- Cakmak S, et al. Bayesian optimization of risk measures. *NeurIPS.* 2020.

## Where to next

- [Bayesian optimisation](bayesian-optimization.md) — the specific framework most labs reach for.
- [Reinforcement learning planners](reinforcement-learning.md) — for sequential-state problems.
- [Engineer: orchestration](../engineer/orchestration.md) — the scheduler that turns acquisitions into runs.
