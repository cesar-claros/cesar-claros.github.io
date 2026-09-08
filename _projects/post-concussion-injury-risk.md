---
layout: page
title: Post-concussion musculoskeletal injury risk
description: An interpretable risk score for musculoskeletal injury after concussion in collegiate athletes, built from longitudinal clinical assessments.
img: assets/img/projects/concussion_timeline.svg
importance: 5
related_publications: true
---

Athletes who return to play after a concussion face an elevated risk of musculoskeletal injury, even after clinical recovery. With [Thomas Buckley](https://sites.udel.edu/buckley-concussion-research/)'s Concussion Research Laboratory we built a risk model from a longitudinal dataset of concussed student athletes at the University of Delaware, with demographic, medical history, and concussion assessment data collected at baseline, acute, asymptomatic, and return-to-play milestones {% cite anderson2024integrative %}.

Combining weight-of-evidence transformations with feature selection and logistic regression yields an interpretable risk score that reaches an AUC of 0.82 on a held-out test set {% cite claros2025concussion %}. The model is being explored for fall risk in Parkinson's disease and for aging research.

The work was featured by UDaily in [A game-changing tool](https://www.udel.edu/udaily/2025/april/kaap-concussion-artificial-intelligence-injury-prediction-athletics/). Code: [R21_risk_model](https://github.com/cesar-claros/R21_risk_model).
