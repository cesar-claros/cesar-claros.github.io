---
layout: page
title: Brain age from MR elastography
description: 3D convolutional networks that relate whole-brain stiffness, damping ratio, and volume maps to age.
img: assets/img/projects/brain_age_maps.png
importance: 4
related_publications: true
---

Magnetic resonance elastography (MRE) measures the mechanical properties of brain tissue, which change across the lifespan and with neurodegenerative disease. With [Curtis Johnson](https://mechneurolab.squarespace.com/)'s Mechanical Neuroimaging Lab we pooled MRE data from several studies covering children through older adults and trained 3D convolutional networks (ResNet-34 and SFCN) to predict chronological age from stiffness, damping ratio, and volume maps {% cite claros2024elastography clements2023mechanical %}. Combining stiffness and volume gives the most accurate brain age estimates.

<div class="row justify-content-sm-center">
    <div class="col-sm-10 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/projects/brain_age_cnn.svg" title="Multimodal 3D CNN" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Multimodal 3D convolutional network taking stiffness, damping ratio, and volume maps as input.
</div>

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/projects/brain_age_predictions.png" title="Predicted versus chronological age" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Predicted versus chronological age for ResNet models trained on stiffness, volume, and their combination.
</div>

The work was featured by UDaily in [How old is your brain?](https://www.udel.edu/udaily/2025/march/brain-stiffness-age-dementia-alzheimers-curtis-johnson-austin-brockmeier/) Code: [MRE_MRI_BrainAge](https://github.com/cesar-claros/MRE_MRI_BrainAge) and [brain_age](https://github.com/cesar-claros/brain_age).
