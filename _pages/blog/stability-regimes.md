---
layout: single
title: "Measuring the Stability Assumption Behind Action Chunking"
permalink: /blog/stability-regimes/
author_profile: false
excerpt: "A state-wise, finite-horizon study of how action errors propagate under open-loop execution and closed-loop replanning."
---

**Aryan Goyal** · Independent Researcher · [garyan18@gmail.com](mailto:garyan18@gmail.com)

![Overview of the state-wise perturbation probe, its finite-horizon stability labels, and the prediction and rollout comparisons.](/images/blog/stability/stability-measurement.svg)

## Abstract

Action chunking improves the performance of policies learned by behavioural cloning, and several mechanisms have been proposed to explain why, including temporal consistency, horizon reduction, representation learning, and reduced error compounding. Every one concerns the error the policy introduces, whether by reducing how much of it enters or how often it re-enters the policy's own input. We instead study what happens to an action error once it enters the system. At each state, we inject a small action error and measure how fast it grows or shrinks under two execution regimes: open-loop, where the rest of the chunk is replayed without replanning, and closed-loop, where the policy replans after the perturbation. The fitted rate labels each state as contracting, expanding, or unresolved. Across twelve manipulation tasks from three benchmark suites, we find that confidently stable states are rare, while error amplification is common among states whose propagation rate can be resolved. We further find that the measured propagation rate depends strongly on the fitting horizon: amplification is typically front-loaded, so short windows can substantially overestimate longer-horizon propagation. Finally, we train predictors on these labels and find that a state's open-loop regime can be recovered from camera frames and proprioception alone, without a simulator, while its closed-loop propagation is only partially recoverable because it also depends on how the policy acts after the perturbation. These results suggest that error-compounding arguments alone do not provide a complete account of action chunking: neither passive open-loop dynamics nor policy replanning consistently contracts an injected error, and replanning rarely turns open-loop amplification into confident contraction. This also suggests that closed-loop reactivity should be trained explicitly, using perturbation- and tree-coverage-oriented training to expose the policy to deviations it must recover from, rather than expected to emerge reliably from standard imitation learning.

## Why measure what happens after an error?

Action chunking is often discussed through how a policy produces a sequence: a chunk may preserve temporal consistency, reduce the number of decisions, or change how prediction errors enter the policy's future inputs. This paper isolates a different part of the problem. Once a small action error has entered the system, do the dynamics absorb it, preserve it, or amplify it? And does the answer change when the policy can replan?

The distinction matters because explanations based on a single contraction rate make a strong assumption: that one rate summarizes propagation over a task or an entire execution horizon. We test that assumption directly by measuring error propagation from individual states, then asking how the measurement changes with its fitting window, whether it can be predicted from observations, and whether the picture holds across different state and action sources.

## A per-state perturbation measurement

At each sampled state, we add a small perturbation to the translation component of the first action. Its scale is set from the trained policy's action prediction error on held-out demonstrations, so the probe reflects an error of the size the policy makes. We then compare the perturbed trajectory with a nominal trajectory from the same state.

In the **open-loop** probe, the remaining recorded actions are replayed unchanged. This measures how the dynamics and task geometry propagate the initial error during a committed sequence. In the **closed-loop** probe, the policy replans from its own observations after the same initial perturbation. Paired nominal and perturbed runs use the same policy randomness, isolating the effect of the injected error from variation caused by sampling.

For each branch, we fit log state deviation over a window of \(K\) steps:

$$\log \lVert y_j-y_j^{(n)}\rVert \approx \alpha_n + \lambda_n j.$$

The aggregated slope \(\lambda\) is a finite-horizon propagation rate: positive values indicate amplification, negative values indicate contraction. The fitting window \(K\) belongs to the measurement; it is not necessarily the length of the executed action chunk. A label is called stable or unstable only when its rate is both large enough to represent a meaningful change over that window and sufficiently resolved from zero. Otherwise, it is assigned to the **deadband**. Deadband therefore means that the probe does not support a confident stable or unstable label; it does not mean that the true rate is zero.

## The horizon changes the answer

The first result is that error propagation is state-dependent and horizon-dependent. Confident contraction is uncommon, while amplification is common among the measurements whose rate can be resolved. More importantly, the measured rate is not a single constant that can be carried unchanged from a short chunk to a longer execution.

The divergence is typically front-loaded: perturbed branches separate quickly in the first few steps, then tend to flatten. A short \(K=8\) window therefore fits mainly the initial rise; a longer \(K=24\) window also includes the later plateau and pulls the average rate toward zero. On robomimic lift demonstrations, for example, the unstable share is 69.7% at \(K=8\) and 21.1% at \(K=24\), while the stable share remains 0%. Across the evaluated conditions, shortening the window moves many stamps from deadband to unstable, with comparatively little change in the stable share.

This is why we treat \(\lambda\) as a summary of propagation over a specified interval, rather than a time-invariant rate for the whole task. The result qualifies contraction-based accounts of chunking: an error may amplify early and then saturate over a longer execution. The probe measures one perturbation's evolution, so it does not establish how errors from successive chunk boundaries accumulate, nor does it identify an optimal chunk length.

## Can observations predict the measured regime?

After measuring stability in simulation, we ask whether the signal is visible to a predictor that receives camera observations and proprioception. For open-loop propagation, the predictor distinguishes more amplification-prone from less amplification-prone states: across demonstration and rollout stamps with enough negative examples, AUROC ranges from 0.63 to 0.94. It is less accurate at predicting the magnitude of the rate than at ranking states by their propensity to amplify. Adding the action sequence being evaluated generally improves rate regression and rank correlation, as expected when the target depends on that particular sequence, though classification metrics do not improve uniformly.

Closed-loop propagation is only partially recoverable from the same inputs. The predictor improves on a constant-median baseline for most regression settings, but the gains are less consistent than for open-loop propagation. This target depends both on the current configuration and on how the deployed policy acts after the perturbation. The difference between the two prediction results reflects what each rate depends on, not a change in predictor architecture.

## Demonstrations, policy rollouts, and emitted chunks

We evaluate twelve manipulation tasks across robomimic, MimicGen, and RoboCasa. The open-loop probes use three stamp types that separate changes in state distribution from changes in action source:

- **Demonstration stamps** use recorded demonstration states and demonstrator actions.
- **Rollout stamps** use states reached by a policy and the actions it executed.
- **Policy-chunk stamps** use policy rollout states at replan boundaries and the action chunk emitted there.

Demonstration versus rollout changes the state distribution; rollout versus policy-chunk changes which actions are replayed. Closed-loop probes are evaluated at the same policy stamps, so replanning can be compared with open-loop execution from identical states. Because policy chunks can be shorter than the main fitting window, we also refit demonstration and rollout probes at \(K=8\) for matched comparisons.

On robomimic, the demonstration and rollout medians at \(K=24\) are close: lift is 0.015 versus 0.019, can 0.009 versus 0.019, square 0.017 versus 0.018, and tool hang 0.026 versus 0.021. The state distribution shift from demonstrations to policy-visited rollouts therefore does not substantially change the median open-loop rate in these tasks. Across the broader comparison, neither passive open-loop dynamics nor policy replanning can be assumed to contract an injected error. States that are unstable in open loop rarely become confidently stable under replanning; they more often move into the deadband or remain unstable.

## What this says about action chunking

We began with a common explanatory step: if chunking prevents error compounding, perhaps the system contracts errors over a task horizon at one characteristic rate. Measuring from individual states shows why that picture is incomplete. The propagation rate varies across states, and the fitted rate changes with the observation window because early amplification is often followed by saturation. A task-wide or horizon-wide contraction constant is not supported by these measurements.

Prediction adds a second part to the story. Some open-loop propagation structure is present in camera observations and proprioception, while closed-loop propagation is less uniformly recoverable because it also depends on the policy's behavior after the perturbation. Comparing demonstrations, policy rollouts, and policy chunks shows that these conclusions are not tied to a single source of states or actions.

The results do not determine the best execution horizon or task performance; stability is one factor alongside temporal consistency, replanning cost, and other effects of action chunking. They do show that reliable closed-loop error recovery should not be expected to emerge from standard imitation learning alone. If recovery is desired, perturbation- and tree-coverage-oriented training can expose policies to the deviations they need to correct.

<details>
  <summary>Video examples (to be refreshed)</summary>

  <div style="display:flex; gap:1.2em; flex-wrap:wrap; justify-content:center; margin:1.5em 0;">
    <figure style="flex:0 1 300px; margin:0; text-align:center;">
      <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
        <source src="/images/blog/stability/video_square_labeled.mp4" type="video/mp4">
        <a href="/images/blog/stability/video_square_labeled.mp4">Download the square label video</a>
      </video>
      <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>Square</b></figcaption>
    </figure>
    <figure style="flex:0 1 300px; margin:0; text-align:center;">
      <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
        <source src="/images/blog/stability/video_can_labeled.mp4" type="video/mp4">
        <a href="/images/blog/stability/video_can_labeled.mp4">Download the can label video</a>
      </video>
      <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>Can</b></figcaption>
    </figure>
    <figure style="flex:0 1 300px; margin:0; text-align:center;">
      <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
        <source src="/images/blog/stability/video_libero_stove_labeled.mp4" type="video/mp4">
        <a href="/images/blog/stability/video_libero_stove_labeled.mp4">Download the LIBERO stove label video</a>
      </video>
      <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>Stove and moka pot</b></figcaption>
    </figure>
    <figure style="flex:0 1 300px; margin:0; text-align:center;">
      <video autoplay loop muted playsinline controls preload="metadata" style="width:100%; height:auto; border-radius:3px;">
        <source src="/images/blog/stability/video_libero_mokapots_labeled.mp4" type="video/mp4">
        <a href="/images/blog/stability/video_libero_mokapots_labeled.mp4">Download the LIBERO moka pots label video</a>
      </video>
      <figcaption style="font-size:0.8em; line-height:1.35; margin-top:0.5em;"><b>Two moka pots</b></figcaption>
    </figure>
  </div>
</details>
