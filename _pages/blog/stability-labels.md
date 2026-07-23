---
layout: single
title: "When Should a Robot Commit? Perturb-and-Replay Stability Labels for Long-Horizon Tasks"
permalink: /blog/stability-labels/
author_profile: true
---

<!-- DRAFT — not yet listed on /blog-posts/ -->

Action-chunking policies face a per-segment choice: **commit** to a long chunk of
actions, or **replan** every step. The right choice depends on whether the segment
absorbs small errors or amplifies them — so we measure exactly that, per timestep,
with a deliberate mistake: perturb once, replay the same actions, and watch whether
the gap to the unperturbed trajectory shrinks or grows.

## The algorithm, in math

**Setup.** Fix a demonstration and a probe timestep $$t_0$$. The simulator is reset
to the exact recorded state at $$t_0$$, and the recorded actions
$$a_{t_0}, \dots, a_{t_0+K-1}$$ are replayed for $$K = 24$$ steps (1.2 s at 20 Hz).
This *nominal branch* gives the reference trajectory
$$x^{\mathrm{nom}}(\tau)$$, $$\tau = 0,\dots,K$$, where $$x$$ is the task-space
vector (end-effector position concatenated with the object observation).

**Step 1 — inject one error.** For each of $$N = 8$$ perturbed branches, reset to
the same state and perturb only the position dimensions of the *first* action:

$$
\tilde a^{(n)}_{t_0} = a_{t_0} + \varepsilon_n,
\qquad
\varepsilon_n \sim \mathcal{N}\!\left(0,\; \sigma_u^2 I_3\right),
\qquad n = 1,\dots,N,
$$

then replay the remaining $$K-1$$ actions **unchanged** — nobody reacts. The scale
$$\sigma_u$$ is not arbitrary: it is set to the trained policy's validation action
RMSE, so the probe injects exactly the size of error the policy actually makes at
deployment. That is what ties the label to the error-compounding theory rather than
being a generic stability test.

**Step 2 — measure the divergence.** At every step $$\tau = 1,\dots,K$$, the gap of
branch $$n$$ to the nominal trajectory is

$$
d_n(\tau) = \max\!\left(\left\lVert x^{(n)}(\tau) - x^{\mathrm{nom}}(\tau)\right\rVert_2,\; 10^{-12}\right),
$$

and the branch-averaged log-gap is

$$
\ell(\tau) = \frac{1}{N}\sum_{n=1}^{N} \log d_n(\tau).
$$

**Step 3 — fit the growth rate.** The label statistic is the ordinary-least-squares
slope of $$\ell(\tau)$$ against $$\tau$$:

$$
\lambda(t_0)
= \frac{\sum_{\tau=1}^{K}\left(\tau - \bar\tau\right)\left(\ell(\tau) - \bar\ell\,\right)}
       {\sum_{\tau=1}^{K}\left(\tau - \bar\tau\right)^2},
\qquad
\bar\tau = \frac{1}{K}\sum_{\tau=1}^{K}\tau,
\quad
\bar\ell = \frac{1}{K}\sum_{\tau=1}^{K}\ell(\tau).
$$

This $$\lambda$$ is a finite-time Lyapunov exponent (the Benettin et al. 1980
estimator in finite-time, finite-perturbation form): the injected error evolves as
$$d(\tau) \approx d(0)\, e^{\lambda \tau}$$, so $$\lambda < 0$$ means the plant
absorbs the mistake and $$\lambda > 0$$ means it amplifies it.

**Step 4 — threshold with a deadband.** With a margin $$\delta$$,

$$
y(t_0) =
\begin{cases}
\textbf{unstable} & \lambda(t_0) > +\delta \\[2pt]
\textbf{stable}   & \lambda(t_0) < -\delta \\[2pt]
\textbf{deadband} & |\lambda(t_0)| \le \delta ,
\end{cases}
$$

where $$\delta$$ is not a free constant: it is the 95th percentile of
$$|\lambda|$$ over free-space stamps of the same dataset — the measured noise floor
of the probe itself, computed per dataset. Deadband stamps are an abstention, not a
third regime; they inherit the label of their confident neighbors, and a median
filter over ~3 consecutive stamps turns the sequence into contiguous segments.
Nothing is segmented by hand.

**The closed-loop variant.** Repeat the identical probe with one change: after the
same first-action injection, the perturbed branch is driven by the *trained policy*
from its own camera observations, replanning every step —

$$
\tilde a^{(n)}_{t_0+\tau} = \pi\!\left(o^{(n)}_{t_0+\tau}\right), \qquad \tau = 1,\dots,K-1,
$$

giving a second exponent $$\lambda_{cl}(t_0)$$ and label $$y_{cl}(t_0)$$. The
open-loop channel isolates the plant alone; the closed-loop channel adds the
policy's feedback loop, and asks whether reacting contains the error or feeds it.
(Every closed-loop probe step is a diffusion inference, so this channel runs on a
subsample: 50 demos, every 10th step, $$N = 4$$.)

## All possible combinations

Each channel reports one of three classes, giving nine combinations. The
recommended execution follows from two rules: the **open-loop class decides whether
committing is safe** (deadband counts as safe to commit — in a neutral segment the
policy is the only error source, so each replan injects one fresh error that
nothing absorbs; committing chunks of $$k$$ accumulates $$T/k$$ errors instead of
$$T$$), and the **closed-loop class decides whether reacting helps or hurts**.

|                 | CL stable       | CL deadband     | CL unstable        |
|-----------------|-----------------|-----------------|--------------------|
| **OL stable**   | commit long $$k$$ | commit long $$k$$ | commit long $$k$$    |
| **OL deadband** | commit long $$k$$ | commit long $$k$$ | commit long $$k$$    |
| **OL unstable** | replan per step | short $$k$$       | intermediate $$k$$   |

The interesting variation is confined to the open-loop-unstable row; everywhere
else commitment wins. The reasons per cell, briefly: in the top-right cell
replanning is the destabilizing choice, so commitment is not just cheaper but
safer; in the middle cell the policy is the only error source, so fewer replans
mean fewer errors; in the bottom row, per-step replanning is right only when
feedback genuinely rescues (bottom left), and when both channels are expansive the
best that can be done is an intermediate $$k$$ that keeps some reaction while
minimizing injected noise.

Empirically the mass concentrates in two cells: transport segments land in
(OL stable or deadband, CL unstable), and contact segments land in
(OL unstable, CL unstable). The rescue cell (OL unstable, CL stable) is rare, and
the whole CL-stable column is thin because the closed-loop channel is positive at
90–99% of stamps.
