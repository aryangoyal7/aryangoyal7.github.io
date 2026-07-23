---
layout: single
title: "Open-Loop and Closed-Loop Stability Regimes in Open-Source Manipulation Datasets"
permalink: /blog/stability-regimes/
author_profile: true
---

## Open-loop and closed-loop stability

Consider a robot arm carrying a peg across a table. If its hand drifts two millimeters off course, nothing happens: it arrives two millimeters to the side and the task still succeeds. Now consider the same arm halfway through inserting that peg into a tight hole. The same two millimeters means the peg catches the rim and jams. Same robot, same error, different outcome. Whether an error is absorbed or amplified is a property of the physics at each moment, and we measured it, timestep by timestep, across several open-source manipulation datasets.

**Open-loop stability** asks: if a small mistake happens here and nobody reacts, does the error die out or grow? A moment is open-loop stable when the error dies out or persists harmlessly, and open-loop unstable when it grows.

**Closed-loop stability** asks: what if the robot reacts? A trained policy observes the scene and corrects at every step after the mistake. A moment is closed-loop stable when reacting shrinks the error, and closed-loop unstable when reacting makes it grow.

## The possible combinations

Each channel reports one of three classes: stable, unstable, or **deadband**, meaning the measured slope sits inside the probe's own noise floor, so neither call is confident (the algorithm section explains how that floor is set). Three classes on two channels gives nine combinations.

What hangs on them is how many actions the robot should commit to before it looks at the world again. Write $k$ for that number: $k = 1$ is replanning at every step, and large $k$ means committing to a long stretch of actions. Two rules then fix every cell. The open-loop class decides whether committing is safe, and the closed-loop class decides whether reacting helps or hurts.

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
