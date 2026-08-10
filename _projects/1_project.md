---
layout: page
title: vit-clustering
description: Adaptive token clustering for DeiT/ViT in PyTorch.
img:
importance: 1
category: Research
github: https://github.com/dingkai0321/vit-clustering
---

`vit-clustering` is the code-facing project behind my University of Sydney thesis on improving the efficiency of Vision Transformers by reducing redundant patch tokens without discarding useful information too aggressively.

This work was carried out under the supervision of [Dr. Karlos Ishac](https://karlosishac.com/).

The core method combines three ideas:

- Use class-to-patch attention as a lightweight importance signal.
- Choose an image-specific token budget instead of keeping a fixed number of tokens.
- Merge redundant tokens into retained centers with feature-position clustering and importance-aware weighting.

The evaluation covered `DeiT-Tiny` and `DeiT-Small` on `ImageNet-100` and `CIFAR-100` with coverage ratios `0.8`, `0.6`, and `0.4`. The main result was consistent across datasets: moderate compression preserved accuracy best, while more aggressive compression improved theoretical compute at the cost of larger information loss.

Two practical findings shaped the conclusion:

- On `DeiT-Small`, shorter token sequences could improve throughput in batch inference.
- On `DeiT-Tiny`, adaptive clustering overhead dominated the savings, so real latency did not improve.

The thesis argues that efficient ViT design should be evaluated beyond GFLOPs, with latency, throughput, memory behavior, and hardware friendliness considered together.

[GitHub Repository](https://github.com/dingkai0321/vit-clustering)

[Read thesis PDF]({{ '/assets/pdf/kai_ding_thesis.pdf' | relative_url }})
