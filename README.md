# How To Silicon

Mac Apple Silicon local LLM agent recipes, Ferroclaw context-engine benchmarks, and progressive limit-campaign reports.

## Live site

**https://austindixson.github.io/howtosilicon/**

| Page | URL |
|------|-----|
| Home | https://austindixson.github.io/howtosilicon/ |
| Build log | https://austindixson.github.io/howtosilicon/blog.html |
| **Limit campaign report** (interactive) | https://austindixson.github.io/howtosilicon/limit-campaign-report.html |

## Local

```bash
open index.html
# or
python3 -m http.server 8765
```

## Contents

| Path | What |
|------|------|
| `index.html` | Home: picks, ON vs OFF, benchmarks |
| `recipes/` | Daily driver ON, control OFF, long-horizon audit |
| `blog.html` | Ferroclaw build narrative |
| `limit-campaign-report.html` | Grok × Qwen pad ladder + dogfood + dual-repo interactive report |

## Headline results (Mac32 · Qwen 35B 4-bit)

| Arm | Multiturn tokens | Wall |
|-----|----------------:|-----:|
| Engine ON | 6 889 | ~23 s |
| Engine OFF | 35 250 | ~129 s |

Limit campaign (2026-07-21): pad 24k → **28/28 recall**; dogfood monorepo audit pass; dual-repo pass under tool cap.
