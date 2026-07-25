---
layout: single
title: "Stable and Unstable Moments in Robot Demonstration Data"
permalink: /blog/stability-regimes/
author_profile: true
---

## The curse of horizon

A policy trained by imitation makes a small error at every step, and the errors do not average out: each one drifts the robot toward states the demonstrations cover less well, where the next error is larger. The classical bound says the total cost of this compounding grows with the *square* of the task length ([Ross et al., 2011](https://arxiv.org/abs/1011.0686)), and in continuous control it is worse: even under exponentially stable dynamics, a smooth imitator's execution error can grow *exponentially* with the horizon ([Simchowitz et al., 2025](https://arxiv.org/abs/2503.09722)). Manipulation demonstrations run hundreds of steps, so compounding, not per-step accuracy, is the failure mode that matters.

Whether an error compounds, though, is not decided by the policy alone. It is decided by the dynamics the error passes through, and those dynamics come in two versions depending on who is allowed to respond.

## Open-loop and closed-loop stability

Suppose the robot's hand is nudged a few millimeters off course at one instant of a demonstration. **Open-loop stability** is the nobody-reacts version of what happens next: the recorded commands keep playing unchanged, and physics alone decides whether the error fades or grows. A peg entering a chamfered hole gets guided back on course; a gripper closing on the thin edge of a part turns the same nudge into a slip. **Closed-loop stability** is the same question with the trained policy allowed to look and correct at every step, so it measures the robot and its feedback law together. Correcting sounds like it should always help, but a correction computed from a slightly-off observation is itself slightly off, so reacting can shrink the error or feed it. Which way it goes is an empirical property of that policy at that state.

The reason to care is a design choice inside modern imitation policies. [ACT](https://arxiv.org/abs/2304.13705) and [Diffusion Policy](https://arxiv.org/abs/2303.04137) do not decide one action at a time; they predict a chunk of $k$ actions and execute the whole block blindly before observing again. Small $k$ means reacting often, large $k$ means committing. The right setting at any moment depends exactly on the two stabilities above, and it is not constant: a demonstration that carries an object across free space and then seats it into a tight fixture wants opposite settings, seconds apart, in the same episode.

## The possible combinations

At each moment, each channel is stable or unstable, and our measurement (defined below) also reports a *deadband* when it cannot certify either direction. Three classes on two channels gives nine combinations, and two rules fix every cell: the open-loop class decides whether committing is safe, and the closed-loop class decides whether reacting helps or hurts.

|                 | CL stable         | CL deadband       | CL unstable        |
|-----------------|-------------------|-------------------|--------------------|
| **OL stable**   | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL deadband** | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL unstable** | replan every step | short $k$       | intermediate $k$ |

The interesting variation is confined to the open-loop-unstable row. Everywhere else committing wins: when physics absorbs errors, or contributes none of its own, the policy is the only error source present, and every extra replan is one extra mistake injected into a plant that was doing fine. In the bottom row physics amplifies errors, so replanning at every step is right where feedback genuinely rescues (bottom left), and when both channels expand (bottom right) the best available compromise is an intermediate $k$ that keeps some reaction while injecting less policy noise.

## Two ways out of the curse

The two labels matter because each names a funnel that removes the horizon from the compounding bound of the opening section, and they call for opposite chunk lengths. Call the size of a single policy mistake $\varepsilon$; it is the $\sigma_u$ measured below.

If a stretch is **open-loop stable**, the plant itself is the funnel. Commit a long chunk and do not react: the error injected at a replan has decayed by a factor $e^{-\lvert\lambda\rvert k}$ by the time the next replan arrives, so each new mistake lands on the shrunken remains of the last one and the total deviation stays near $\varepsilon$ however long the stretch runs. The horizon has dropped out of the bound. Reacting here is worse than unnecessary: it injects a fresh $\varepsilon$ into states that physics was already cleaning up.

If a stretch is **open-loop unstable but closed-loop stable**, the funnel is made of corrections instead of physics. Committing is what fails, because $k$ blind steps let the error grow to $\varepsilon\, e^{\lambda k}$, exponential in the chunk length. Replan every step instead: if each correction removes a fixed fraction of the error, leaving $\rho < 1$ of it, the deviation settles near $\varepsilon / (1 - \rho)$. The horizon has dropped out again, through the other channel.

![The two funnels: plant contraction under commitment, feedback contraction under replanning](/images/blog/stability/fig_funnel.png)

With neither funnel available nothing contracts, mistakes pile up, and the quadratic worst case is what remains. Read this way, the curse is not a law of imitation learning. It is what happens when the execution mode ignores the regime. Each moment of a demonstration offers at most one funnel, and the chunk length is the dial that selects it: long $k$ harvests the plant's funnel, $k = 1$ harvests the feedback funnel. The labels say which funnel exists at each timestep.

That is what sent us to the data. We labeled sixteen open-source datasets stamp by stamp, the open-loop channel on all of them and the closed-loop channel on the four robomimic tasks, to find out how often a demonstration switches regime and where the mass of these datasets actually sits.

## The algorithm

One measurement answers one question: if an error of realistic size were injected at time $t_0$ of this demonstration, would it grow or shrink over the next 1.2 seconds? Four steps.

First, reset the simulator to the exact recorded state at $t_0$. This is the step that needs a simulator; there is no rewind button on the real world.

Second, run the next $K = 24$ steps (1.2 seconds at 20 Hz) two ways. The nominal branch replays the recorded actions $a_{t_0}, \ldots, a_{t_0 + K - 1}$ unchanged. The perturbed branch adds Gaussian noise to the position components of the *first* action only, never rotation and never the gripper, then replays the rest exactly as recorded:

$$
\tilde{a}_{t_0} = a_{t_0} + \varepsilon, \qquad \varepsilon \sim \mathcal{N}\left(0, \sigma_u^2 I_3\right)
$$

The noise scale $\sigma_u$ is the measured action error of a trained policy (details below), so the injected mistake is the size a deployed policy actually makes. Because it enters through the action, it reaches the state the way a real policy error does: by passing through the plant.

Third, at every step $\tau$ after the nudge, record the distance between the two branches in task space:

$$
d_n(\tau) = \left\lVert x^{\mathrm{pert}}_n(\tau) - x^{\mathrm{nom}}(\tau) \right\rVert_2
$$

One nudge could be lucky, so repeat with $N = 8$ independent nudges and average the log of the gap: $\ell(\tau) = \frac{1}{N} \sum_{n} \log d_n(\tau)$.

Fourth, fit a slope. If the moment amplifies errors the gap grows exponentially, which makes $\ell(\tau)$ a straight rising line; if it absorbs them, a falling one. So the answer is the least-squares slope of $\ell(\tau)$ against time:

$$
\lambda(t_0) = \frac{\sum_{\tau=1}^{K} \left(\tau - \bar{\tau}\right)\left(\ell(\tau) - \bar{\ell}\right)}{\sum_{\tau=1}^{K} \left(\tau - \bar{\tau}\right)^2}
$$

This slope is a finite-time Lyapunov exponent, estimated the textbook way (Benettin et al., 1980). Finite-time because a manipulation episode is a sequence of short phases with no steady state to settle into.

**Reading the answer.** $\lambda > 0$ means the error was amplified, $\lambda < 0$ means it was absorbed. We call one measured timestep of one demonstration a *stamp*, and label it against a threshold $\delta$: **unstable** above $+\delta$, **stable** below $-\delta$, **deadband** in between, meaning the slope sits inside the probe's own noise and the stamp is unproven either way. $\delta$ is not a tuned constant. It is the 95th percentile of $\lvert \lambda \rvert$ over free-space stamps of the same dataset, the measured noise floor of the probe itself, computed separately for every dataset.

**The closed-loop version.** Same four steps, one substitution in step 2: after the identical nudge, the perturbed branch is no longer replay. The trained policy $\pi$ picks a fresh action from its own camera observations at every step,

$$
\tilde{a}_{t_0 + \tau} = \pi\left(o_{t_0 + \tau}\right), \qquad \tau = 1, \ldots, K-1
$$

and everything else, the injection, the window, the slope, the threshold, is held identical. The two channels therefore differ only in what happens after the error is introduced, which is exactly the difference the two labels are supposed to capture.

## The datasets

We labeled the four robomimic tasks (lift, can, square, tool_hang; 200 human demonstrations each), two MimicGen datasets (stack and square variants), and all ten LIBERO-Long files. They cover the difficulty range of benchmark manipulation, from lift, where a cube is picked off an open table, to tool_hang, where a frame is threaded onto a thin stand at millimeter tolerance. The result is a map, available before any policy is evaluated, of how much of each dataset is free transport and exactly where its demonstrations are fragile.

## Running the probe

$\lambda$ is measured every second timestep of every demonstration, across hundreds of demonstrations per dataset. Deadband stamps inherit the label of their confident neighbors, a median filter over about three stamps smooths the sequence, and contiguous runs of one label are the segments. Nothing is segmented by hand.

Each stamp looks $K$ steps ahead, so a window that straddles a regime boundary blends both regimes, produces a small slope, and lands in the deadband to be resolved by its neighbors. The practical effect is that segment edges arrive slightly early on the way into an unstable stretch, which is the right direction for a safety label to err: committing a chunk just before contact really is risky, because the contact falls inside the commitment window.

A trained policy enters in exactly two places. Its validation action error sets $\sigma_u$, so the probe injects the size of mistake a deployed policy actually makes (we trained diffusion policies to 0.9 to 1.0 success first), and the closed-loop channel puts it in the loop. Policy rollouts play no part in labeling.

The raw measurement on the square task looks like this. Thin gray lines are the $N = 8$ perturbed branches, blue is their average, and the dashed orange line is the least-squares fit whose slope is $\lambda(t_0)$; the dotted horizontal line marks the injected magnitude $\sigma_u$. The vertical axis is logarithmic, so exponential growth appears straight:

| Stable moment | Unstable moment |
| :---: | :---: |
| ![Divergence curve at a stable timestep](/images/blog/stability/curve_square_stable_0.png) | ![Divergence curve at an unstable timestep](/images/blog/stability/curve_square_unstable_0.png) |

The corresponding scenes: a transport moment on the left (deadband), the gripper engaging the nut on the right (unstable):

| Deadband moment | Unstable moment |
| :---: | :---: |
| ![Frame at a deadband timestep](/images/blog/stability/frame_square_gray.png) | ![Frame at an unstable timestep](/images/blog/stability/frame_square_red.png) |

The point of the exercise is that this label moves as the demonstration does. Replaying a demo with its border tinted by the label active at each timestep makes the structure visible directly: red where the measured slope exceeds the deadband, green where it is confidently below, gray inside it, with the running value of $\lambda$ printed along the top.

<div style="display:flex; gap:1.2em; flex-wrap:wrap; justify-content:center; margin:1.5em 0;">
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_square_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_square_labeled.mp4">Download the square label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>square</b> (robomimic): approach is deadband, the border goes red as the gripper closes on the nut, back to deadband while carrying it, red again as the nut is lowered onto the peg.</figcaption>
  </figure>
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_can_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_can_labeled.mp4">Download the can label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>can</b> (robomimic): a mostly-transport task. The long reach and the carry are both deadband; red appears only at the grasp and at the release into the bin.</figcaption>
  </figure>
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_libero_stove_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_libero_stove_labeled.mp4">Download the LIBERO stove label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>stove + moka pot</b> (LIBERO-Long): the one place green appears. The knob turn is mechanically guided, so the mechanism absorbs the injected error and the border turns green; red returns for the pot grasp and placement.</figcaption>
  </figure>
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_libero_mokapots_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_libero_mokapots_labeled.mp4">Download the LIBERO moka pots label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>both moka pots</b> (LIBERO-Long): a two-object task, so the whole cycle repeats. Deadband reach, red grasp, deadband carry, red set-down, then the same pattern again for the second pot.</figcaption>
  </figure>
</div>

A handful of demonstrations is not a statistic, and the proportions below come from hundreds of them rather than from these four. What the videos are meant to show is the shape of the signal: the labels arrive in contiguous stretches tied to contact events, they switch several times within a single demonstration, and the pattern repeats when the task repeats. That within-episode switching is the fact the rest of this post depends on.

## Results

**Most timesteps are neutral, and the unstable ones cluster.** In every suite a quarter to a third of timesteps are confidently unstable and almost all the rest are deadband. Confidently stable stamps are 0 to 4 percent, found essentially only where a mechanism guides the motion, as in the LIBERO stove task above. Can, the mostly-transport control, has the smallest unstable share:

![Regime proportions per dataset](/images/blog/stability/fig_regimes.png)

**The slopes are one-sided.** Every dataset has a median near zero, a short left tail, and a long right tail: moments are either roughly neutral or strongly divergent, rarely in between:

![Lambda distribution per dataset](/images/blog/stability/fig_lambda_dist.png)

**Labels form long bands, not scattered noise.** 82 to 94 percent of stamp mass sits in runs of six or more consecutive timesteps, and the bands sit on grasps, insertions, and handoffs, consistently across demonstrations of the same task. On tool_hang, the hardest task, 40 percent of stamps are unstable, aligned with the threading phases:

![Label timeline for tool_hang](/images/blog/stability/timeline_tool_hang.png)

**Reacting usually makes divergence worse.** On the same states where fixed playback absorbs the injected error, letting the trained policy react at every step makes the divergence positive at 90 to 99 percent of timesteps (99.3 on lift, 90.4 on can, 93.0 on square, 97.6 on tool_hang). Open loop (blue) sits at zero; closed loop (magenta) shifts positive almost everywhere:

![Open-loop versus closed-loop divergence on lift](/images/blog/stability/fig_ol_vs_cl_lift.png)

These policies are at 0.9 to 1.0 success, so this is not policy quality. Where physics absorbs errors the policy is the only error source present, and every reaction injects one more sample of it. In the table's terms: the rescue cell (open-loop unstable, closed-loop stable) is rare, and the cell where physics absorbs the error while reacting harms is the most common state in these datasets.

**Summary.** The stability regime is not a property of a task; it changes within a demonstration, several times, in long bands tied to contact events. Instability concentrates in those bands, which the probe finds automatically. Outside them, the robot's own corrections are the dominant noise source. The map is cheap to compute and its primary channel needs no trained policy.

*The probe constants, per-suite determinism checks, and full per-dataset statistics are in the project's technical report; this post kept only what is needed to read the figures.*
