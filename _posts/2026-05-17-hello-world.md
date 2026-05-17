---
layout: post
title: "Hello, world"
date: 2026-05-17 17:45:00 +0800
categories: [meta]
---

This is the first post on `blog.higcp.com`. The blog is built with Jekyll
on GitHub Pages, with a custom skin mimicking Google Cloud Console design:
white background, Google Sans typography, Google Blue accents, no
gradients, no decorative emoji.

## What I'll write about

- **TPU v7 (Ironwood)** — training and inference experience: model
  loading, checkpoint conversion, sharding strategies, performance
  optimization.
- **GPU inference** — vLLM and SGLang deployment notes: model registration
  quirks, MoE prefetch deadlocks, KV cache tuning, FP8/FP4 trade-offs.
- **Multi-agent systems** — running multiple LLM-powered bots on the same
  infrastructure, IPC patterns, debugging cold-path bugs.
- **Cloud infra** — GCP Cloud DNS, GKE topology, Cloud Storage gotchas,
  cross-project IAM headaches.

## Why Jekyll

Three reasons:

1. **Markdown all the way down.** Source files are just `.md` text under
   `_posts/`. No CMS, no DB, no auth. `git push` is the publish button.
2. **GitHub Pages handles hosting + HTTPS.** Let's Encrypt certificate
   provisioned automatically for the `blog.higcp.com` custom domain. Zero
   server maintenance.
3. **The default theme `minima` is solid.** With a small SCSS override
   file (`_sass/gcp-overrides.scss`), it ports cleanly to Material Design
   without forking a heavy theme.

## Code style example

```python
import jax
import jax.numpy as jnp

@jax.jit
def matmul(x: jax.Array, y: jax.Array) -> jax.Array:
    return x @ y

# Trillium-class TPU (v5p) — 4096 chips, BF16
x = jnp.ones((8192, 8192), dtype=jnp.bfloat16)
y = jnp.ones((8192, 8192), dtype=jnp.bfloat16)
out = matmul(x, y)
print(out.shape)  # (8192, 8192)
```

## Tables for hardware specs

| Chip          | HBM       | BF16 TFLOPS | FP8 TFLOPS | Pod scale   |
|---------------|-----------|-------------|------------|-------------|
| TPU v5p       | 95 GB     | 459         | —          | 8,960 chips |
| TPU v7 (Ironwood) | 192 GB | ~2,307      | 4,614      | 9,216 chips |
| NVIDIA B200   | 192 GB    | 1,125       | 4,500      | per node    |

> Specs sourced from Google Cloud official announcement (Ironwood, 2025)
> and NVIDIA Blackwell datasheet.

That's it for now. More posts will follow as I write them.
