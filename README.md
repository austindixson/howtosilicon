# How To Silicon

A [howtospark.com](https://howtospark.com)-inspired site for **Apple Silicon** local LLM agent recipes — with and without the Ferroclaw **context memory engine**.

## Open locally

```bash
open /Users/ghost64/Desktop/howtosilicon/index.html
# or
cd /Users/ghost64/Desktop/howtosilicon && python3 -m http.server 8765
# then http://127.0.0.1:8765
```

## Contents

| Path | What |
|------|------|
| `index.html` | Home: picks, ON vs OFF table, benchmarks |
| `recipes/` | Daily driver ON, control OFF, long-horizon audit |
| `blog.html` | Full build narrative (from Desktop markdown) |
| `Building-Ferroclaw-Context-Memory-Engine.md` | Same blog as source |

## Measured headline (Mac32 · Qwen 35B 4-bit)

| Arm | Multiturn tokens | Wall |
|-----|----------------:|-----:|
| Engine ON | 6 889 | ~23 s |
| Engine OFF | 35 250 | ~129 s |

See private repo: https://github.com/austindixson/ferroclaw
