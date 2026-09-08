---
layout: page
title: Max-sliced Bures distance and distribution shift
description: A tractable lower bound on the max-sliced Wasserstein-2 distance whose optimal slice acts as a witness function that pinpoints the instances driving a discrepancy between two samples.
img: assets/img/projects/max_sliced_bures_row.gif
importance: 6
related_publications: true
---

The max-sliced Bures distance is a lower bound on the max-sliced Wasserstein-2 distance between two distributions. Unlike heuristic algorithms for max-sliced Wasserstein, the optimal slice can be found globally and scales to large samples. The slice decomposes into two asymmetric divergences, each with a witness function that takes large values on a localized subset of instances in one sample versus the other {% cite brockmeier2021maxsliced %}.

At the NeurIPS 2021 Workshop on Distribution Shifts we used these witness functions to identify the instances associated with a shift and to correct covariate shift by reweighting {% cite brockmeier2021identifying %}.

<div class="row justify-content-sm-center">
    <div class="col-sm-12 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/projects/max_sliced_bures_row.gif" title="Reweighted flows under max-sliced Bures distances" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Reweighted gradient flows that move a sample (red) onto a target distribution (blue ring) under the max-sliced Bures distance, without and with random Fourier bases (linear and circular slicing).
</div>

Code: [sliced_bures_flows](https://github.com/cesar-claros/sliced_bures_flows).
