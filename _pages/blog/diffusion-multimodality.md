---
layout: single
title: "Does Diffusion Actually Capture Multi-Modal Behavior? Re-Running a Myth-Buster's Tests"
permalink: /blog/diffusion-multimodality/
author_profile: true
---

*I also wrote this up as a thread on X — [read it here](https://x.com/arygoy/status/2074891040184795258?s=20).*

One of the reasons GCPs (generative control policies, like flow and diffusion models) were
considered better than RCPs (regression control policies) at task success is that they show
multi-modal behavior: when the demonstrations contain several valid actions at the same
situation, a generative model is theoretically capable of learning that full distribution,
while regression averages the choices into a useless in-between action. However,
[*Much Ado About Noising* (Pan et al.)](https://arxiv.org/abs/2512.01809) suggests this myth
is busted, and provides evidence that GCPs in practice also collapse to a single mode — the
**flow** policies they test behave essentially unimodally on the standard benchmarks — hence
multi-modality is not the reason they win. In this blog post we perform the same evaluation,
but on **diffusion** rather than flow, to check whether diffusion policies also collapse to a
single mode, or whether there is a change in task success and multi-modality.

## Setup

We keep everything the same as the paper's evaluation scheme and change only the model under
test: the same U-Net architecture, the same data and training recipe (400k steps, state
observations), trained once with the DDPM **diffusion** objective and once with **flow
matching** — only the training target equation differs. Every evaluation below is run
identically on both.

**How Pan et al. evaluate a learned policy's multi-modality** — three methods:

1. **Sampling-strategy test.** Roll out the *same* trained checkpoint three ways: noise frozen
   at zero, noise sampled randomly, and the *average* of many sampled actions. If the policy
   truly holds several modes, averaging blends them into an invalid action and success should
   drop; if success barely changes (their largest gap was 0.04), the noise channel carries no
   real modes.
2. **Return-colored sample plot.** At an ambiguous state, sample many action chunks, embed
   them in 2D, and color each by its Monte-Carlo return (did continuing from this action
   succeed?). A multi-modal policy shows several separated clusters that are *all*
   high-return. They found none.
3. **Deterministic-expert control.** Retrain on demonstrations from a scripted, deterministic
   expert (no multi-modality in the data at all) — if multi-modality explained the GCP
   advantage, it should vanish there. It didn't. (We don't re-run this one; our near-unimodal
   control dataset plays the same role.)

## First: is the data itself multi-modal?

A policy can only learn multi-modal behavior if the **demonstration data is multi-modal in
the first place** — this is the paper's own caveat about its benchmarks. So before evaluating
any policy, we check the data. We use the RoboMimic **multi-human** datasets (`can`, `square`,
`transport` — six different operators demonstrating each task) plus `lift` from a single
proficient expert as the control.

**The test:** for each state, collect the K = 32 most similar states across all
demonstrations and gather the action chunks that were actually taken there. Then run k-means
on those actions for k = 1, 2, 3, … and watch how much of the action variance each extra
cluster removes. One blob of actions barely splits — the variance ratio stays near 1. Two
(or three) genuinely different behaviors split cleanly — the ratio drops sharply, then
flattens. The elbow of that curve is the number of modes at that state; a state is
multi-modal if it has 2 or more.

The variance-drop curve at one state of each kind — the multi-human state has a sharp elbow
at k = 2 (two real modes), the control state has none:

![Variance-reduction elbow](/images/blog/mmc/method_step2_ratio.png)

Aggregating over 5000 states per dataset: **45–66% of states in the multi-human datasets are
multi-modal, versus 28% in the control** — and a quarter of the multi-modal states in the
hardest sets carry three modes:

![Fraction of multi-modal vs unimodal states per dataset](/images/blog/mmc/e0_kstar_fraction.png)

So the modes are really in the data, and any collapse we see next is the policy's doing.

## Results: the policies

**Test 1 — sampling strategies.** On the control task the gap is 0.00, reproducing the
paper's null exactly. On the multi-human data the invariance breaks for *both* families:
averaging or freezing the noise changes success by up to 0.30 (diffusion) / 0.20 (flow) on
`square`. Once the data has modes, the noise channel carries them.

![Success under three sampling strategies](/images/blog/mmc/sampling_grid.png)

**Test 2 — return-colored samples.** Distinct high-return modes *do* appear. On `square`,
diffusion's 48 samples concentrate onto a few clearly separated actions — two distinct
successful behaviors — while flow's 48 samples are near-copies of a single action (note the
axis scales: flow's whole cloud spans ~100× less). On `transport`, both families show modes.

| Diffusion on `square` — two modes | Flow on `square` — one mode |
|---|---|
| ![Diffusion samples](/images/blog/mmc/pan_tsneq_square_mh_diffusion_s0.png) | ![Flow samples](/images/blog/mmc/pan_tsneq_square_mh_flow_s0.png) |

**So: does diffusion collapse like flow?** Not here — and flow itself doesn't fully collapse
on this data either. But the extra mode diffusion captures buys nothing: on `square` the two
families tie on success (0.90 each), and on the hardest task (`transport`) flow *wins*
(0.55 vs 0.30) even though both show modes.

## Takeaway

- Pan et al.'s null reproduces perfectly on low-multimodality data — it is a property of
  their **datasets**, not of generative policies. Given genuinely multi-modal demonstrations,
  both diffusion and flow visibly capture modes.
- Their broader conclusion survives and sharpens: even where diffusion captures a mode that
  flow drops, **it does not translate into better task success**. The choice of generative
  objective still doesn't seem to be what matters.

*Caveats: single seed, 40 evaluation episodes per cell, state-based observations. Data:
the public [RoboMimic](https://robomimic.github.io) multi-human suite
([HF mirror](https://huggingface.co/datasets/ChaoyiPan/mip-dataset)).*
