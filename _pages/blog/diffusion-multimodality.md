---
layout: single
title: "Does Diffusion Actually Capture Multi-Modal Behavior? Re-Running a Myth-Buster's Tests"
permalink: /blog/diffusion-multimodality/
author_profile: true
---

*I also wrote this up as a thread on X — [read it here](https://x.com/arygoy/status/2074891040184795258?s=20).*

## The myth, and the myth-buster

For years the standard story in robot learning has been: diffusion and flow policies beat plain
regression policies because human demonstrations are **multi-modal** — at the same situation,
different demonstrators do different valid things (grasp from the left *or* the right) — and
regression averages those choices into a useless in-between action, while generative models can
keep both.

A recent paper, [*Much Ado About Noising* (Pan et al.)](https://arxiv.org/abs/2512.01809), put
that story to the test and busted it. They rolled out trained **flow** policies on the standard
benchmarks (Push-T, Kitchen, Tool-Hang) and found essentially **no multimodality in the trained
policies at all**: freezing the noise, sampling it randomly, or even *averaging* many sampled
actions changed task success by at most 0.04, and the sampled actions never formed distinct
modes. Whatever makes generative policies win, it isn't mode-capturing.

## Why I wanted to check diffusion too

Two things made me curious:

1. Their headline tests use **flow models** — a deterministic ODE whose only randomness is the
   initial noise. A DDPM **diffusion** policy injects fresh noise at every denoising step, which
   is a mechanically different (and arguably more robust) way of holding modes apart. A negative
   result for flow doesn't automatically carry over.
2. The paper itself admits its benchmarks may contain little multimodality to begin with. If the
   data has no modes, no policy can show any — so the null could be a property of the *datasets*,
   not the models.

So the experiment: train **diffusion and flow policies, identical in every other way**, on data
that *provably* contains modes, and re-run Pan et al.'s exact tests.

## Setup

**Data.** The RoboMimic **multi-human** datasets (`can`, `square`, `transport` — six different
operators demonstrating each task), plus `lift` (single proficient expert) as a control that
should have almost no modes. Before training anything, I audited the data: find each state's
nearest neighbours, cluster the actions taken there, and let an elbow criterion pick the number
of modes. Result: **45–66% of states in the multi-human data genuinely carry two or more valid
actions** (the control: 28%). The modes are real.

![Fraction of multi-modal states per dataset](/images/blog/mmc/e0_kstar_fraction.png)

Here is what that looks like at a single state — two cleanly separated action clusters in the
multi-human data, one blob in the control:

![Action clusters at one state](/images/blog/mmc/method_step2_pca.png)

**Policies.** The same U-Net architecture and training recipe (400k steps, state observations),
trained twice per task: once with the DDPM diffusion objective, once with flow matching. Only
the generative objective differs. Sanity check: my flow success rates reproduce the paper's own
numbers task-for-task.

## The two evaluation tests (theirs, unchanged)

**Test 1 — does averaging hurt?** Roll out the *same* checkpoint three ways: noise frozen at
zero, noise sampled normally, and the *mean* of 32 sampled actions. If the policy really holds
several modes, averaging should blend them and success should drop. Pan et al.'s largest gap:
**0.04**.

**Test 2 — do sampled actions form modes?** At the most ambiguous state, sample many actions,
plot them in 2D, and colour each by its Monte-Carlo return (did continuing from this action
succeed?). A multi-modal policy shows several separated clusters that are *all* high-return.
Pan et al. found none, anywhere.

## Results

**Test 1: the invariance breaks on multi-modal data — for both families.** On the control task
the gap is 0.00, exactly reproducing the paper. But on `square` the gap opens to **0.30
(diffusion) / 0.20 (flow)**, and on `transport` to 0.13 / 0.25. The noise channel clearly
carries task-relevant modes once the data actually has them.

![Success under three sampling strategies](/images/blog/mmc/sampling_grid.png)

**Test 2: distinct high-return modes appear.** On `square`, diffusion's samples split into two
separated high-return clusters — flow's don't (below, left vs right). On `transport`, both
families show modes. This was the one real diffusion-vs-flow difference I found.

| Diffusion on `square` — two modes | Flow on `square` — one mode |
|---|---|
| ![Diffusion samples](/images/blog/mmc/pan_tsneq_square_mh_diffusion_s0.png) | ![Flow samples](/images/blog/mmc/pan_tsneq_square_mh_flow_s0.png) |

**The twist: capturing modes buys nothing.** Where diffusion captures a mode flow misses
(`square`), the two tie on success (0.90 each). On `transport`, flow *wins* (0.55 vs 0.30)
even though both show modes.

## Takeaway

- Pan et al.'s null result reproduces perfectly on low-multimodality data — but it's a property
  of their **datasets**, not of generative policies. Give the policies multi-modal data and both
  diffusion and flow visibly capture modes.
- Their broader point survives, and gets sharper: even when diffusion captures modes that flow
  drops, **it doesn't translate into better task performance**. The choice of generative
  objective still doesn't seem to be what matters.

*Caveats: single seed, 40 evaluation episodes per cell, state-based observations. Data:
the public [RoboMimic](https://robomimic.github.io) multi-human suite
([HF mirror](https://huggingface.co/datasets/ChaoyiPan/mip-dataset)).*
