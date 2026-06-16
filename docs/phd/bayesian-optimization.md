# Bayesian optimisation for experiments

> *Surrogate models, acquisition functions, and the extensions that make BO actually fit a real lab.*

Bayesian optimisation (BO) is the dominant framework for autonomous-lab planners today. This chapter assumes you've met BO at the "scikit-optimize tutorial" level and need to go further: choose the right surrogate, pick a defensible acquisition function, and handle batches, multiple objectives, constraints, and cost.

## The setup

We want to maximise an unknown function $f : \mathcal{X} \rightarrow \mathbb{R}$, observed only via noisy samples $y = f(x) + \varepsilon$.

The protocol:

1. Maintain a probabilistic surrogate $p(f \mid \mathcal{D}_n)$ given history $\mathcal{D}_n = \{(x_i, y_i)\}_{i=1}^{n}$.
2. Choose $x_{n+1} = \arg\max_{x} \alpha(x \mid \mathcal{D}_n)$ where $\alpha$ is the acquisition function.
3. Observe $y_{n+1}$. Append to $\mathcal{D}$. Repeat.

The art is in step 1 (the surrogate) and step 2 (the acquisition).

## Surrogate models

### Gaussian processes (GPs)

The default. A GP defines a prior over functions:

$$
f \sim \mathcal{GP}(m, k)
$$

with mean function $m(x)$ and kernel $k(x, x')$. The posterior given $\mathcal{D}_n$ is closed-form:

$$
\mu_n(x) = k_n(x)^{\top}(K_n + \sigma_\varepsilon^2 I)^{-1}\mathbf{y}, \quad
\sigma_n^2(x) = k(x, x) - k_n(x)^{\top}(K_n + \sigma_\varepsilon^2 I)^{-1} k_n(x).
$$

Kernel choice matters more than people expect.

| Kernel | When |
| --- | --- |
| **RBF** | Smooth, well-behaved $f$. The "default" but often too smooth for real labs. |
| **Matérn-5/2** | Mildly less smooth than RBF; usually a safer default. |
| **Matérn-3/2** | Rough functions; neuro / biology often. |
| **Additive kernels** | Decomposes $f$ across input groups. Useful when factors don't interact much. |
| **Categorical kernels** | When $\mathcal{X}$ has categorical dimensions (cell lines, compound classes). |

Hyperparameters (length scales, signal variance, noise) are fit by marginal-likelihood maximisation. Restart from multiple initialisations; the surface is multi-modal.

### When GPs break

GPs scale $\mathcal{O}(n^3)$. Beyond a few thousand points, switch to:

- **Sparse / inducing-point GPs** (FITC, SVGP).
- **Random-feature approximations.**
- **Bayesian neural networks** with deep kernels or with stochastic-weight-averaging Gaussian (SWAG) posteriors.
- **Tree-based surrogates** (BORE, SMAC) — robust on heterogeneous, high-dim, mixed-type spaces.

Most autonomous labs never get to $n = 1000$. GPs are fine for the typical regime.

## Acquisition functions

### Expected Improvement (EI)

$$
\alpha_{\mathrm{EI}}(x) = \mathbb{E}_{f \sim p(f \mid \mathcal{D}_n)}\big[\max\big(f(x) - f^{*}, 0\big)\big]
$$

with $f^{*}$ the current best observed value. For GPs this has a closed form involving $\Phi$ and $\phi$. A widely-used default, tends to under-explore.

### Upper Confidence Bound (UCB)

$$
\alpha_{\mathrm{UCB}}(x) = \mu_n(x) + \beta_n^{1/2}\,\sigma_n(x)
$$

The $\beta_n$ schedule trades off exploration and exploitation. With $\beta_n = 2\log(n^d t^2 \pi^2 / 6\delta)$ you get sub-linear regret bounds (Srinivas et al., 2010).

### Thompson Sampling

Draw $\tilde f$ from $p(f \mid \mathcal{D}_n)$; pick $x = \arg\max \tilde f(x)$. Nicely handles batched / parallel settings.

### Knowledge Gradient (KG)

$$
\alpha_{\mathrm{KG}}(x) = \mathbb{E}\big[\max_{x'} \mu_{n+1}(x') \mid x_{n+1} = x\big] - \max_{x'} \mu_n(x')
$$

KG looks one step ahead — the expected improvement in the *best believed value* after the next observation. Better than EI when observations are noisy.

### Entropy-based (PES, MES)

Predictive Entropy Search and Max-value Entropy Search choose the next point to reduce uncertainty about the maximiser or the maximum value. Strong empirical results; more compute per step.

## Batch BO

You can run 96 wells in parallel. You want 96 distinct, diverse, *jointly informative* points, not 96 copies of the current EI optimum.

| Method | Idea |
| --- | --- |
| **q-EI** | The joint expected improvement over a batch. Closed-form only for small q with GP; otherwise Monte Carlo. |
| **Local penalisation** | After picking a point, penalise the acquisition near it. |
| **Thompson sampling for batches** | Draw $q$ posterior samples; pick the maximiser of each. |
| **DPP-based diversification** | Sample diverse points via a Determinantal Point Process over the acquisition. |

In practice with BoTorch, `qExpectedImprovement` + `optimize_acqf` is the workhorse.

## Multi-objective BO

You care about both potency *and* toxicity. The "best" $x$ is now a Pareto front, not a point.

Acquisition functions:

- **Expected Hypervolume Improvement (EHVI)** — expected gain in the hypervolume of the Pareto front when adding $x$. Closed-form in low dimensions; Monte Carlo via `qEHVI` for batches.
- **Pareto-front Thompson sampling.**
- **Scalarised** — random weights on objectives at each iteration. Cheap; surprisingly good.

The honest practical move: define the trade-off you actually care about (e.g., maximise potency subject to toxicity $\leq T$). Constrained BO is often cleaner than multi-objective BO.

## Constrained BO

Some $x$ are infeasible (unsafe doses, unavailable reagents, physical limits).

- **Feasibility predictor** — a separate GP classifier on feasibility; multiply EI by feasibility probability.
- **Expected Constrained Improvement (EIC).**
- **Safe-BO** — never propose an $x$ that is unsafe with high probability (Sui et al.). Useful when feasibility is also expensive to observe.

## Cost-aware and multi-fidelity BO

Experiments differ in cost (reagents, time, ethics).

- **Cost-cooled acquisitions** — $\alpha(x) / c(x)$.
- **Multi-fidelity BO** — pick both *what* to test and at *which fidelity*. Cheap simulations vs. expensive wet-lab.
- **MF-MES, BOCA** — entropy-style multi-fidelity acquisitions.

This is the BO frontier most relevant to real labs and to most neuroscience experiments.

## High-dimensional BO

Real search spaces have hundreds of factors. Plain GPs collapse.

- **Random embeddings** (REMBO) — project into a low-dimensional subspace.
- **TuRBO** — local trust-region BO; multiple regions in parallel.
- **SAASBO** — sparsity-inducing priors on length scales; uncovers axis-aligned structure.
- **Bayesian Optimisation with Categorical Confounders** — for protein/chemistry spaces.

## Implementation: BoTorch sketch

```python
import torch
from botorch.models import SingleTaskGP
from botorch.fit import fit_gpytorch_mll
from gpytorch.mlls import ExactMarginalLogLikelihood
from botorch.acquisition import qExpectedImprovement
from botorch.optim import optimize_acqf

train_x = torch.tensor(history_x, dtype=torch.double)
train_y = torch.tensor(history_y, dtype=torch.double).unsqueeze(-1)

model = SingleTaskGP(train_x, train_y)
mll   = ExactMarginalLogLikelihood(model.likelihood, model)
fit_gpytorch_mll(mll)

best_f = train_y.max().item()
qEI = qExpectedImprovement(model, best_f=best_f)

bounds = torch.tensor([[0.0]*d, [1.0]*d], dtype=torch.double)
candidates, _ = optimize_acqf(
    acq_function=qEI,
    bounds=bounds,
    q=8,           # 8 wells in parallel
    num_restarts=10,
    raw_samples=512,
)
```

The lab-integrated version submits each row of `candidates` to the robot queue and feeds results back as new `(train_x, train_y)` rows.

## Evaluation

Public benchmarks worth knowing:

- **Olympus / DESIGN-BENCH** — chemistry / materials benchmarks for autonomous-lab BO.
- **Hartmann, Branin, etc.** — synthetic functions; useful for unit tests, not for real-world claims.
- **Drug-discovery-style benchmarks** (FreeSolv, ESOL, DEL hits).

For neuroscience: there's no canonical benchmark; treat your own data as the eval.

## Honest warnings

- **The GP prior is a strong assumption.** Misfit kernels give optimistic uncertainty. Inspect posterior at known points; cross-validate.
- **Acquisition functions assume the surrogate is calibrated.** If $\sigma_n$ is wrong, $\alpha$ is wrong.
- **Initialisation matters.** Without 5–20 well-spread seed points (Latin Hypercube), early BO is unstable.
- **Failure modes are silent.** A planner stuck on a noisy local optimum will sample there forever. Log acquisition values; alarm when they collapse.

## References

- Srinivas N, Krause A, Kakade SM, Seeger M. Gaussian process optimization in the bandit setting: no regret and experimental design. *ICML.* 2010.
- Balandat M, Karrer B, Jiang DR, et al. BoTorch: a framework for efficient Monte-Carlo Bayesian optimization. *NeurIPS.* 2020.
- Eriksson D, Pearce M, Gardner J, et al. Scalable global optimization via local Bayesian optimization. *NeurIPS.* 2019.
- Eriksson D, Jankowiak M. High-dimensional Bayesian optimization with sparse axis-aligned subspaces. *UAI.* 2021.

## Where to next

- [Active learning under constraints](active-learning.md) — generalises beyond BO.
- [Reinforcement learning planners](reinforcement-learning.md) — when MDP framing fits.
- [Engineer: architecture](../engineer/architecture.md) — running BO as a service.
