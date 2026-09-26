---
title: "DiffusionXRay: A Diffusion and GAN-Based Approach for Enhancing Digitally Reconstructed Chest Radiographs"
collection: publications
category: conferences
permalink: /publication/2025-diffusionxray
excerpt: 'A diffusion model that turns blurry chest X-rays projected from CT scans into sharp, realistic X-rays while keeping small lung nodules visible.'
date: 2025-10-01
venue: 'DEMI Workshop at MICCAI'
paperurl: 'https://arxiv.org/abs/2603.01686'
citation: 'Goyal, A.*, Mittal, A.*, Rao, P., Tadepalli, M., and Putha, P. (2025). &quot;DiffusionXRay: A Diffusion and GAN-Based Approach for Enhancing Digitally Reconstructed Chest Radiographs&quot;. DEMI Workshop at MICCAI. (*equal contribution)'
---

## Summary

Chest X-rays made by projecting CT scans are a cheap source of training data for lung nodule detectors, but they come out blurry and lose fine lung detail. DiffusionXRay is a diffusion model that enhances these images into realistic chest X-rays while keeping small nodules visible. The main difficulty is getting training data for it, so most of the method is about building realistic training pairs.

## The problem

Deep learning models that detect lung nodules on chest X-rays need a lot of labelled data, especially examples with small, subtle nodules. Such examples are hard to collect because they are hard to spot even for radiologists.

One way around this is to make synthetic X-rays from CT scans. You insert a nodule into the CT volume and project the volume into a frontal X-ray. These images are called digitally reconstructed radiographs (DRRs). Because you placed the nodule yourself, the label comes for free.

The catch is image quality. DRRs have low contrast, blurred anatomy and missing fine lung structures. These come from noise in the CT scan and from the way CT volumes are reconstructed. A detector trained on DRRs learns from images that do not look like real X-rays.

An enhancement model could fix this, but training one needs pairs of the same X-ray in low and high quality, and those pairs do not exist. The usual shortcut is to blur good X-rays with bicubic downsampling. That blur looks nothing like DRR degradation, so a model trained on it does not work well on real DRRs.

## Method

<img src="/assets/figures/diffusionxray/pipeline_overview.png" alt="DiffusionXRay pipeline: unpaired datasets, domain transfer with MUNIT-LQ or DDPM-LQ, and the enhancement model" width="800">

*Overview. (a) Unpaired sets of low-quality and high-quality chest X-rays. (b) Two ways to turn a high-quality X-ray into a realistic low-quality copy: MUNIT-LQ and DDPM-LQ. (c) The enhancement model, trained on the resulting pairs.*

The pipeline has two steps.

**Step 1: make realistic low-quality copies of good X-rays.** We treat this as style transfer: keep the anatomy of a high-quality X-ray, and change its appearance to look like a DRR. We try two models for this, both trained on unpaired data.

- **MUNIT-LQ** is a GAN-based image translation model. It splits an image into a content code (the anatomy) and a style code (the image quality). Combining the content of a good X-ray with a low-quality style gives a degraded copy of that X-ray.
- **DDPM-LQ** is a diffusion model trained in two stages. First it is trained on DRRs alone, so it learns what low-quality X-rays look like. Then it is fine-tuned to produce the low-quality version of a given high-quality X-ray.

<img src="/assets/figures/diffusionxray/degradation_comparison.png" alt="Low-quality X-rays generated with bicubic interpolation, MUNIT-LQ and DDPM-LQ" width="700">

*Low-quality copies of the same X-ray made with bicubic interpolation, MUNIT-LQ and DDPM-LQ. Bicubic barely changes the image, while the other two reproduce the haze and loss of detail seen in DRRs.*

**Step 2: train the enhancement model.** We pair 300,000 high-quality X-rays with their synthetic low-quality copies and train a diffusion model (DDPM-HQ) on them. Given a low-quality image, it learns to produce the high-quality one, upscaling from 512×512 to 1024×1024 at the same time.

## Results

We test on the ChestX-ray8 test set (25,596 images), degraded with MUNIT-LQ or DDPM-LQ. The baseline is the same diffusion model trained on bicubic pairs instead of our synthetic pairs.

| Test data degraded with | Model | PSNR ↑ | SSIM ↑ |
|---|---|---|---|
| MUNIT-LQ | Bicubic baseline | 20.08 | 0.83 |
| MUNIT-LQ | DiffusionXRay | **27.50** | **0.92** |
| DDPM-LQ | Bicubic baseline | 19.85 | 0.78 |
| DDPM-LQ | DiffusionXRay | **22.21** | 0.78 |

<img src="/assets/figures/diffusionxray/enhancement_comparison.png" alt="Enhancement results: input, bicubic baseline, DiffusionXRay and reference" width="700">

*Enhancement results. The bicubic baseline keeps most of the blur and haze. DiffusionXRay restores contrast and lung detail and is closer to the original X-ray.*

PSNR and SSIM do not tell you whether a nodule is still visible, so radiologists also compared the two models without knowing which output came from which.

- **Nodule visibility.** The nodule was easy to see in 100% of DiffusionXRay outputs, against 6.6% for the baseline. It could be confused with other structures, such as bones, leads or buttons, in 0% of our outputs and 30% of the baseline's.
- **Image quality.** Lung field clarity improved in 100% of our outputs and 66.7% of the baseline's. However, radiologists also reported a significant increase in noise in 72.9% of our outputs, against 25% for the baseline. The sharper images come with more noise.

The model is also expensive to run, since diffusion models generate images over many denoising steps.

## Released data

- **LQ-CXR12K:** 12,580 low-quality chest X-rays projected from low-dose CT scans.
- Low-quality versions of the ChestX-ray8 test split (25,596 images), made with MUNIT-LQ and DDPM-LQ, together with the original high-quality images.
