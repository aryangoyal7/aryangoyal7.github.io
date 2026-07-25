---
layout: single
title: "Stable and Unstable Moments in Robot Demonstration Data"
permalink: /blog/stability-regimes/
author_profile: true
---

## Open-loop and closed-loop stability

One question, asked once per timestep: an error occurs at this instant. Do the dynamics absorb it or amplify it? Open-loop and closed-loop stability are the control-theoretic names for two versions of that question, and everything below turns on the difference.

**Open-loop stability** is a property of the plant alone: the commands are held to what the demonstration recorded, nothing reacts, and what is left is physics and contact geometry. **Closed-loop stability** asks the same question with the controller back in the loop, so what is measured is the plant and the feedback law together. Correcting is supposed to shrink the error. Whether it does is an empirical property of that policy at that state, and it can go either way, because a correction computed from a slightly-off observation is itself slightly off.

**Making that precise.** Reset the simulator to the exact recorded state at time $t_0$ and run the next $K = 24$ steps (1.2 seconds at 20 Hz) twice. The nominal branch replays the recorded actions $a_{t_0}, \ldots, a_{t_0 + K - 1}$ unchanged. Each perturbed branch adds Gaussian noise to the position components of the first action only, never rotation and never the gripper, then replays the remaining $K-1$ actions exactly as recorded:

$$
\tilde{a}_{t_0} = a_{t_0} + \varepsilon, \qquad \varepsilon \sim \mathcal{N}\left(0, \sigma_u^2 I_3\right)
$$

The error therefore enters through the actuation channel, the way a real policy error enters, and reaches the state only by passing through the plant. Writing $x^{\mathrm{nom}}$ for the task-space vector of the nominal branch and $x^{\mathrm{pert}}$ for that of the $n$-th perturbed branch, both at step $\tau$ after $t_0$, the divergence between them is

$$
d_n(\tau) = \left\lVert x^{\mathrm{pert}}_n(\tau) - x^{\mathrm{nom}}(\tau) \right\rVert_2
$$

If the moment amplifies errors this distance grows exponentially in $\tau$; if it absorbs them it decays. The rate is a finite-time Lyapunov exponent. Finite-time because a manipulation episode is a sequence of short phases with no steady state to settle into, and it is estimated the textbook way (Benettin et al., 1980), as the least-squares slope of the branch-averaged log divergence $\ell(\tau) = \frac{1}{N} \sum_{n=1}^{N} \log d_n(\tau)$ against time, over $N = 8$ independent perturbations:

$$
\lambda(t_0) = \frac{\sum_{\tau=1}^{K} \left(\tau - \bar{\tau}\right)\left(\ell(\tau) - \bar{\ell}\right)}{\sum_{\tau=1}^{K} \left(\tau - \bar{\tau}\right)^2}
$$

So $\lambda(t_0) > 0$ means an error introduced at $t_0$ is amplified, and $\lambda(t_0) < 0$ means it is absorbed. We call one measured timestep of one demonstration a *stamp*, and the label of a stamp is the sign of $\lambda$ read against a threshold $\delta$: **unstable** above $+\delta$, **stable** below $-\delta$, and **deadband** in between, meaning the slope falls inside the probe's own noise floor and the stamp is unproven either way. Crucially $\delta$ is not a tuned constant. It is the 95th percentile of $\lvert \lambda \rvert$ over free-space stamps of the same dataset, which is the measured noise floor of the probe itself, computed separately for every dataset.

The closed-loop channel is the same probe with one substitution. After the identical injection, the perturbed branch is no longer replay: the trained policy $\pi$ picks a fresh action from its own camera observations at every step,

$$
\tilde{a}_{t_0 + \tau} = \pi\left(o_{t_0 + \tau}\right), \qquad \tau = 1, \ldots, K-1
$$

and $\lambda$ is computed from the resulting divergence exactly as above. Everything else, the injection, the window, the readout, the threshold, is held identical, so the two channels differ only in what happens after the error is introduced.

## Why this is worth measuring

A policy trained by imitation makes a small error at every step, and those errors do not stay small. Each one nudges the robot toward states that appear less often in the demonstrations, where the policy is less reliable and errs by more, which nudges it further still. The consequence is that the cost of behavior cloning grows with the *square* of the task horizon rather than linearly with it ([Ross et al., 2011](https://arxiv.org/abs/1011.0686)). Manipulation tasks are long. The tool_hang demonstrations used here run past six hundred steps, so this compounding, not per-step accuracy, is the dominant failure mode.

The prescribed escape is action chunking. Rather than predicting one action and re-deciding at every timestep, the policy predicts a block of $k$ actions and executes the whole block open-loop before observing again; [ACT](https://arxiv.org/abs/2304.13705) and [Diffusion Policy](https://arxiv.org/abs/2303.04137) both work this way. The horizon does not get any shorter, but the number of decision points along it falls from $T$ to $T/k$, and so does the number of chances to inject a fresh error. What is given up is reactivity: for the length of a chunk the robot is committed, and nothing that happens in the world can change what it does.

So the choice is between long chunks and short chunks, and it is normally made once per task and then held fixed. Long chunks suppress compounding but ride out any disturbance blindly; short chunks stay responsive but re-inject policy noise at every step. Which one is right depends on whether the physics at the current moment forgives a mistake or punishes it, and that is not constant along a demonstration. A demonstration that carries an object across free space and then seats it into a tight fixture wants opposite settings for the two, and they are a second apart in the same episode.

That is what sent us to the data. We labeled sixteen open-source datasets stamp by stamp, the open-loop channel on all of them and the closed-loop channel on the four robomimic tasks, to find out how often a demonstration switches regime and where the mass of these datasets actually sits.

## The possible combinations

Each channel reports one of the three classes above, stable, unstable, or deadband, so three classes on two channels gives nine combinations.

What hangs on them is the chunk length $k$: $k = 1$ is replanning at every step, and large $k$ means committing to a long stretch of actions. Two rules then fix every cell. The open-loop class decides whether committing is safe, and the closed-loop class decides whether reacting helps or hurts.

|                 | CL stable         | CL deadband       | CL unstable        |
|-----------------|-------------------|-------------------|--------------------|
| **OL stable**   | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL deadband** | commit long $k$ | commit long $k$ | commit long $k$  |
| **OL unstable** | replan every step | short $k$       | intermediate $k$ |

The interesting variation is confined to the open-loop-unstable row; everywhere else commitment wins. Taking the cells in turn: when physics absorbs the error but the policy's corrections add new error (top right), reacting is the harmful choice, so committing is not merely cheaper but safer. A deadband row behaves the same way and for a sharper reason. The plant contributes no error of its own there, so the policy is the only error source present, and each replan injects one fresh mistake that nothing absorbs; committing chunks of $k$ accumulates $T/k$ such mistakes over a $T$-step segment instead of $T$. In the bottom row, physics amplifies the error, and replanning at every step is right only where feedback genuinely rescues (bottom left). When both channels expand (bottom right), neither mode contains the error, success depends on the task geometry guiding the state back the way a hole guides a peg, and the best available compromise is an intermediate $k$ that keeps some reaction while injecting less policy noise.

The results below show which of these actually occur in the data, and in what proportion.

## The datasets

We labeled the four robomimic tasks (lift, can, square, tool_hang; 200 human demonstrations each), two MimicGen datasets (stack and square variants), and all ten LIBERO-Long files. They cover the difficulty range of benchmark manipulation, from lift, where a cube is picked off an open table, to tool_hang, where a frame is threaded onto a thin stand at millimeter tolerance.

The reason to label them: a stability map shows what kind of data a dataset contains, how much is transport where any policy coasts, how much is contact where errors compound and failures concentrate, and at exactly which timesteps demonstrations are fragile. All of this is available before any policy is evaluated on the data.

## Running the probe

Stamps sit on a stride-2 grid, so $\lambda$ is measured every second timestep of every demonstration, across hundreds of demonstrations per dataset. Deadband stamps then inherit the label of their confident neighbors, and a median filter over roughly three consecutive stamps smooths the sequence; contiguous runs of one label are the segments. Nothing is segmented by hand.

Each stamp looks $K$ steps ahead while the stamps themselves are two steps apart, so neighboring windows overlap almost entirely and a window near a regime boundary spans both a contracting stretch and a diverging one. That blend is the honest answer to the question the label asks: committing a chunk just before contact really is risky, because the contact falls inside the commitment window. A window that mixes the two regimes evenly produces a small slope, which lands in the deadband rather than forcing a confident call in the wrong direction. Segment edges therefore arrive slightly early going into an unstable stretch and roughly on time coming out, which is the direction of bias a safety label should have.

The one place a trained policy enters is $\sigma_u$ and the closed-loop channel. We trained and verified diffusion policies first (lift and can at 1.0 success, square and tool_hang at 0.9); their validation action RMSE sets $\sigma_u$, so the probe injects exactly the size of error a deployed policy actually makes rather than an arbitrary constant, and the closed-loop channel puts them in the loop. Ordinary policy rollouts play no part in labeling at all.

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

**Most timesteps are neutral; unstable ones are a minority and they cluster.** Across every suite, a quarter to a third of timesteps are confidently unstable and the rest are deadband. Confidently stable stamps are almost nonexistent: 0 to 4 percent per dataset, essentially only where a mechanism guides the motion, as in the LIBERO stove task above. The can task, chosen as a mostly-transport control, has the smallest unstable share:

![Regime proportions per dataset](/images/blog/stability/fig_regimes.png)

**The distribution of slopes is one-sided.** Every dataset shows a median near zero, a short left tail, and a long right tail of strong expansion. Moments are either roughly neutral or strongly divergent, with little in between:

![Lambda distribution per dataset](/images/blog/stability/fig_lambda_dist.png)

**Labels form long contiguous bands, not scattered noise.** 82 to 94 percent of open-loop stamp mass sits in runs of three stamps or more, six or more consecutive timesteps carrying the same label. The bands align with grasps, insertions, and handoffs, and the alignment holds across demonstrations of the same task. On tool_hang, the hardest task, 40 percent of stamps are unstable and the bands sit exactly on the threading phases:

![Label timeline for tool_hang](/images/blog/stability/timeline_tool_hang.png)

**Reacting usually makes divergence worse.** On the same states where fixed playback absorbs the injected error, letting the trained policy react at every step makes the divergence positive at 90 to 99 percent of timesteps: 99.3 percent on lift, 90.4 on can, 93.0 on square, 97.6 on tool_hang. Both histograms below measure the same lift states with the same injected error; open loop (blue) sits at zero, closed loop (magenta) shifts positive almost everywhere:

![Open-loop versus closed-loop divergence on lift](/images/blog/stability/fig_ol_vs_cl_lift.png)

These policies are at 0.9 to 1.0 success, so the explanation is not policy quality. In regions where physics absorbs errors, the plant contributes no error of its own; the only error source present is the policy, and each reaction injects one more sample of it. Mapped onto the table above, the rescue cell (open-loop unstable, closed-loop stable) is rare in our measurements, while the cell where physics absorbs the error and reacting harms (open-loop stable, closed-loop unstable) is the most common state in these datasets. The whole closed-loop-stable column is thin for the same reason.

**Summary.** The stability regime is not a property of a task; it changes within a demonstration, several times, in long bands tied to contact events. Instability concentrates in those few bands, which the probe finds automatically. Outside them, the robot's own corrections are the dominant noise source. The map is cheap to compute and its primary channel needs no trained policy.

*The probe constants, per-suite determinism checks, and full per-dataset statistics are in the project's technical report; this post kept only what is needed to read the figures.*
