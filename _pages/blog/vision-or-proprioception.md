---
layout: single
title: "Do You Need Vision to Predict Stability Regimes?"
permalink: /blog/vision-or-proprioception/
author_profile: true
---

At any moment in a manipulation episode, can a model predict whether a small error will be absorbed or amplified? And if it can, what kind of model does that take: a video model that predicts in latent space, or a plain image encoder?

I ran the ablation. Neither is needed.

## The setup

The target is a per-timestep stability label, measured by perturbing one action in the simulator and recording how fast the perturbed and unperturbed branches separate. About 102,000 labeled timestamps across robomimic and MimicGen.

The model is a small attention head on frozen features. Nothing is fine-tuned. The only thing that changes between rows is what the head is allowed to see: same pool, same held-out demonstrations, same threshold, same seed. Scores are AUROC on the stamps whose label is unambiguous.

## The result

| Input | Head size | AUROC |
|---|---|---|
| Proprioception only, 9 numbers | 22k params | **0.895** |
| V-JEPA 2, 16 frames, plus proprioception | 4M params | 0.883 |
| V-JEPA 2, 16 frames, vision only | 4M params | 0.871 |
| DINOv2, 1 frame, plus proprioception | 4M params | 0.863 |
| DINOv2, 1 frame, vision only | 4M params | 0.844 |

Nine numbers of joint state beat a 300M-parameter video transformer. Every addition of visual information made it slightly worse.

For scale, re-running one condition at a different seed moves the score by about 0.007, so the exact ordering sits near the edge of what a single seed resolves. The claim that survives that is not "proprioception wins." It is that vision adds nothing.

## Why

Whether a small error grows depends on the contact configuration you are in right now: what is gripped, how tightly, how close the part is to the constraint that will jam it. Joint angles and gripper width already encode most of that. The camera is looking at a scene whose stability-relevant content is largely already present in the arm's own state.

The same result says the 16-frame clip is not buying motion either. A single DINOv2 frame ties the video model to within noise. The quantity being predicted is a static property of the current configuration, not a property of the trajectory.

## So, world model or DINOv2?

Wrong question for this target. Both were used identically, as frozen feature extractors, and neither justified its cost.

It is worth being precise about what a video model was not doing here. V-JEPA 2 is a latent-space predictive model, but we never asked it to predict anything. We took its encoder output and pooled it. That is not a test of world models. It is a test of two frozen encoders, one trained on video and one on images.

A world model earns its place when you need it to imagine, not to encode. The real use is producing the labels in the first place. On hardware there is no simulator to reset and replay, so an action-conditioned model has to roll out the perturbed and unperturbed futures in latent space and measure the divergence between them. An image encoder cannot do that job at all. Reading a static property off the current configuration, by contrast, can be done by almost anything, including nothing visual.

## The limit

None of it transfers. A head trained on one task scores 0.49 on an unseen task. A head trained on seven task families scores 0.35 on the eighth, which is below chance. Whatever these models learn about stability, they learn it in the coordinate system of the tasks they were shown.

So the practical answer is small and specific. Predict the regime from proprioception, spend nothing on a visual encoder, and budget for labeling each new task family you care about.
