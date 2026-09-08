---
layout: page
title: Early prediction of cognitive decline
description: Which baseline measurements predict the transition from normal cognition to mild cognitive impairment or dementia? A prognostic pipeline prototyped on ADNI for the Delaware Longitudinal Study for Alzheimer's Prevention.
img: assets/img/projects/adni_longitudinal.png
importance: 2
---

The [Delaware Center for Cognitive Aging Research](https://sites.udel.edu/memory-research/) at the University of Delaware, with support from the Delaware Community Foundation, is expanding the Delaware Longitudinal Study for Alzheimer's Prevention (DeLSAP), a statewide observational study of how health, lifestyle, and biology shape dementia risk over time. While DeLSAP data collection is under way, we use the [Alzheimer's Disease Neuroimaging Initiative](https://adni.loni.usc.edu/) (ADNI) as a development dataset to prototype the modeling workflow and to identify which measurements are worth prioritizing in the DeLSAP protocol.

The prediction task starts from subjects who are cognitively normal at baseline and asks whether they later transition to mild cognitive impairment or dementia. Most published ADNI work targets the later MCI-to-dementia conversion; this earlier transition is the stage where prevention is still plausible. We compare three baseline representations of risk under one leakage-safe protocol:

- a compact LIBRA-style lifestyle risk score reconstructed from the factors ADNI records,
- a broader panel of modifiable risk factors (social, vascular, metabolic, behavioral), and
- biomarker, clinical, and cognitive assessments.

Progressors and stable subjects are paired by exact matching on sex and APOE genotype with age differences minimized by a binary linear program, and matched pairs are kept together in grouped cross-validation with paired bootstrap confidence intervals. The same question is then re-asked under a survival framing, with longitudinal exposure trajectories, and with volumetric MRI added as a third modality.

A manuscript is in preparation. Next steps include structural MRI features, pretrained tabular and imaging foundation models, and external validation on an independent cohort. Code: [dcf-adni](https://github.com/cesar-claros/dcf-adni).
