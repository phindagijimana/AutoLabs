# Reinforcement learning planners

> *When the autonomous lab is best framed as an MDP and an RL policy fits better than Bayesian optimisation.*

BO assumes you're picking one experiment at a time to maximise a single function. RL assumes a sequential decision process where actions have downstream effects. Some labs are MDPs; many are not. This chapter is about how to tell, and how to do RL when the framing fits.

## When BO is enough, and when it isn't

| Reach for | Conditions |
| --- | --- |
| BO | One-shot optimisation. Each experiment's only effect is the result it produces. |
| RL | Each experiment changes the *state* of the lab (cells differentiate, fermentation accumulates, mice learn). |
| RL | The objective is realised after a sequence of actions, not after each one. |
| RL | You can simulate the environment cheaply enough to train. |

Typical RL-fit settings: long bioprocess optimisation (fermenter pH/T schedules), behavioural training in model animals, multi-step organic synthesis route planning, adaptive clinical-trial designs.

## MDP framing

State $s_t$, action $a_t$, transition $p(s_{t+1} \mid s_t, a_t)$, reward $r(s_t, a_t)$, discount $\gamma$. The policy $\pi(a \mid s)$ maximises

$$
J(\pi) = \mathbb{E}_{\pi}\left[\sum_{t=0}^{T} \gamma^{t} r(s_t, a_t)\right].
$$

For a lab, the state is everything we know about the in-progress experiment + the lab's calibration + the prior history. The action is the next operation. The reward usually accumulates: small intermediate rewards for moving toward the goal, terminal reward for hitting the target.

## Algorithm choices

| Algorithm | When |
| --- | --- |
| **Tabular Q-learning / SARSA** | Tiny discrete state/action spaces. Toy work. |
| **DQN, Double-DQN** | Discrete actions, continuous state. |
| **PPO** | Continuous or discrete actions; the modern default. |
| **SAC** | Continuous actions; sample-efficient; entropy-regularised. |
| **Model-based RL (PETS, MBPO, Dreamer)** | When samples are expensive — i.e., real labs. |
| **Offline RL (CQL, IQL, BCQ)** | When you only have logged data and can't roll out new experiences. |

For autonomous labs, *model-based* and *offline* RL are far more relevant than the on-policy methods that dominate game RL. Real experiments are too expensive to do millions of rollouts.

## Sim-to-real

You cannot train RL from scratch on a real lab. The standard recipe:

1. Build a digital twin: a simulator of the assay / process.
2. Train the policy in simulation, often with domain randomisation.
3. Deploy with a *safe-exploration* wrapper that constrains real actions.
4. Continue learning from real data with offline updates.

Robotics' sim-to-real gap shows up here too. Calibrate the simulator against a small set of real experiments before trusting it.

## Reward shaping

Sparse rewards (only at the end) kill RL in expensive environments. Common shaping:

- **Curriculum** — start with simpler goals; widen as the policy improves.
- **Potential-based shaping** — add $\gamma \Phi(s_{t+1}) - \Phi(s_t)$; provably preserves the optimal policy.
- **Inverse RL** — recover a reward from expert behaviour (a human's prior protocols).
- **Preference-based RL** — humans compare pairs of trajectories; learn the reward from preferences.

## Safety

A policy that maximises reward will sometimes propose unsafe actions (overdose, prolonged stress, dangerous reagent volumes). Hard safety constraints:

- **Constrained MDPs** — solve $\max J(\pi)$ subject to $J_C(\pi) \le d$.
- **Shielding** — a hand-written safety layer vetoes unsafe actions before they hit the lab.
- **Lagrangian methods** — PPO with cost-augmented Lagrangian.

In any setting with biological harm potential, the shield is non-negotiable. RL alone is not enough.

## Multi-task and meta-RL

A planner trained for one assay should ideally transfer. Meta-RL (MAML-style), task-conditional policies, and contextual MDPs all push toward this. In practice the transfer story for lab RL is still weak — same lab, slightly different conditions, often retrain.

## Where it has worked

Recent published examples worth knowing:

- **Bioprocess fermentation optimisation** — RL has beaten heuristic schedules on yield.
- **Organic synthesis route planning** — RL + symbolic search (MCTS) for retrosynthesis.
- **Adaptive clinical-trial designs** — RL-style adaptive randomisation has FDA precedent (REMAP-CAP, I-SPY).
- **Behavioural-experiment scheduling** in rodent labs — small but real impact on training time.

Few of these are "pure" RL deployments. They're hybrid: BO inside an episode, RL across episodes; or RL with a heavy expert prior.

## A small PPO sketch

```python
import gymnasium as gym
from stable_baselines3 import PPO

env = LabEnv(             # your simulator
    assay="dose_response",
    cell_line="SH-SY5Y",
)
model = PPO(
    policy="MlpPolicy",
    env=env,
    learning_rate=3e-4,
    n_steps=2048,
    batch_size=64,
    gamma=0.99,
    verbose=1,
)
model.learn(total_timesteps=2_000_000)

# evaluation, then careful real-world rollout under a safety shield
```

The hard part is `LabEnv` — modelling the lab faithfully. Without that, the policy learns to game the simulator.

## Evaluation

| What | How |
| --- | --- |
| **Sample efficiency** | Reward vs. number of real experiments. |
| **Safety** | Frequency and severity of constraint violations during deployment. |
| **Robustness** | Performance under shifted dynamics (new batch of cells, new reagent lot). |
| **Sim-to-real gap** | Reward gap between simulator and live lab; smaller is better. |

## Honest warnings

- **Reward hacking is real.** Any reward you under-specify, the policy will exploit. Audit trajectories.
- **A policy trained in sim will fail in lab if calibration drifts.** Continually re-evaluate.
- **Offline RL is brittle without overlap.** If the deployed policy strays far from logged data, performance collapses. CQL / IQL help; nothing eliminates this.
- **You will be tempted to use RL for things BO handles fine.** Resist; BO is simpler, faster, and easier to defend.

## References

- Schulman J, et al. Proximal policy optimization algorithms. *arXiv:1707.06347.* 2017.
- Haarnoja T, et al. Soft actor-critic. *ICML.* 2018.
- Kumar A, et al. Conservative Q-learning for offline RL. *NeurIPS.* 2020.
- Janner M, et al. When to trust your model: model-based policy optimization. *NeurIPS.* 2019.
- Hafner D, et al. Mastering Atari with discrete world models. *ICLR.* 2021.

## Where to next

- [Active learning under constraints](active-learning.md) — generalises BO and RL.
- [LLM scientific reasoning](llm-scientific-reasoning.md) — language models above the RL loop.
- [Engineer: safety & governance](../engineer/safety-governance.md) — the shielding layer in production.
