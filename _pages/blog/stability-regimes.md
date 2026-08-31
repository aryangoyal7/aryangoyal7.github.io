---
layout: single
title: "Stable and Unstable Moments in Robot Demonstration Data"
permalink: /blog/stability-regimes/
author_profile: true
---

## The curse of horizon

Unlike the symbolic world, where the agent is still in the distribution after predicting the next wrong token, in robotic control the agent can move away from its distribution as its error compounds along the horizon. The classical result says the total cost grows with the *square* of the task length ([Ross et al., 2011](https://arxiv.org/abs/1011.0686)), and newer theory shows that in continuous control it can even grow *exponentially* with it ([Simchowitz et al., 2025](https://arxiv.org/abs/2503.09722)).

This error compounding is not only a function of the trained policy of the agent, but also of the dynamics of the world, which comprise the environment around it and the robotic controller. The error at the starting point could either die down as it moves along the horizon, or compound with the horizon.

## Open-loop and closed-loop stability


**Open-loop stability** is a situation where, once we observe an error, the error somehow funnels out with the horizon and is absorbed by the dynamics. This is an ideal scenario where the error is eliminated without any property of the policy.

**Closed-loop stability** refers to a situation where the agent makes an error and goes off track. You need feedback from the policy to nudge it in the correct direction, such that it goes back to its original track and the error dies out via the feedback from the policy.


Modern imitation policies like [ACT](https://arxiv.org/abs/2304.13705) and [Diffusion Policy](https://arxiv.org/abs/2303.04137) have the option of either predicting a chunk of $k$ actions and executing them at once, or taking one action and then asking the policy to compute again and predict the next action. This choice, however, should depend upon the stability regime.

## The combinations, and what to do in each

There are a few combinations possible for these regimes before we label the datasets and check. Let's see which combinations are possible.

|                 | CL stable         | CL deadband       | CL unstable        |
|-----------------|-------------------|-------------------|--------------------|
| **OL stable**   | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL deadband** | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL unstable** | replan every step | short $k$       | intermediate $k$ |

Open-loop stable means the physics of the plant and the dynamic environment will absorb the error. Closed-loop stable means the policy is performing well enough to correct the errors and put the agent back in the right direction. However, we might also see some deadband regions where we are unable to comment on the stability.

## Two ways out of the curse

The table gives us two ways to stop error compounding, and both of them work like a funnel. Let's say the size of one policy mistake is $\varepsilon$.

If a stretch of the task is open-loop stable, physics is the funnel, so we should commit to a long chunk. During the chunk, the plant keeps shrinking the error by a factor of $e^{-\lvert\lambda\rvert k}$ over $k$ steps. By the time the next replan arrives, the previous mistake has mostly died out, so mistakes never stack up. The total deviation stays around $\varepsilon$ no matter how long the task is, and the horizon does not matter anymore. Reacting here would only inject fresh error into a system that was already correcting itself.

If a stretch is open-loop unstable but closed-loop stable, feedback is the funnel, so we should replan at every step. Committing fails here, because $k$ blind steps let the error grow like $\varepsilon\, e^{\lambda k}$. If we correct at every step instead, each correction removes a fixed fraction of the error, and the deviation settles near a constant $\varepsilon / (1 - \rho)$. Again, the horizon does not matter anymore.

![The two funnels: plant contraction under commitment, feedback contraction under replanning](/images/blog/stability/fig_funnel.png)

If neither funnel exists, nothing shrinks the error, and we are back to the compounding from the first section. Seen this way, the curse of horizon is not a law of imitation learning. It is what happens when the execution mode ignores the stability regime. The chunk length is the dial that picks the funnel: a long chunk uses the physics funnel, and a chunk length of one uses the feedback funnel.

So the practical question is: do real demonstrations actually contain different regimes, and where do they sit? At contact events, in free motion, or nowhere? We labeled twenty open-source datasets, timestep by timestep, to check this.

## The algorithm

Each measurement answers one simple question: if a realistic error is injected right here, will it grow or shrink over the next 1.2 seconds? There are four steps.

First, reset the simulator to the exact recorded state at time $t_0$. This step needs a simulator, because the real world has no rewind button.

Second, play the next $K = 24$ steps (1.2 seconds) twice. The nominal run replays the recorded actions without any change. The perturbed run adds a small random nudge to the position part of the first action only, and then replays the rest without any change:

$$
\tilde{a}_{t_0} = a_{t_0} + \varepsilon, \qquad \varepsilon \sim \mathcal{N}\left(0, \sigma_u^2 I_3\right)
$$

The nudge size $\sigma_u$ is the measured action error of a trained policy. So we inject exactly the size of mistake that a deployed policy makes.

Third, at every step after the nudge, record the gap between the two runs:

$$
d(\tau) = \left\lVert x^{\mathrm{pert}}(\tau) - x^{\mathrm{nom}}(\tau) \right\rVert_2
$$

One nudge could be lucky, so we repeat this with eight independent nudges. Each nudge gets its own fitted slope; their mean is the measurement, and their spread gives its standard error.

Fourth, look at how the log gap moves across the window. If it climbs, errors grow at this moment. If it falls, errors get absorbed. The fitted slope of this curve is our stability number $\lambda$ (the standard finite-time Lyapunov exponent estimate, Benettin et al., 1980). Positive means unstable, and negative means stable.

There are two honesty rules on top. The slope has to be large enough to matter: at least one doubling, or one halving, of the error over a 16-step chunk, $\lvert\lambda\rvert > \ln 2 / 16$. And the eight nudges have to agree on the sign: $\lvert\lambda\rvert$ must also exceed 2.365 times its own standard error, a 95 percent test. A timestep that clears both bars is unstable or stable by the sign of $\lambda$; anything else is labeled deadband, which means we cannot tell. So every measured timestep ends up as unstable, stable, or deadband.

The closed-loop label repeats the same four steps with one change. After the identical nudge, the perturbed run is not a replay anymore. The trained policy picks a fresh action from its own camera view at every step, $\tilde{a}_{t_0+\tau} = \pi(o_{t_0+\tau})$, and so does the nominal run. That raises a subtlety the first version of this post missed: a stochastic policy does not even reproduce its own trajectory, so two unperturbed runs drift apart at some rate $\lambda_{\mathrm{ctrl}}$ with no error injected at all. We therefore add two unperturbed control runs at every state and report the excess, $\lambda_{\mathrm{excess}} = \lambda_{\mathrm{pert}} - \lambda_{\mathrm{ctrl}}$. Both labels are measured at states the policy actually visits, so the two channels differ only in what happens after the error appears, and that is exactly the difference the two labels are meant to capture.

## The datasets

We labeled the four robomimic tasks (lift, can, square, tool_hang; 200 human demonstrations each), policy rollouts on lift and can, four MimicGen tasks (coffee, nut assembly, square, stack), and all ten LIBERO-Long tasks. That is twenty datasets in total, labeled at every second timestep of every demonstration; the LIBERO-Long ten keep the earlier single-bar labels, because their simulator turned out not to be repeatable from run to run. The closed-loop channel was run on eight tasks across three platforms: robomimic lift and square with our diffusion policies, MimicGen coffee and nut assembly, and four RoboCasa kitchen tasks with the GR00T N1.5 model, about 26,000 states, each measured both ways. Every timestep keeps its own label, and nothing is segmented by hand.

## What the label looks like

Here is the raw measurement at one moment of the square task. The thin gray lines are the eight perturbed runs, the blue line is their average, and the dashed line is the fitted slope $\lambda$. The vertical axis is logarithmic, so exponential growth looks like a straight line. At a stable moment the gap falls, and at an unstable moment it climbs:

| Stable moment | Unstable moment |
| :---: | :---: |
| ![Divergence curve at a stable timestep](/images/blog/stability/curve_square_stable_0.png) | ![Divergence curve at an unstable timestep](/images/blog/stability/curve_square_unstable_0.png) |

These are the scenes behind those two measurements. On the left, the robot is carrying the nut through free space (deadband). On the right, the gripper is engaging the nut (unstable):

| Deadband moment | Unstable moment |
| :---: | :---: |
| ![Frame at a deadband timestep](/images/blog/stability/frame_square_gray.png) | ![Frame at an unstable timestep](/images/blog/stability/frame_square_red.png) |

The label also moves as the demonstration moves. In the videos below, the border color is the live label: red is unstable, green is stable, and gray is deadband, with the running $\lambda$ printed on top:

<div style="display:flex; gap:1.2em; flex-wrap:wrap; justify-content:center; margin:1.5em 0;">
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_square_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_square_labeled.mp4">Download the square label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>square</b> (robomimic): approach is deadband, red as the gripper closes on the nut, deadband while carrying it, red again as the nut is lowered onto the peg.</figcaption>
  </figure>
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_can_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_can_labeled.mp4">Download the can label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>can</b> (robomimic): a mostly-transport task. The long reach and the carry stay deadband; red appears only at the grasp and at the release into the bin, and the retreat afterwards, with the can at rest, is one of the rare green stretches.</figcaption>
  </figure>
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_libero_stove_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_libero_stove_labeled.mp4">Download the LIBERO stove label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>stove + moka pot</b> (LIBERO-Long): the one place green appears. The knob turn is guided by the mechanism, which absorbs the nudge, so the border turns green; red returns for the pot grasp and placement.</figcaption>
  </figure>
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_libero_mokapots_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_libero_mokapots_labeled.mp4">Download the LIBERO moka pots label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>both moka pots</b> (LIBERO-Long): two objects, so the cycle repeats. Deadband reach, red grasp, deadband carry, red set-down, then the same pattern for the second pot.</figcaption>
  </figure>
</div>

The four videos are examples, not evidence. The numbers below come from hundreds of demonstrations. What the videos show is the shape of the signal: the label switches several times inside a single demonstration, the switches happen at contact events, and the pattern repeats when the task repeats.

## Results

**Different regimes do exist inside a single task.** The first graph shows the share of each label in every dataset. Across the ten two-bar datasets, 15 to 31 percent of the timesteps are confidently unstable, and most of the rest are deadband. So every task mixes fragile moments into mostly neutral motion. Confidently stable moments are rare everywhere (0 to 6 percent); the largest shares sit on the two MimicGen assembly tasks and on the mechanically guided stove knob of LIBERO. Can, our transport-heavy control task, has the smallest unstable share, which is what we expected:

![Regime proportions per dataset](/images/blog/stability/fig_regimes.png)

**Instability is the strong signal.** The graph of the slopes is one-sided. Most moments sit near zero, and the long tail is only on the positive side: the 95th percentile is between 0.10 and 0.18 per step in every dataset, while the 5th percentile never drops below $-0.05$. In these datasets, a moment is either roughly neutral or clearly divergent. It is rarely strongly self-correcting:

![Lambda distribution per dataset](/images/blog/stability/fig_lambda_dist.png)

**The unstable moments sit exactly where you would expect: at contact.** The left panel below stacks the tool_hang demonstrations row by row. Even with no smoothing at all, the labels come in bands, not scattered dots: on tool_hang and square, more than half of the unstable mass sits in runs of six or more consecutive stamps, and the red bands line up with the grasps, insertions, and handoffs across demonstrations. On tool_hang, the hardest task, 31 percent of the timesteps are unstable. The right panel is the closed-loop channel on square, one row per policy rollout. The same contact events are where the policy's feedback gets tested, and this is where blue, the stable label, finally shows up:

![Label timelines: tool_hang open loop, square closed loop](/images/blog/stability/timeline_tool_hang.png)

**Letting the policy react does not make things worse. Its own noise does.** The first version of this post compared policy-driven runs against fixed replay, saw a positive divergence at 90 to 99 percent of the timesteps, and concluded that every correction adds one more error. The control runs show why that reading was wrong. Two unperturbed runs of the same diffusion policy separate at a median rate of about 0.09 per step, roughly the rate we had blamed on the injected error: the raw number was sampling noise, not compounding. Once that floor is subtracted, the closed-loop score is centred on zero on every platform (medians $-0.009$ on RoboCasa, $-0.020$ on robomimic, $-0.002$ on MimicGen), and on robomimic it leans negative: the policy pulls the nudged run back toward its own path. The graph below shows the same robomimic states measured both ways. The open-loop channel is hot, because the planned chunk amplifies a nudge about four times over 16 steps, while the corrected closed-loop channel sits at zero:

![Open-loop versus closed-loop divergence on robomimic](/images/blog/stability/fig_ol_vs_cl.png)

**Where physics fails and feedback works.** With both channels measured at the same states, the table from the start of the post can finally be filled in. The cell that calls for replanning at every step, open-loop unstable and closed-loop stable, holds about 1 percent of RoboCasa states, 2 percent of MimicGen states, and 9 percent of robomimic states (4 percent on lift, 15 percent on square) at the primary bars, and roughly twice that with the looser bar of $m = 1.5$. The opposite cell, where physics absorbs the error but the policy makes it worse, is essentially empty: 0.0 to 0.4 percent everywhere. Most states sit in the deadband on the closed-loop side, which is the honest answer when the policy's own noise is as large as the effect being measured.

| | RoboCasa | MimicGen | robomimic | all |
|---|:---:|:---:|:---:|:---:|
| states measured both ways | 13,420 | 6,470 | 6,172 | 26,062 |
| OL unstable and CL stable ($m = 2$) | 1.0% | 2.0% | 9.3% | 3.2% |
| OL unstable and CL stable ($m = 1.5$) | 3.6% | 3.1% | 14.1% | 6.0% |
| OL stable and CL unstable ($m = 2$) | 0.4% | 0.3% | 0.0% | 0.3% |

**Summary.** The stability regime is not a property of a task. It changes several times within a single demonstration, in bands that sit on contact events: grasps, insertions, and handoffs. Free motion is neutral ground. The policy's corrections are not the villain we first took them for: once its sampling noise is measured and removed, feedback is neutral at most states and helpful at a minority of them, and that minority is largest exactly where the open-loop channel is hottest. The map is cheap to compute, and its primary channel does not need a trained policy at all.

*The probe constants, the determinism checks, and the full per-dataset statistics are in the project's technical report. This post kept only what is needed to read the figures.*
