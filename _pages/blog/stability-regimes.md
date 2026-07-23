---
layout: single
title: "Where Can a Robot Act Blind? Open-Loop and Closed-Loop Stability Regimes in Open-Source Manipulation Datasets"
permalink: /blog/stability-regimes/
author_profile: true
---

Imagine a robot arm carrying a peg across a table. If its hand drifts two millimeters off course, nothing happens: it arrives two millimeters to the side and the task still succeeds. Now imagine the same arm halfway through inserting that peg into a tight hole. The same two millimeters means the peg catches the rim, jams, and the episode fails. Same robot, same error, completely different outcome. The difference is a property of the physics at that moment, and this post is about measuring it, timestep by timestep, across popular open-source manipulation datasets: robomimic, MimicGen, and LIBERO-Long.

The question we ask at every single timestep of every demonstration is simple: **if a small mistake happens right here, does physics forgive it or punish it?** We call the forgiving moments open-loop stable and the punishing ones open-loop unstable, and we also ask a second question: does reacting to the mistake help? That second question turns out to have a surprising answer.

## The algorithm

**Open-loop stability.** For a given demonstration and timestep, we reset the simulator to the exact recorded state and roll the next 1.2 seconds forward twice. The first run replays the demonstration's recorded actions exactly. The second run replays the same actions, except we add a small amount of Gaussian noise to the position part of the very first action. Nothing else changes and nobody reacts; both runs are fixed playback. We then track the distance between the two trajectories over those 24 control steps and fit the slope of its logarithm. If the gap shrinks or stays flat, that timestep is open-loop stable: physics absorbed the error. If the gap grows exponentially, it is open-loop unstable: physics amplified it. The slope is a finite-time Lyapunov exponent, and we compute it with 8 noise samples per timestep, every 2 steps, over hundreds of demonstrations per dataset.

Two design choices matter. First, the perturbation is applied to an action, never to the state. We never teleport the robot; the error enters through the actuation channel, exactly the way real errors do. Second, the noise magnitude is not arbitrary: it is set to the validation action error of a policy actually trained on the dataset, so the probe injects precisely the size of mistake a deployed policy makes.

Here is what the two outcomes look like as raw divergence curves on the square task. The thin gray lines are the 8 individual noise samples, blue is their mean, the dashed orange line is the fitted exponential whose slope is the label, and the axis is the log distance between the perturbed and unperturbed runs over the 24-step window:

| Stable moment | Unstable moment |
| :---: | :---: |
| ![Divergence curve at a stable timestep](/images/blog/stability/curve_square_stable_0.png) | ![Divergence curve at an unstable timestep](/images/blog/stability/curve_square_unstable_0.png) |

And here is what those moments look like in the scene. The left frame is a neutral transport moment (the label is in the deadband, explained below). The right frame is an unstable moment, the gripper about to engage the nut where a small error changes the outcome:

| Deadband moment | Unstable moment |
| :---: | :---: |
| ![Frame at a deadband timestep](/images/blog/stability/frame_square_gray.png) | ![Frame at an unstable timestep](/images/blog/stability/frame_square_red.png) |

**The deadband.** A measured slope near zero does not prove stability or instability; it may just be measurement noise. So we compute, per dataset, the 95th percentile of the slope magnitude over free-space moments, where nothing interesting is happening, and treat that value as the noise floor of the probe itself. Any timestep whose slope lands within the floor is labeled deadband, which honestly means "no confident call either way." Deadband stamps then inherit the label of their confident neighbors, and a short median filter removes single-stamp flickers. Nothing is ever segmented by hand.

**Closed-loop stability.** The closed-loop probe repeats the same experiment with one change: after the identical injected error, the perturbed run is no longer fixed playback. A trained diffusion policy looks at its own camera observations and picks a fresh action at every step, reacting the way it would at deployment. If reacting shrinks the gap, the timestep is closed-loop stable. If reacting makes the gap grow, closed-loop unstable. Comparing the two channels at the same timesteps asks the interesting question: does feedback correct errors, or does it inject them?

**The possible regime combinations.** Each channel answers grow or not-grow, so there are four physical scenarios. Both stable: everything is forgiving, act however you like. Open-loop stable but closed-loop unstable: physics forgives the error, yet the policy's own corrections add error; reacting is the harmful choice here. Open-loop unstable but closed-loop stable: physics punishes blind execution but feedback rescues it; this is the textbook case where reacting is essential. Both unstable: nothing locally contains the error, and success depends on the task geometry funneling the state back, like a hole guiding a peg. Keep these four in mind; the punchline of the data is which of them actually occur.

## The datasets

We labeled the four robomimic tasks (lift, can, square, tool_hang; 200 human demonstrations each), two MimicGen datasets (stack and square variants), and all ten LIBERO-Long files. These suites span the difficulty range of current benchmark manipulation: from lift, where the robot picks a cube off an open table, to tool_hang, where a frame must be threaded onto a thin stand with millimeter tolerances.

Why bother labeling at all? Because a stability map tells you what kind of data you actually have: how much of a dataset is easy transport where any policy will coast, versus contact where errors compound and failures concentrate. It localizes the exact timesteps where a demonstration is fragile, which is where trained policies actually fail. And it does this from the dataset alone, before any policy is evaluated on it.

## The trained policies

For each robomimic task we trained an image-based diffusion policy and verified it properly before using it: lift and can reach a 1.0 success rate, square 0.9, tool_hang 0.9. These policies serve the labeling in two distinct places. First, their validation action error calibrates the size of the injected perturbation, for both channels. Second, only the closed-loop channel puts a policy in the loop, driving the perturbed branch step by step. The open-loop channel never touches a policy, which is why it could be computed on every dataset, including ones where we trained nothing.

## Findings

**1. Manipulation data is mostly forgiving, with concentrated pockets of danger.** Across every suite, roughly a quarter to a third of timesteps are confidently unstable, almost none are confidently stable, and the rest sit in the deadband:

![Regime proportions per dataset](/images/blog/stability/fig_regimes.png)

Red is unstable, gray is deadband, green is stable. Two outliers carry the message. The can task, which we chose as a mostly-transport control, has by far the smallest red share. And the only dataset with a visible green share is the LIBERO stove task, where a knob turn is mechanically guided: the mechanism physically funnels errors back, which is exactly what confident stability should mean.

**2. The instability lives in a long right tail.** The slope distribution has the same one-sided shape in every dataset: a median just above zero, a short left tail, and a long right tail of strong expansion. Nature appears to offer manipulation two options, roughly neutral or strongly divergent, and almost nothing in between:

![Lambda distribution per dataset](/images/blog/stability/fig_lambda_dist.png)

**3. The labels form long coherent bands, not noise.** Unstable timesteps cluster into contiguous segments that line up with contact phases, insertions, grasps, and handoffs, with clean transitions into and out of them. Here is a timeline for tool_hang, the hardest task, where 40 percent of stamps are unstable and the unstable bands align with the threading phases:

![Label timeline for tool_hang](/images/blog/stability/timeline_tool_hang.png)

**4. Reacting is usually the destabilizing choice.** This is the finding we did not expect to be so extreme. On the same states where fixed playback absorbs the injected error, letting the trained policy react at every step makes the divergence positive at 90 to 99 percent of timesteps, task after task: 99.3 percent on lift, 90.4 on can, 93.0 on square, 97.6 on tool_hang. Here is lift, the most benign plant of all. Both histograms measure the same states with the same injected error size. The open-loop distribution (blue) sits concentrated at zero: fixed replay carries the error without amplifying it. The closed-loop distribution (magenta) is the same measurement with the policy reacting, and the whole distribution shifts positive:

![Open-loop versus closed-loop divergence on lift](/images/blog/stability/fig_ol_vs_cl_lift.png)

The interpretation is not that the policies are bad; these are policies at 0.9 to 1.0 success. It is that in forgiving regions the plant contributes no error of its own, so the only error source present is the policy, and every fresh reaction injects one more sample of it. Of the four regime combinations, the textbook rescue case (open-loop unstable, closed-loop stable) turned out to be rare in our measurements, while the counterintuitive one (open-loop fine, reacting harmful) is the single most common state of affairs in these datasets.

**5. The practical reading.** If you take one thing away: in these benchmarks, the danger is concentrated in a few long contact segments that the probe can find automatically, and outside those segments the robot's own corrections are the dominant noise source. A stability map of a dataset is cheap to compute, needs no trained policy for its primary channel, and tells you where your demonstrations are fragile before you ever train on them.

*The probe constants, per-suite determinism checks, and full per-dataset statistics are documented in the project's technical report; this post kept only what is needed to read the figures.*
