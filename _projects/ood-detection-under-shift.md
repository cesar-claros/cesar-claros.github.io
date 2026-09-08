---
layout: page
title: Out-of-distribution detection under distribution shift
description: Which OOD detector to trust depends on the learned representation. A systematic benchmark across CNN and ViT backbones, training paradigms, and shift regimes, plus worst-case calibration error bounds.
importance: 1
related_publications: true
---

Confidence score functions (CSFs) for out-of-distribution (OOD) detection are usually compared on a fixed backbone, which hides how much the answer depends on the representation. In {% cite claros2025systematic %} we benchmark CSFs across CNN and ViT backbones, several training paradigms, four source datasets (CIFAR-10, CIFAR-100, SuperCIFAR-100, TinyImageNet), and OOD sets grouped into near, mid, and far regimes by CLIP-derived semantic distance. A multiple-comparison-controlled ranking pipeline identifies cliques of statistically indistinguishable winners under AURC and AUGRC.

The competitive detector family depends more on the learned representation than on the score itself. Neural Collapse metrics of the last-layer representation explain the ranking shifts: prototype- and boundary-aware scores win when the representation is more collapsed and aligned with the classifier weights, while weaker-collapse regimes favor gradient- and manifold-based scores. From this we derive a PCA-based projection-filtering step that improves detectors, and a predictor that recommends a competitive detector shortlist for a trained classifier without any OOD validation data.

A companion workshop paper {% cite claros2026bounding %} bounds the worst-case calibration error of OOD detectors under distribution shift.

Code: [cesar-claros/ood_systematic](https://github.com/cesar-claros/ood_systematic).
