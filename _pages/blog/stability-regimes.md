---
layout: single
title: "Open-Loop and Closed-Loop Stability Regimes in Open-Source Manipulation Datasets"
permalink: /blog/stability-regimes/
author_profile: true
---

## Open-loop and closed-loop stability

Consider a robot arm carrying a peg across a table. If its hand drifts two millimeters off course, nothing happens: it arrives two millimeters to the side and the task still succeeds. Now consider the same arm halfway through inserting that peg into a tight hole. The same two millimeters means the peg catches the rim and jams. Same robot, same error, different outcome. Whether an error is absorbed or amplified is a property of the physics at each moment, not of the robot or of the task as a whole.

Open-loop and closed-loop stability are the control-theoretic names for the two versions of that question, and it is worth being precise about them, because everything below turns on the difference.

**Open-loop stability** is a property of the plant alone. Hold the commands fixed to what the demonstration recorded, nudge the very first one slightly, and watch what the dynamics do to the resulting deviation over the next fraction of a second. If nearby trajectories are pulled back together, the moment contracts errors; if they are driven apart, it expands them. The rate of separation between two initially-close trajectories is a Lyapunov exponent, measured here over a finite window rather than asymptotically, because a manipulation episode is a sequence of short phases with no steady state to settle into. A negative rate means the error decays on its own; a positive one means it grows exponentially. No policy appears anywhere in this definition — it is a question about physics and contact geometry.

**Closed-loop stability** asks the same question with the controller put back in the loop. After the identical nudge, the robot looks at the scene and issues a fresh command at every step, so what is measured is the plant and the feedback law together. Correcting is supposed to shrink the error. Whether it does is an empirical property of that policy at that state, and it can go either way: a correction computed from a slightly-off observation is itself slightly off, and injects error of its own.

## Why this is worth measuring

A policy trained by imitation makes a small error at every step, and those errors do not stay small. Each one nudges the robot toward states that appear less often in the demonstrations, where the policy is less reliable and errs by more, which nudges it further still. The consequence is that the cost of behavior cloning grows with the *square* of the task horizon rather than linearly with it ([Ross et al., 2011](https://arxiv.org/abs/1011.0686)). Manipulation tasks are long — the tool_hang demonstrations used here run past six hundred steps — so this compounding, not per-step accuracy, is the dominant failure mode.

The prescribed escape is action chunking. Rather than predicting one action and re-deciding at every timestep, the policy predicts a block of $k$ actions and executes the whole block open-loop before observing again; [ACT](https://arxiv.org/abs/2304.13705) and [Diffusion Policy](https://arxiv.org/abs/2303.04137) both work this way. The horizon does not get any shorter, but the number of decision points along it falls from $T$ to $T/k$, and so does the number of chances to inject a fresh error. What is given up is reactivity: for the length of a chunk the robot is committed, and nothing that happens in the world can change what it does.

So the choice is between long chunks and short chunks, and it is normally made once per task and then held fixed. Long chunks suppress compounding but ride out any disturbance blindly; short chunks stay responsive but re-inject policy noise at every step. Which one is right depends on whether the physics at the current moment forgives a mistake or punishes it — and that is not constant along a demonstration. The transport phase and the insertion in the peg example want opposite settings, and they are a second apart in the same episode.

That is what sent us to the data. We labeled sixteen open-source datasets stamp by stamp, the open-loop channel on all of them and the closed-loop channel on the four robomimic tasks, to find out how often a demonstration switches regime and where the mass of these datasets actually sits.

## The possible combinations

Each channel reports one of three classes: stable, unstable, or **deadband**, meaning the measured slope sits inside the probe's own noise floor, so neither call is confident (the algorithm section explains how that floor is set). Three classes on two channels gives nine combinations.

What hangs on them is the chunk length $k$: $k = 1$ is replanning at every step, and large $k$ means committing to a long stretch of actions. Two rules then fix every cell. The open-loop class decides whether committing is safe, and the closed-loop class decides whether reacting helps or hurts.

|                 | CL stable         | CL deadband       | CL unstable        |
|-----------------|-------------------|-------------------|--------------------|
| **OL stable**   | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL deadband** | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL unstable** | replan every step | short $k$       | intermediate $k$ |

The interesting variation is confined to the open-loop-unstable row; everywhere else commitment wins. Taking the cells in turn: when physics absorbs the error but the policy's corrections add new error (top right), reacting is the harmful choice, so committing is not merely cheaper but safer. A deadband row behaves the same way and for a sharper reason — the plant contributes no error of its own there, so the policy is the only error source present, and each replan injects one fresh mistake that nothing absorbs; committing chunks of $k$ accumulates $T/k$ such mistakes over a $T$-step segment instead of $T$. In the bottom row, physics amplifies the error, and replanning at every step is right only where feedback genuinely rescues (bottom left). When both channels expand (bottom right), neither mode contains the error, success depends on the task geometry guiding the state back the way a hole guides a peg, and the best available compromise is an intermediate $k$ that keeps some reaction while injecting less policy noise.

The results below show which of these actually occur in the data, and in what proportion.

## The datasets

We labeled the four robomimic tasks (lift, can, square, tool_hang; 200 human demonstrations each), two MimicGen datasets (stack and square variants), and all ten LIBERO-Long files. They cover the difficulty range of benchmark manipulation, from lift, where a cube is picked off an open table, to tool_hang, where a frame is threaded onto a thin stand at millimeter tolerance.

The reason to label them: a stability map shows what kind of data a dataset contains, how much is transport where any policy coasts, how much is contact where errors compound and failures concentrate, and at exactly which timesteps demonstrations are fragile. All of this is available before any policy is evaluated on the data.

## The algorithm

For a given demonstration and timestep, reset the simulator to the exact recorded state and roll the next 1.2 seconds forward twice. The first run replays the demonstration's recorded actions exactly. The second replays the same actions, except a small Gaussian error is added to the position part of the first action. Track the distance between the two trajectories over the 24 control steps and fit the slope of its logarithm (a finite-time Lyapunov exponent). If the gap shrinks or stays flat, the timestep is open-loop stable; if it grows exponentially, open-loop unstable. We use 8 noise samples per timestep, a stamp every 2 steps, across hundreds of demonstrations per dataset.

The closed-loop probe repeats the same experiment with one change: after the identical injected error, the perturbed run is no longer fixed playback. A trained diffusion policy picks a fresh action from its own camera observations at every step. We trained and verified these policies first (lift and can at 1.0 success, square and tool_hang at 0.9). The policies enter the labeling in exactly two places: their validation action error sets the size of the injected noise for both channels, and the closed-loop channel puts them in the loop.

Three details keep the measurement honest. The perturbation is applied to an action, never to the state, so errors enter the way real errors do. The noise magnitude equals the error a deployed policy actually makes, not an arbitrary constant. And a slope within the probe's own measured noise floor is labeled **deadband**, meaning no confident call either way; deadband stamps inherit the label of their confident neighbors instead of being forced into a wrong verdict.

The raw measurement on the square task looks like this. Thin gray lines are the 8 noise samples, blue is their mean, dashed orange is the fitted exponential whose slope is the label:

| Stable moment | Unstable moment |
| :---: | :---: |
| ![Divergence curve at a stable timestep](/images/blog/stability/curve_square_stable_0.png) | ![Divergence curve at an unstable timestep](/images/blog/stability/curve_square_unstable_0.png) |

The corresponding scenes: a transport moment on the left (deadband), the gripper engaging the nut on the right (unstable):

| Deadband moment | Unstable moment |
| :---: | :---: |
| ![Frame at a deadband timestep](/images/blog/stability/frame_square_gray.png) | ![Frame at an unstable timestep](/images/blog/stability/frame_square_red.png) |

The point of the exercise is that this label moves as the demonstration does. Replaying a demo with its border tinted by the label active at each timestep makes the structure visible directly: red where the measured slope exceeds the deadband, gray inside it, with the running value of $\lambda$ printed along the top.

<div style="display:flex; gap:1.2em; flex-wrap:wrap; justify-content:center; margin:1.5em 0;">
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_square_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_square_labeled.mp4">Download the square label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>square</b> — approach is deadband, the border goes red as the gripper closes on the nut, back to deadband while carrying it, red again as the nut is lowered onto the peg.</figcaption>
  </figure>
  <figure style="flex:0 1 300px; margin:0; text-align:center;">
    <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
      <source src="/images/blog/stability/video_can_labeled.mp4" type="video/mp4">
      <a href="/images/blog/stability/video_can_labeled.mp4">Download the can label video</a>
    </video>
    <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>can</b> — a mostly-transport task. The long reach and the carry are both deadband; red appears only at the grasp and at the release into the bin.</figcaption>
  </figure>
</div>

Two demonstrations are not a statistic, and the proportions below come from hundreds of them rather than from these two. What the videos are meant to show is the shape of the signal: the labels arrive in contiguous stretches tied to contact events, not as frame-to-frame flicker.

## Results

**Most timesteps are neutral; unstable ones are a minority and they cluster.** Across every suite, a quarter to a third of timesteps are confidently unstable, almost none are confidently stable, and the rest are deadband. The can task, chosen as a mostly-transport control, has the smallest unstable share. The only dataset with a visible stable (green) share is the LIBERO stove task, where the knob turn is mechanically guided:

![Regime proportions per dataset](/images/blog/stability/fig_regimes.png)

**The distribution of slopes is one-sided.** Every dataset shows a median near zero, a short left tail, and a long right tail of strong expansion. Moments are either roughly neutral or strongly divergent, with little in between:

![Lambda distribution per dataset](/images/blog/stability/fig_lambda_dist.png)

**Labels form long contiguous bands, not scattered noise.** Unstable stamps group into segments aligned with grasps, insertions, and handoffs. On tool_hang, the hardest task, 40 percent of stamps are unstable and the bands align with the threading phases:

![Label timeline for tool_hang](/images/blog/stability/timeline_tool_hang.png)

**Reacting usually makes divergence worse.** On the same states where fixed playback absorbs the injected error, letting the trained policy react at every step makes the divergence positive at 90 to 99 percent of timesteps: 99.3 percent on lift, 90.4 on can, 93.0 on square, 97.6 on tool_hang. Both histograms below measure the same lift states with the same injected error; open loop (blue) sits at zero, closed loop (magenta) shifts positive almost everywhere:

![Open-loop versus closed-loop divergence on lift](/images/blog/stability/fig_ol_vs_cl_lift.png)

These policies are at 0.9 to 1.0 success, so the explanation is not policy quality. In regions where physics absorbs errors, the plant contributes no error of its own; the only error source present is the policy, and each reaction injects one more sample of it. Mapped onto the table above, the rescue cell (open-loop unstable, closed-loop stable) is rare in our measurements, while the cell where physics absorbs the error and reacting harms (open-loop stable, closed-loop unstable) is the most common state in these datasets. The whole closed-loop-stable column is thin for the same reason.

**Summary.** Instability in these benchmarks concentrates in a few long contact segments, which the probe finds automatically. Outside them, the robot's own corrections are the dominant noise source. The map is cheap to compute and its primary channel needs no trained policy.

*The probe constants, per-suite determinism checks, and full per-dataset statistics are in the project's technical report; this post kept only what is needed to read the figures.*
