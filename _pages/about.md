---
permalink: /
title: ""
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

Hi, I'm Aryan Goyal.
I study at [Indian Institute of Technology Bombay](https://www.iitb.ac.in/).
I work in AI research.
I currently work on mechanistic interpretability for vision-language-action models.
I previously worked on fine-grained medical imaging.
I write technical blogs on [Substack](https://substack.com/@aryanx07).

{% include base_path %}

## Publications
- [DiffusionXRay: A Diffusion and GAN-Based Approach for Enhancing Digitally Reconstructed Chest Radiographs](/publication/2025-diffusionxray). DEMI Workshop at [MICCAI](https://conferences.miccai.org/2025/en/), 2025.
- [A Diffusion-Driven Fine-Grained Nodule Synthesis Framework for Enhanced Lung Nodule Detection from Chest Radiographs](/publication/2026-nodule-synthesis). [OpenReview](https://openreview.net/forum?id=7DL7cu8Ui8), MIDL 2026 (submitted).
- [ACADEMIA: Curriculum-Aligned Multi-Agent Orchestration for K-12 Education](https://openreview.net/). ICLR 2026 Workshop (submitted).
- A System and Method for Projecting Synthetic Nodules in Medical Imaging. Indian Patent Application No. 202521024259 (filed).

## Blog Posts

{% for post in site.posts reversed %}
  {% include archive-single.html %}
{% endfor %}

## Talks and Tutorials

- [Normalising flows and flow models slides](/files/Normalising_flows_and_flow_models_slides.pdf)
- [Normalising flows and flow models slides (alternate)](/files/Normalising_flows_and_flow_models_slides%20(1).pdf)
- [Qure Presentation](/files/Qure_Presentation%20(9).pdf)

{% for post in site.talks reversed %}
  {% include archive-single-talk.html %}
{% endfor %}

{% for post in site.tutorials reversed %}
  {% include archive-single.html %}
{% endfor %}

## Previous Experience

See [Previous Experience](/previous-experience/) for internships, research assistantship, and teaching assistantship.
