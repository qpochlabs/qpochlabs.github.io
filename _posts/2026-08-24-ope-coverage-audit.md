---
title: "The 95% confidence interval that was right 0% of the time"
description: "I audited the confidence intervals RL practitioners actually use for off-policy evaluation on a tiny gridworld with exact ground truth. Nominal 95% intervals from bootstrap FQE covered the truth 0% of the time at N=10 episodes, and no standard interval was trustworthy below ~200 episodes on this task."
---

Last week I ran a nominal 95% confidence interval a hundred times against a ground truth I could compute exactly, to the sixth decimal place. It contained the truth in zero of the hundred runs. Not 80%, not 50%. Zero. And it came from bootstrap FQE (fitted Q-evaluation, which learns a value function from logged data, with intervals built by resampling that data), one of the estimators people treat as the sensible default when deciding whether an RL policy is safe to deploy.

> **TL;DR**
> - On a 25-state gridworld with exact ground truth, nominal 95% intervals from bootstrap FQE and doubly-robust estimators covered the true policy value in as few as 0% and 5% of replicates at N=10 episodes, and WIS intervals were still 12 to 17 points below nominal at N=200.
> - Below roughly 200 episodes, no standard off-policy interval on this task was trustworthy, and even the on-policy baseline failed at tiny N because of a 1.3% rare event.
> - The payoff: once calibrated (N >= 200), FQE and DR intervals were 2 to 3x tighter than just running the policy, on this toy task.

## The gap nobody had filled

Off-policy evaluation (OPE) means estimating how well a new policy would perform using only data collected by an old one, without deploying the new policy. Every offline deploy/no-deploy decision ultimately rests on an uncertainty interval around an OPE estimate. So you would expect someone to have checked whether the intervals practitioners actually compute deliver their advertised coverage. The usual suspects: bootstrap intervals around importance sampling (reweighting old data by how likely the new policy was to take each action), fitted Q-evaluation (FQE, fitting a value function to the logged data), and doubly-robust hybrids (combining the model estimate with an importance-weighted correction). Coverage here means the plain thing: a 95% interval should contain the true value about 95% of the time.

Mostly, nobody had. The big empirical studies never audit intervals: [COBS](https://arxiv.org/abs/1911.06854) compares roughly 30 estimators on point-estimate accuracy, and [DOPE](https://arxiv.org/abs/2103.16596) benchmarks value error and ranking metrics. The methods literature keeps proposing new interval constructions instead: high-confidence bounds ([Thomas & Brunskill 2016](https://arxiv.org/abs/1604.00923)), bootstrap corrections ([Kostrikov & Nachum 2020](https://arxiv.org/abs/2007.13609)), empirical-likelihood intervals ([CoinDICE](https://arxiv.org/abs/2010.11652)), and the conformal OPE line from 2022 onward ([Taufiq et al. 2022](https://arxiv.org/abs/2206.04405); see also Foffano, Russo & Proutiere 2023 in that same line).

One honest caveat up front: this is not the first coverage number ever published. [Hao et al. 2021](https://arxiv.org/abs/2102.03607) embedded a coverage-vs-episodes study for their own bootstrap-FQE method and noted undercoverage at 10 episodes. What I could not find in an adversarial three-search sweep of 2022+ work is any paper whose primary contribution is a cross-estimator audit of the intervals practitioners currently use, with sample-size guidance attached. That sweep had limited recall (the OpenAlex queries were noisy), so if you know a paper I missed, please send it.

## The experiment: a world small enough to compute the truth

The whole design hinges on one choice: an environment where the true policy value is exact, so any interval failure is unambiguously the interval's fault. I used a 5x5 slippery gridworld (slip probability 0.2, goal +1, pit -1, gamma 0.99, horizon 50). The target policy is a noisy greedy policy; the behavior policy that generates the data mixes in 30% uniform noise, with action probabilities known exactly, so importance weights are exact too. Ground truth comes from finite-horizon dynamic programming: V = 0.874020, cross-checked against 50,000 Monte Carlo rollouts (0.873701 +/- 0.000959, z = -0.33).

<figure>
  <img src="/assets/blog/ope-coverage-audit/gridworld-env.png" alt="A 5 by 5 grid with the start cell in the top-left corner, a pit worth minus one in the exact center, and the goal worth plus one in the bottom-right corner, alongside a slip-dynamics panel showing the intended move succeeding with probability 0.8 and slipping to each perpendicular direction with probability 0.1.">
  <figcaption>Figure 1 — The environment: a 5×5 slippery gridworld with the start at state 0, a −1 pit dead center, and the +1 goal in the far corner. Every move goes as intended with probability 0.8 and slips sideways 0.1 each way.</figcaption>
</figure>

I made the return distribution deliberately safety-flavored: the target policy hits the pit 1.3% of the time, giving a skew of -8.3. Then: datasets of N in {10, 50, 200, 1000} behavior episodes, 100 independent replicates per N, four estimators (IS, WIS, DR, tabular FQE), percentile bootstrap (B=200) and normal-approximation intervals, plus an on-policy Monte Carlo baseline at matched budget. Monte Carlo error on each coverage number is about 2 to 3 points.

The whole pipeline in one picture: the behavior policy generates the offline datasets along the top path, dynamic programming computes the exact truth along the bottom path, and the audit checks how often each estimator's interval contains that truth.

<figure>
  <img src="/assets/blog/ope-coverage-audit/experiment-pipeline.png" alt="Pipeline diagram: a slippery gridworld feeds a behavior policy that generates offline datasets for four estimators with bootstrap confidence intervals, and a target policy whose exact value comes from dynamic programming; both paths converge on the coverage audit, which checks whether each 95% interval contains the true value over 100 replicates.">
  <figcaption>Figure 2 — The audit pipeline: offline estimates flow along the top path, the exact truth along the bottom, and coverage is checked where they meet.</figcaption>
</figure>

Empirical coverage of nominal **95%** intervals:

| Estimator / CI | N=10 | N=50 | N=200 | N=1000 |
|---|---|---|---|---|
| IS / bootstrap | 0.67 | 0.88 | 0.90 | 0.93 |
| IS / normal | 0.70 | 0.87 | 0.87 | 0.92 |
| WIS / bootstrap | 0.78 | 0.82 | 0.83 | 0.92 |
| WIS / normal | 0.62 | 0.78 | 0.78 | 0.91 |
| DR / bootstrap | **0.05** | **0.33** | 0.93 | 0.91 |
| FQE / bootstrap | **0.00** | **0.05** | 0.96 | 0.93 |
| On-policy MC / normal | 0.29 | 0.38 | 0.90 | 0.92 |

<figure>
  <img src="/assets/blog/ope-coverage-audit/coverage_vs_N.png" alt="Empirical coverage of nominal 95% confidence intervals versus dataset size for seven estimator and interval combinations, all falling well below the nominal line at small N.">
  <figcaption>Figure 3 — Empirical coverage of nominal 90% and 95% intervals vs dataset size. FQE and DR bootstrap intervals collapse to near-zero coverage at small N; WIS is the last to recover.</figcaption>
</figure>

Not a single estimator-interval combination stayed within 5 points of nominal at all N <= 200. Under my pre-stated rule (miscalibration if any N <= 200 cell misses nominal by more than 10 points), miscalibration was demonstrated many times over.

The FQE and DR collapse has a clean mechanism, and it is not heavy tails. With 10 episodes, most (state, action) pairs are never visited, and the fitted Q-function defaults them to zero. That produces a strong pessimistic bias in the point estimate (about -0.24 to -0.25) while the bootstrap, resampling the same blinkered data, produces narrow intervals around the wrong value. Confidently wrong is the worst failure mode an interval can have, and the "safe default" DR estimator was among the worst calibrated at small N. That was one of the surprises I had pre-registered as "would surprise me" in the plan, and it happened.

Two more surprises. First, WIS (weighted importance sampling, which normalizes the weights to cut variance) was the only estimator still failing at N=200, sitting 12 to 17 points below nominal, after everything else had recovered. Low variance, persistent bias. Second, the honest complication: the on-policy baseline also failed at tiny N (0.29 and 0.38 coverage at N <= 10 and 50). With a 1.3% catastrophic event, small samples usually contain zero catastrophes, so every variance-based interval collapses around an optimistic value. The tiny-N failure is partly universal rare-event skew, not an OPE problem. The OPE-specific findings are the far more extreme, bias-driven FQE/DR collapse (0.00 to 0.05 versus 0.29 to 0.38) and WIS still failing at N=200 after on-policy has recovered.

## How many episodes before you can trust the interval

The audit only earns its keep if it translates into guidance. N* is the smallest tested N where 95% coverage lands in [90%, 100%]: 200 episodes for bootstrap IS, DR, FQE, and on-policy MC; 1000 for WIS (both interval types) and normal-approximation IS. Note the grid: N* is resolved only on {10, 50, 200, 1000}, so the true thresholds sit somewhere between grid points.

Width is where OPE redeems itself. Median 95% interval width relative to on-policy Monte Carlo at matched budget:

| Estimator / CI | N* | Width vs on-policy (N=200) | Width vs on-policy (N=1000) |
|---|---|---|---|
| FQE / bootstrap | 200 | 0.35x | 0.31x |
| DR / bootstrap | 200 | 0.51x | 0.59x |
| WIS / bootstrap | 1000 | 0.82x | 0.88x |
| IS / bootstrap | 200 | 9.5x | 10.6x |

<figure>
  <img src="/assets/blog/ope-coverage-audit/width_vs_N.png" alt="Median 95% confidence interval width versus dataset size on a log axis, showing importance sampling intervals roughly ten times wider than the rest.">
  <figcaption>Figure 4 — Median 95% interval width vs dataset size (log scales). IS intervals are honest but roughly 10x wider than the rest; calibrated FQE and DR are the tightest.</figcaption>
</figure>

So the practitioner takeaway, scoped strictly to this toy task: below ~200 episodes, trust nothing. At 200+, prefer bootstrap FQE or DR; once calibrated, their intervals are 2 to 3x tighter than actually running the policy on-policy at the same episode budget. IS is calibrated at 200 but roughly 10x wider than just running the policy (honest but useless). Distrust WIS intervals until around 1000 episodes.

## What this does and does not show

This is a 25-state tabular world with exact importance weights, exact ground truth, and a hand-picked rare-event structure. Real offline RL has function approximation, estimated behavior policies, and estimated weights, none of which will make calibration easier. That is exactly why the framing is "even in the easiest possible setting, standard intervals fail at small N": the failure here is a floor, not a ceiling. Add the N* grid coarseness and the limited-recall novelty sweep, and the right reading is "a confirmed, consequential problem on the easiest instance," not a field-wide measurement.

## What's next

The obvious follow-up is the same audit at D4RL scale: continuous control, function-approximation FQE, learned behavior policies, with the deliverable being per-estimator N* tables a practitioner can actually use. Two backlog directions from this run also survived: anytime-valid stopping rules for policy certification (confidence sequences that answer "how many eval rollouts before I can certify this policy" without fixing N in advance), and testing whether formal robustness certificates on policy networks actually predict return degradation under distribution shift, which nobody appears to have checked empirically.

Everything above is reproducible from the run artifacts: two Python files (numpy and matplotlib only), fully seeded with MASTER_SEED=20260824, about 20 seconds on a laptop CPU. If a 95% interval can be wrong 100 times out of 100 in a world this small, it is worth 20 seconds to check yours.

## References

1. Voloshin, Le, Jiang, Yue (2021). *Empirical Study of Off-Policy Policy Evaluation for Reinforcement Learning* (COBS). NeurIPS Datasets & Benchmarks. [arXiv:1911.06854](https://arxiv.org/abs/1911.06854)
2. Fu et al. (2021). *Benchmarks for Deep Off-Policy Evaluation* (DOPE). ICLR. [arXiv:2103.16596](https://arxiv.org/abs/2103.16596)
3. Thomas & Brunskill (2016). *Data-Efficient Off-Policy Policy Evaluation for Reinforcement Learning*. ICML. [arXiv:1604.00923](https://arxiv.org/abs/1604.00923)
4. Kostrikov & Nachum (2020). *Statistical Bootstrapping for Uncertainty Estimation in Off-Policy Evaluation*. [arXiv:2007.13609](https://arxiv.org/abs/2007.13609)
5. Dai, Chow, Nachum, Li, Szepesvari, Schuurmans (2020). *CoinDICE: Off-Policy Confidence Interval Estimation*. NeurIPS. [arXiv:2010.11652](https://arxiv.org/abs/2010.11652)
6. Taufiq, Ton, Cornish, Teh, Doucet (2022). *Conformal Off-Policy Prediction in Contextual Bandits*. NeurIPS. [arXiv:2206.04405](https://arxiv.org/abs/2206.04405)
7. Hao, Ji, Duan, Lu, Szepesvari, Wang (2021). *Bootstrapping Fitted Q-Evaluation for Off-Policy Inference*. ICML. [arXiv:2102.03607](https://arxiv.org/abs/2102.03607)
8. Foffano, Russo, Proutiere (2023). *Conformal Off-Policy Evaluation in Markov Decision Processes*. (No link — I could not verify the arXiv ID during the literature sweep.)
