---
permalink: /
title: ""
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

Hi, I'm Aryan Goyal. 
I will graduate from IIT Bombay this summer.

I work in AI research.
Currently I am working with Mechansitic Interpretability of Vision Language Action Models.

My previous work was most with fine-grained medical imaging.

I write technical blogs about my scrappy experiments on [Substack](https://substack.com/@aryanx07).

{% include base_path %}

## Publications
Publications and Patents
- Aryan Goyal, Ashish Mittal. "DiffusionXRay: A Diffusion and GAN-Based Approach for Enhancing Digitally Reconstructed Chest Radiographs." Proceedings of the Data Engineering in Medical Imaging (DEMI) Workshop @ MICCAI 2025. [Blog](#) | [Paper](/publication/2025-diffusionxray) | [Poster](#) | [Code](#)
- Aryan Goyal, Shreshtha Singh. "A Diffusion-Driven Fine-Grained Nodule Synthesis Framework for Enhanced Lung Cancer Detection from Chest Radiographs." (Medical Imaging with Deep Learning, MIDL 2026)
- Aryan Goyal. "ACADEMIA: Curriculum-Aligned Multi-Agent Orchestration for K-12 Education." Multi-Agent Learning and Its Opportunities in the Era of Generative AI (ICLR 2026 Workshop), Submitted
- Aryan Goyal, Qure.ai Technologies Private Limited. "A System and Method for Projecting Synthetic Nodules in Medical Imaging." Indian Patent Application No.: 202521024259 (Filed)

## Work Experience
- Qure.AI | AI Scientist Intern [May '24 - Present]
  18+ months contributing to the lung cancer team working on diffusion models, resulting in 2 publications and a patent.
  Developed synthetic chest X-ray generation pipelines for training while preserving subtle yet clinically critical features.
- Uptrain.AI (YC W23) | AI Research Intern [Jan '24 - Apr '24]
  Contributed to advancing UpTrain's core evaluation platform for custom LLMs, designing novel subjective metrics.
  Worked closely with the founder to design the subjective metrics, and authored technical blogs on LLM evaluations.
- UnScript.AI | Deep Learning Engineer Intern [Nov '23 - Dec '23]
  Worked on diffusion-based text-to-speech models for Hindi, advancing speech synthesis quality, latency, and accuracy.
  Trained text-to-speech models and achieved effective speaker voice cloning using minimal audio sample data.

## Teaching Assistant Experience
- ES-682: Numerical Methods for Environmental Systems | Prof. Amritanshu Shriwasta [Spring '26]
  Assisted in logistics and doubt clearing for the class covering numerical methods, ODEs/PDEs, and stability analysis.
- ES-201: Applied Environmental Microbiology and Ecology | Prof. Swantantra Pratap Singh [Autumn '25]
  Contributed to course delivery for a class of 50+ students by managing logistics and assisting with evaluation activities.

## Previous Research Assistant Experience
- Research Assistant with [Prof. Saket Choudhary (KCDH, IIT Bombay)](https://saket-choudhary.me/about)
  Worked on research problems at the intersection of computational biology and AI for health.

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
