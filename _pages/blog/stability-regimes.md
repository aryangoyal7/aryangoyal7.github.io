---
layout: single
title: "Stable and Unstable Moments in Robot Demonstration Data"
permalink: /blog/stability-regimes/
author_profile: true
---

## The curse of horizon

A policy trained by imitation makes a small error at every step, and the errors do not cancel out. Each one pushes the robot slightly away from the situations the demonstrations covered, where the policy is less sure of itself, so the next error is bigger. Over a long task this compounding, not any single mistake, is what fails. The classical result says the total cost grows with the *square* of the task length ([Ross et al., 2011](https://arxiv.org/abs/1011.0686)), and newer theory shows that in continuous control it can even grow *exponentially* with it ([Simchowitz et al., 2025](https://arxiv.org/abs/2503.09722)).

But whether an error compounds is not decided by the policy alone. It is decided by what the world and the controller do with the error next. That is what stability means here, and it comes in two versions.

## Open-loop and closed-loop stability

Suppose the robot's hand is nudged a few millimeters off course at one instant of a demonstration.

**Open-loop stability** is the nobody-reacts version of what happens next. The recorded commands keep playing unchanged, and physics alone decides what the nudge becomes. Sometimes the scene corrects it for free: a peg sliding into a chamfered hole is guided back on course. Sometimes the scene amplifies it: a gripper closing on the thin edge of a part turns the same nudge into a slip. Open-loop stable means physics absorbs the error. Open-loop unstable means physics grows it.

**Closed-loop stability** asks the same question when the trained policy is allowed to look and correct at every step, so it measures the robot and its feedback together. Correcting sounds like it should always help, but each correction is computed from a slightly wrong observation, so it is itself slightly wrong. Closed-loop stable means the corrections genuinely pull the error down. Closed-loop unstable means they add more error than they remove.

The link back to compounding is direct: an error only compounds if nothing absorbs it, and at any moment there are exactly two candidate absorbers, the physics and the policy's feedback. The two labels tell you which of them, if either, is working right now.

They also tell you how to act, through one specific knob. Modern imitation policies like [ACT](https://arxiv.org/abs/2304.13705) and [Diffusion Policy](https://arxiv.org/abs/2303.04137) predict a chunk of $k$ actions and execute the whole chunk blindly before observing again. Small $k$ means react often; large $k$ means commit. And one $k$ per task cannot be right, because a demonstration that carries an object through free space and then seats it into a tight fixture wants opposite settings, seconds apart.

## The combinations, and what to do in each

Our probe reports each channel as stable, unstable, or *deadband*, which means the measurement cannot certify either direction. Two channels with three answers each gives nine combinations, and two rules cover all of them: the open-loop answer says whether committing is safe, and the closed-loop answer says whether reacting helps.

|                 | CL stable         | CL deadband       | CL unstable        |
|-----------------|-------------------|-------------------|--------------------|
| **OL stable**   | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL deadband** | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL unstable** | replan every step | short $k$       | intermediate $k$ |

In the top two rows physics either absorbs the error or adds none of its own, so the policy is the only remaining error source, and every extra replan is one extra mistake injected into a plant that was doing fine. Commit. In the bottom row physics grows the error, so it depends on the policy: replan every step where its feedback genuinely rescues, keep chunks short where the feedback is unproven, and when reacting also hurts, no mode contains the error and an intermediate $k$ is the least bad compromise.

## Two ways out of the curse

The two clean cells of that table are both funnels: places where compounding simply stops. Call the size of one policy mistake $\varepsilon$.

If a stretch is open-loop stable, physics is the funnel, so commit. During a committed chunk the plant keeps shrinking the error, by a factor $e^{-\lvert\lambda\rvert k}$ over $k$ steps, so by the time the next replan arrives the previous mistake has mostly dissolved and mistakes never stack. The total deviation stays around $\varepsilon$ no matter how long the task runs. The horizon has dropped out. Reacting here would only inject fresh error into states physics was already cleaning up.

If a stretch is open-loop unstable but closed-loop stable, feedback is the funnel, so replan. Committing is what fails there, because $k$ blind steps let the error grow like $\varepsilon\, e^{\lambda k}$. Correct at every step instead, with each correction removing a fixed fraction of the error, and the deviation settles near $\varepsilon / (1 - \rho)$, a constant. The horizon has dropped out again, through the other channel.

![The two funnels: plant contraction under commitment, feedback contraction under replanning](/images/blog/stability/fig_funnel.png)

With neither funnel nothing contracts, and the compounding of the opening section is what remains. Read this way, the curse is not a law of imitation learning. It is what happens when the execution mode ignores the regime. The chunk length is the dial that picks a funnel: long $k$ harvests the physics funnel, $k = 1$ harvests the feedback funnel.

So the practical question is whether real demonstrations actually contain different regimes, and where they sit: at contact events, in free motion, or nowhere. We labeled sixteen open-source datasets, timestep by timestep, to check.

## The algorithm

One measurement answers one question: if a realistic error were injected right here, would it grow or shrink over the next 1.2 seconds? Four steps.

First, reset the simulator to the exact recorded state at time $t_0$. This is the step that needs a simulator; the real world has no rewind button.

Second, play the next $K = 24$ steps (1.2 seconds) twice. The nominal run replays the recorded actions unchanged. The perturbed run adds a small random nudge to the position part of the first action only, then replays the rest unchanged:

$$
\tilde{a}_{t_0} = a_{t_0} + \varepsilon, \qquad \varepsilon \sim \mathcal{N}\left(0, \sigma_u^2 I_3\right)
$$

The nudge size $\sigma_u$ is the measured action error of a trained policy, so we inject exactly the size of mistake a deployed policy makes.

Third, at every step after the nudge, record the gap between the two runs:

$$
d(\tau) = \left\lVert x^{\mathrm{pert}}(\tau) - x^{\mathrm{nom}}(\tau) \right\rVert_2
$$

One nudge could be lucky, so repeat with eight independent nudges and average the log of the gap.

Fourth, look at how that log gap moves across the window. If it climbs, errors amplify here; if it falls, they are absorbed. Its fitted slope is the stability number $\lambda$ (the standard finite-time Lyapunov exponent estimate, Benettin et al., 1980): positive means unstable, negative means stable.

One honesty rule on top: the probe has its own noise. We measure that noise on free-space moments of the same dataset, and any slope smaller than it is labeled deadband, meaning cannot tell. So every measured timestep ends up unstable, stable, or deadband.

The closed-loop label repeats the same four steps with one change. After the identical nudge, the perturbed run is no longer a replay: the trained policy picks a fresh action from its own camera view at every step, $\tilde{a}_{t_0+\tau} = \pi(o_{t_0+\tau})$. The two channels therefore differ only in what happens after the error appears, which is exactly the difference the two labels are meant to capture.

## The datasets

We labeled the four robomimic tasks (lift, can, square, tool_hang; 200 human demonstrations each), two MimicGen variants (stack and square), and all ten LIBERO-Long tasks. Sixteen datasets, every second timestep of every demonstration, the open-loop channel on all of them and the closed-loop channel on the four robomimic tasks. Deadband timesteps take the label of their confident neighbors and a small median filter smooths the sequence, so labels group into segments on their own; nothing is segmented by hand.

## What the label looks like

The raw measurement at one moment of the square task. Thin gray lines are the eight perturbed runs, blue is their average, and the dashed line is the fitted slope $\lambda$. The vertical axis is logarithmic, so exponential growth appears straight. At a stable moment the gap falls; at an unstable one it climbs:

| Stable moment | Unstable moment |
| :---: | :---: |
| ![Divergence curve at a stable timestep](/images/blog/stability/curve_square_stable_0.png) | ![Divergence curve at an unstable timestep](/images/blog/stability/curve_square_unstable_0.png) |

The scenes behind those two measurements: carrying the nut through free space on the left (deadband), the gripper engaging it on the right (unstable):

| Deadband moment | Unstable moment |
| :---: | :---: |
| ![Frame at a deadband timestep](/images/blog/stability/frame_square_gray.png) | ![Frame at an unstable timestep](/images/blog/stability/frame_square_red.png) |

And the label moves as the demonstration moves. In the videos the border color is the live label: red unstable, green stable, gray deadband, with the running $\lambda$ printed on top:

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
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>can</b> (robomimic): a mostly-transport task. The long reach and the carry stay deadband; red appears only at the grasp and at the release into the bin.</figcaption>
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

Four videos are examples, not evidence; the numbers below come from hundreds of demonstrations. What the videos show is the shape of the signal: the label switches several times inside a single demonstration, the switches sit on contact events, and the pattern repeats when the task repeats.

## Results

**Different regimes do exist inside a single task.** The first graph shows the share of each label per dataset. In every one of the sixteen, a quarter to a third of timesteps are confidently unstable and almost all the rest are deadband, so every task mixes fragile moments into mostly neutral motion. Confidently stable moments are rare everywhere (0 to 4 percent), essentially only the mechanically guided stove knob. Can, our transport-heavy control task, has the smallest unstable share, as it should:

![Regime proportions per dataset](/images/blog/stability/fig_regimes.png)

**Instability is the strong signal.** The histogram of the slopes is one-sided: most moments sit near zero, and the long tail is on the positive side only. In these datasets a moment is either roughly neutral or clearly divergent, rarely strongly self-correcting:

![Lambda distribution per dataset](/images/blog/stability/fig_lambda_dist.png)

**The unstable moments sit exactly where you would expect: at contact.** The timeline below stacks tool_hang demonstrations row by row. The labels come in long bands, not scattered dots (82 to 94 percent of label mass sits in runs of six or more consecutive timesteps), and the red bands line up with the grasps, insertions, and handoffs across demonstrations. On tool_hang, the hardest task, 40 percent of timesteps are unstable:

![Label timeline for tool_hang](/images/blog/stability/timeline_tool_hang.png)

**Letting the policy react usually makes things worse.** The last graph compares the same lift states under the same nudge, with and without the policy correcting. Pure replay (blue) sits at zero: physics absorbs the nudge. With the policy correcting at every step (magenta), divergence turns positive at 90 to 99 percent of timesteps across the four tasks. These policies succeed 90 to 100 percent of the time, so this is not a bad-policy artifact. Where physics already absorbs errors, the policy is the only error source left, and every correction adds one more sample of it:

![Open-loop versus closed-loop divergence on lift](/images/blog/stability/fig_ol_vs_cl_lift.png)

**Summary.** The stability regime is not a property of a task. It changes several times within a single demonstration, in long bands that sit on contact events: grasps, insertions, handoffs. Free motion is neutral ground where the robot's own corrections are the main source of error. The map is cheap to compute, and its primary channel needs no trained policy at all.

*The probe constants, per-suite determinism checks, and full per-dataset statistics are in the project's technical report; this post kept only what is needed to read the figures.*
