# Building a Context/Memory Engine for Long-Horizon Agents on Mac Silicon

**Subtitle:** Context is a cache. Disk is memory. Advertised context is display-only.  
**Stack:** Ferroclaw (Rust harness) + optional hermes-context-memory MLX quarantine  
**Hardware:** Apple Silicon 32 GB · Qwen3.6-35B-A3B-4bit (MLX)  
**Repo:** [github.com/austindixson/ferroclaw](https://github.com/austindixson/ferroclaw) *(private)*  
**Status date:** 2026-07-20 (updated same day — **WORKING: orca-coder full audit ~1 minute**)  
**Tip of `main`:** `54bbaec` (Hermes2-style iteration finalizer)  

This is the honest build log: original problem, hypotheses, remedies, tests, failures, revised hypotheses, and major successes through orca-coder full audit — **plus critical analysis of product work** (short-task path, stop UX, discovery) — **and the production upgrade that made orca-class audits complete in about one minute** on local Qwen 35B / Mac32.

---

## 0. What we were trying to do

Run **long, tool-heavy coding agents** on a **local** 35B-class model on a **Mac with 32 GB unified memory**—audits, multi-file discovery, multi-turn tool loops—without:

1. Host panics (`IOGPUMemory` / Metal completeMemory underflow)  
2. Silent quality collapse when the prompt “looks fine” but KV is dead  
3. Cloud-only crutches (65k advertised context as if it were free)  
4. Hard-stopping audits with empty errors or fake **PARTIAL** reports  

The product name for the harness became **Ferroclaw**. The survival layer became the **context memory engine**. Optional process-level survival on MLX stayed in a **Python package** (`hermes-context-memory`), deliberately *not* patching a daily Hermes Agent install.

---

## 1. Original problem

### Observed failure modes

| Symptom | What it felt like | What it actually was |
| --- | --- | --- |
| Agent “dies” mid-audit | Model stupid / hung | Prompt + KV outgrew local envelope |
| `completeMemory() prepare count underflow` | Random crash | Metal IOGPU cliff (~weights + KV) |
| Quality falls off a cliff late in a session | Hallucinations | Context still “fits” textually; protocol or budget already broken |
| Empty ERROR stop | Harness bug | Budget paths hard-stopped with no artifact |
| Invented file paths thrash | Bad model | No discovery hygiene; refuse logic false-positives |
| **PARTIAL / tool budget reached** tombstones | “Finished” | Tool counters used as circuit breakers |

### Physics (32 GB Mac, Qwen 35B 4-bit)

Approximate envelope we treated as law:

- Weights alone ≈ **19 GB**  
- Comfortable Metal working set ≈ **~26–27 GB**  
- Headroom for KV + activations ≈ **7–8 GB**  
- “Advertised 65k context” is **display-only** for local hard budgets  

Filling an advertised cloud window on this hardware is not a quality experiment—it is a path to host death.

---

## 2. Original hypothesis and remedies

### North-star theory (became design law)

| # | Statement | Remedy |
| --- | --- | --- |
| T1 | Local death is **memory physics**, not model stupidity | Hard local ceilings; refuse or pack before dispatch |
| T2 | **Tool bulk** (not chat prose) is the main growth term | Externalize large tool results → evidence capsules on disk |
| T3 | **Protocol integrity** is reliability | Turn-safe eviction; never orphan tool_call / tool_result pairs |
| T4 | **Harness ownership** beats host patching | Engine lives in Ferroclaw; Hermes stays stock |
| T5 | **Schema tax** is fixed cost every turn | Compact wire tools + WireOnlyHint DietMCP |

### First remedies we built

1. **Context budget** with three numbers always: advertised / safe_context / usable_input  
2. **Evidence store** — digests in-prompt, raw on disk, retrieve by id  
3. **Checkpoints + resume** injected into the *user* message (not mid-history system spam)  
4. **MLX quarantine supervisor** (optional) — restart before residency cliff; pressure scale target_input  
5. **Mac32 profile** — safe_context 10240, target ~6400, hard_local_budget  

---

## 3. Test matrix — what we measured

### Offline engine (no GPU)

From harness reports (`LOCAL_LLM_HARNESS_LATEST.md`):

| Test | Result | Meaning |
| --- | --- | --- |
| A1 externalize 50kb tool | PASS | Bulk leaves the prompt |
| A2 tool-heavy trajectory | PASS | ~108k raw → fitted ~6100 under ceiling |
| A3 turn-safe pairs | PASS | No orphan tool pairs |
| A4 hard refuse | PASS | Oversized protected context refused (later: soft pack) |
| A8 mid-file fact capsule | PASS | `FACT 19427` survives externalize |
| A9 error tail capsule | PASS | Error lines kept in capsule |
| A10 resume checkpoint | PASS | Resume block reconstructible |

### Live ON vs OFF (same model, same tool seed)

| Arm | Multiturn prompt tokens | Wall time | Externalized |
| --- | ---: | ---: | ---: |
| Engine **ON** | **6 889** | ~23 s | 12 |
| Engine **OFF** | **35 250** | ~129 s | 0 |

**OFF/ON ≈ 5.12× tokens, ≈ 5.7× wall.**  

Quality probe: mid-file ground truth recovered at **100%** from capsules (ON) with far fewer input tokens than full-file OFF.

### Ultimate eval gates

- Offline corpus recovery **12/12**  
- Tool wire bytes: compact core ≈ **20%** of full tool JSON  
- Live arms gated when GPU available  

---

## 4. Failures (load-bearing — do not re-litigate without new data)

### F1 — Patch Hermes as the production path

**Tried:** inject context engine into Hermes Agent via plugins/patches.  
**Failed:** broken daily configs, plugin footguns, “repair install” required.  
**New law:** Ferroclaw owns the harness. Python package = MLX supervisor + reference math only.

### F2 — Soft compression that gives up

**Tried:** stop compressing after N ineffective attempts even if still over ceiling.  
**Failed:** oversized prompts still hit the backend → panic risk.  
**New law:** over-budget never disables the gate.

### F3 — Advertised context as input budget

**Tried:** use 65k / “safe context” as input target without reserves.  
**Failed:** no room for completion + tools; estimates lied.  
**New law:** usable_input = safe − output_reserve − tool_reserve.

### F4 — Head/tail digests drop the middle

**Tried:** naive trim of large tool results.  
**Failed:** lost mid-file facts and error tails.  
**Fix:** smart capsules (markers, paths, error tails). Offline A8/A9 guard this.

### F5 — DietMCP double-tax

**Tried:** full tool catalog in system prompt *and* wire tools[].  
**Failed:** hello tax ~5.8k → compact+WireOnlyHint ~1.3k.  
**Fix:** wire schemas authoritative; catalog off by default on local.

### F6 — Unfair OFF comparisons

**Tried:** OFF arm with fewer tools “because legacy.”  
**Failed:** fake quality wins.  
**Fix:** matched tool seed and history bulk for ON vs OFF.

### F7 — Mid-history system “we compacted” messages

**Tried:** insert system notices on prune.  
**Failed:** local chat templates break.  
**Fix:** mutate leading system or inject resume in user message.

### F8 — Orphan tool pairs

**Tried:** drop oldest messages by token count.  
**Failed:** tool_call without result → hard local failures.  
**Fix:** turn-safe eviction + property tests.

### F9 — Instruments on a saturated MLX process

**Tried:** `footprint --sample` near ceiling.  
**Failed:** tipped host over IOGPU cliff.  
**Fix:** supervisor residency JSON only.

### F10 — Path / model / OAuth routing

GGUF absolute paths, mlx-community ids, Application Support vs `~/.config` OAuth.  
**Fix:** explicit local profiles + sync scripts.

### F11 — Thinking mode on tight max_tokens

**Tried:** Qwen thinking for quality.  
**Failed:** burned completion budget; empty-looking finals.  
**Fix:** `enable_thinking = false` on Mac32 defaults.

### F12 — Docs pile as false progress

Remotion, glitter-verbs, “COMPLETE” markdown noise.  
**Fix:** keep measured reports + theories log; delete the rest.

---

## 5. Product failures on the *agent UX* path (repair sprint)

These came later, while driving real audits (orca-coder, screenshots, TUI).

### F13 — Discovery thrash / invented modules

Model guessed `Sources/Foo/Bar.swift` paths that never existed.  
**Fix:** DiscoveryTracker — list/glob confirm paths; thrash lock on **true ENOENT only**; soft steers (not red errors).

### F14 — False REFUSED of real files

Lock treated “unconfirmed” as refuse even when file existed.  
**Fix:** **stat before refuse**; existing files auto-confirmed; parent-listed children allowed.

### F15 — Refuse → path_miss cascade

Soft steers counted as misses → lock tightened forever.  
**Fix:** discovery_steer is not path_miss.

### F16 — Two budgets confused in the UI

Run token counter vs CM dispatch ceiling looked like one meter.  
**Fix:** plain-language status: “this turn tokens” vs “prompt % full” + Explore DISC%.

### F17 — Report cut mid-sentence

`max_tokens` 1536 truncated audits.  
**Fix:** floor / config **4096** for finish turns.

### F18 — Hard refuse “used 10085 of 7783”

CM hard-stop empty-handed.  
**Hypothesis then:** always emit *something*.  
**Bad remedy:** offline **PARTIAL** artifacts + tool-cap soft-finish.  
**User rejection:** “I do not want partial reports, I do not want tool limits.”

### F19 — PARTIAL / tool-budget tombstones

**What shipped briefly:** when tool budget or iterations hit, finish with:

```text
# PARTIAL report
**Reason:** tool budget reached …
completed_partial:tool_budget
```

**Why it failed product-wise:** looks like completion; is not an audit. Trains the system to quit.  

**Why it failed research-wise:** tool count is not a measure of task completion.

---

## 6. New hypothesis after PARTIAL rejection

### H-new-1 — Tool count is not a circuit breaker

**Prediction:** unlimited tools + context packing alone can finish long audits.  
**Partial result:** first re-run listed **250+** directories and never wrote a report (thrash without finish pressure).  
**Revision:** unlimited tools is necessary but not sufficient.

### H-new-2 — Completion is a *steer*, not a *budget*

**Prediction:** progressive report-completion steers (nudge → pack → optional text-only finish attempt) produce a real multi-section report without PARTIAL or tool caps.  
**Result:** **Confirmed** on orca-coder-main (~5 min headless run): full senior-dev report, no PARTIAL, no tool-budget note. Saved as `docs/reports/orca-coder-audit.md`.

### H-new-3 — Pack + continue beats pack + quit

**Prediction:** emergency_collapse under local ceiling should keep the loop alive with full tools (or deliberate text finish), never offline tombstone.  
**Result:** **Implemented permanently** in AgentLoop + ContextMemoryEngine.

---

## 7. Successes (chronological, condensed)

| When | Success | Evidence |
| --- | --- | --- |
| Early | Engine ON ≪ OFF tokens | 6889 vs 35250 multiturn |
| Early | Capsule fact recovery 100% | FACT 19427 probes |
| Early | Tool wire tax crushed | core ~20% of full bytes |
| Mid | Mac32 profile + MLX quarantine | soak / residency JSON |
| Mid | Discovery hygiene | no false refuse of real files |
| Mid | Hermes slate + Awal-inspired chrome | status bar readable |
| Mid | Soft pack under ceiling | no empty ERROR on CM pressure |
| Late | Unlimited tools (max_tool_calls_total=0) | no tool-budget kill switch |
| Late | Progressive completion steers | real orca audit report |
| Late | 407 lib tests green through all of above | `cargo test --lib` |

---

## 8. Architecture that stuck

```
User ──▶ Ferroclaw AgentLoop
              │
              ├─ DietMCP (compact wire tools)
              ├─ DiscoveryTracker (list → glob → read hygiene)
              ├─ ContextMemoryEngine
              │     ├─ budget + pressure (mac32 envelope)
              │     ├─ externalize → evidence capsules
              │     ├─ turn-safe fit / checkpoint / resume
              │     └─ emergency_collapse → pack + continue
              ├─ Progressive completion steers (report synthesis)
              └─ OpenAI-compat provider → local MLX Qwen (:8080)
                    └─ optional quarantine supervisor
```

### Runtime knobs (live Mac32-ish)

| Knob | Value | Role |
| --- | --- | --- |
| `max_tool_calls_total` | **0** | Unlimited lifetime tools |
| `max_iterations` | 500 (soft) | Extend / pack / continue |
| `token_budget` | 500k (soft, expands) | Not a PARTIAL tombstone |
| `max_tokens` | 4096 | Finish reports not mid-sentence |
| CM `safe_input` / ceiling | ~7.7–8.2k usable | Local physics |
| Completion steers | permanent in `loop.rs` | Nudge → pack → text attempt |

---

## 9. Progressive report-completion steers — are they permanent?

**Yes.** They are permanent product code in `src/agent/loop.rs` (not a one-off script), shipped on `main`:

- Track discovery signal + tool count  
- Every ~18 tools after “enough signal”: synthesis nudge  
- Strong stage: emergency pack + “stop bulk discovery, write report”  
- Up to 3 **text-only** finish attempts when tools≥90 or reads≥40  
- Accept only if `looks_like_full_report` (multi-section, rejects PARTIAL/tool-budget stubs)  
- Thin drafts fall through — **full tools resume**  
- **Never** offline PARTIAL quit on tool budget  

Commits (representative):

- `948836d` — unlimited tools, pack-and-continue, no PARTIAL tool-budget tombstones  
- `b6810d7` — progressive completion steers  
- `7934393` — orca-coder full audit doc  

---

## 10. Remedies that *failed product* even when they “worked”

| Remedy | “Worked” how | Why we ripped it out |
| --- | --- | --- |
| PARTIAL finish_artifact | Always non-empty exit | Fake completion; user-hated |
| Tool budget soft-finish | Forced a stop | Task ≠ tool count |
| Force synthesize + strip tools | Stopped thrash | Premature; blocked needed reads |
| Hard CM refuse ERROR | Saved the host | Empty-handed operator |

The durable pattern is: **pack context, keep going, steer completion, accept only real reports.**

---

## 11. What “done” looked like mid-day (before the thrash sprints)

1. Local long-horizon agent on Mac Silicon with engine ON  
2. ~5× fewer multiturn tokens vs engine OFF on matched bulk  
3. Discovery hygiene that does not refuse real files  
4. No PARTIAL / tool-budget terminal outcomes  
5. Full senior-dev audit of `orca-coder-main` written as a real report  
6. Private GitHub repo with measured docs (`THEORIES_AND_RESULTS.md`, harness reports)  

Companion site concept: **How To Silicon** — curated Mac Silicon recipes and ON vs OFF context-engine benchmarks (see Desktop `howtosilicon/`).

That list was **necessary but not sufficient**. The afternoon runs proved a second product surface: **control, task class, and stop**.

---

## 12. Critical analysis of recent work (same day, later commits)

This section is intentionally hard-edged. The earlier chapters celebrate survival physics. This one grades **operator experience** after we “solved” PARTIAL.

### 12.1 Timeline of the latest commits (tip → older)

| Commit | Intent | Grade (honest) |
| --- | --- | --- |
| `f8ec670` | Docs-first discovery; soft-block `node_modules` / `.build` / bulk reads | **B+** — correct product law; needs live soak |
| `897130a` | Ceiling-aware short path; synth strips tools; stuck Rotate×3; sticky class; checkpoint de-dupe; cancel not ERROR | **A−** — fixes root causes of 17m thrash |
| `906167c` | Esc / Ctrl+C actually stop a running turn | **A** — should have existed before unlimited tools |
| `d793a40` | Short-task fast path (classify, allowlist, write-now) | **B** — right idea; first externalize knob was wrong |
| `948836d` / `b6810d7` | Unlimited tools + pack-continue + completion steers; no PARTIAL tool-budget | **B+** for long audits; **D** for short tasks until sticky/synth fixed |
| `be29278` | Soft pack / force synthesize / never empty ERROR | **C** — survival yes; PARTIAL + full-tools-under-synth seeded thrash |

### 12.2 What went well

**Cancel is product, not polish.**  
Before `906167c`, every key during a run printed *“Agent is still running — wait…”* — including Esc and Ctrl+C. After a 399-tool, 16m51s disaster, cooperative cancel returned `Interrupted` with `user_cancel`. That is the correct contract: **operator sovereignty > model momentum**.

**PARTIAL was the right enemy.**  
Ripping tool-budget / iteration soft-finish tombstones was correct. Users do not want fake completion. Measuring success by a real multi-section report (orca-coder) is the right north star for long-horizon.

**Task class is the right abstraction.**  
`ShortFile` vs `LongHorizon` vs `ScopedCode` is how you avoid paying audit tax on a blog critique. Sticky class across `1A 2B 3A` clarify answers was a necessary patch after classify(steer letters) → `scoped` reopened full tools.

**CM still does its job under pressure.**  
Status stayed “Healthy” at ~32% prompt fill while the **agent loop** was unhealthy. That split is good engineering (two meters) and bad UX (operators read Healthy as “fine”). Recent work is learning to show **rounds / tools / stuck** as first-class, not only CM score.

**Discovery noise policy is overdue.**  
Ignoring `node_modules`, `.build`, `target/`, vacuum globs, and large non-core full-reads is how you stop 339× `read_file` “research.” Docs/README/filenames first, then grep for TODO/FIXME/doc comments, then full-read only core entrypoints — that is senior-dev methodology, not a vibe.

### 12.3 What went badly (and what we learned)

**F20 — Short-path externalize set to 200k “keep blog inline.”**  
Predicted: one HTML stays readable.  
Actual: est ≈ **11–12k tokens > ceiling ≈ 7.8k** forever; `externalized=0`; **367 checkpoints** in ~22 minutes; evidence folder empty.  
Lesson: **never raise externalize above dispatch physics.** Short path must capsule *earlier*, not later.

**F21 — `force_synthesize` + “continuing with full tools.”**  
UI said `synth=true` while the model still had list/read. That is a **lie to the controller**. Synthesize pressure without tool stripping is an invitation to thrash. Fixed: short → text-only; long → evidence/read only under synth; stuck Rotate×3 forces text path.

**F22 — Unlimited tools without stop.**  
Shipping max_tool_calls_total=0 *before* Esc cancel was a sequencing error. Long-horizon freedom without a kill switch is hostage-taking. Order should have been: **cancel → then unlimited**.

**F23 — Clarify steers destroy task class.**  
Blog critique correctly classified `short`, then operator answered `1A 2B 3A` (or similar) and the next turn reclassified as `scoped` with full tools — 359+ tools, 343 rounds, 900s+ wall.  
Lesson: **MC answers are not a new task.** Sticky class + re-check CM goal.

**F24 — Checkpoint spam as false progress.**  
Hundreds of identical `(Rotate, est=11488, externalized=0)` events looked like work. They were not. De-dupe is mandatory for any “always pack” design.

**F25 — Status theater.**  
“Healthy · Explore: OK · 0 misses” while round 343 and turn tokens 773k/500k. CM health ≠ task health. Operators need a **progress** meter (rounds, tools, stuck flag), not only a **context** meter.

**F26 — This blog as product surface.**  
HowToSilicon / `blog.html` became the *test fixture* that exposed short-path failure. Meta lesson: dogfood your docs as short tasks continuously, not only long audits.

### 12.4 Critical product judgment

| Surface | Mid-day claim | Later reality | Now (after repairs) |
| --- | --- | --- | --- |
| Long audit | No PARTIAL; pack+continue | Orca report real | Still the win case |
| Short critique | “Unlimited tools is fine” | 17m thrash, 399 tools | Sticky short + ceiling externalize + text under synth |
| Stop | — | Impossible (key swallow) | Esc / Ctrl+C cancel; Interrupted not ERROR |
| Discovery | List/glob/read hygiene | Vacuum + full-file dumps | Noise soft-block + docs-first policy |
| Observability | CM score | Events almost only checkpoints | Still weak on tool histograms — backlog |

**Overall grade for the afternoon arc: B.**  
We fixed the right things under pressure, in the right order *once thrash was undeniable*, but we shipped **long-horizon freedom without short-horizon discipline and without stop**, which is how you burn a 35B session and an operator’s patience.

### 12.5 What “done” means *now* (revised)

1. Long-horizon: unlimited tools, pack+continue, completion steers, **no PARTIAL tool-budget**  
2. Short-horizon: sticky ShortFile, ceiling-aware externalize, few tools, write-now  
3. Under pressure: **synthesize means fewer tools**, not full tools  
4. Operator: Esc stops; second Esc force-aborts; cancel is not painted as ERROR  
5. Discovery: skip build/vendor noise; README/docs/filenames first; grep dev comments; full-read only core  
6. Still open: tool-level event telemetry, MLX quarantine UX, stuck auto-cancel after N minutes  

### 12.6 Recommendations (next engineering, prioritized)

1. ~~**Live soak** short blog critique + orca-style audit~~ → **done** for orca: ~**82s** full report (see §15).  
2. Log `tool_batch` / `llm_round` into `events.jsonl` — **done** in 0.3.0.  
3. Status bar: `rounds · tools · stuck?` — **done** in 0.3.0.  
4. Align MLX server `--max-tokens` with Ferroclaw `max_tokens` (still worth watching).  
5. Auto-cancel after N identical Rotate signatures or wall > T for ShortFile — **done** for short path.  
6. Keep HowToSilicon / this blog as a **CI short-task fixture**, not only narrative.

---

## 13. Open questions (honest backlog)

- Multi-hour idle soak under MLX quarantine still optional measurement  
- Evidence-first scoring for findings that over-infer from path names  
- Public HowToSilicon with multi-Mac profiles (M1–M4 · 16/32/64/128 GB)  
- Whether “dev comments only” should become a **grep-first hard phase** before any full read (policy yes; hard gate optional)  
- Message-only context budget accounting (tool schema weight) — residual from Grok-local research  

---

## 14. One-paragraph thesis (updated)

Long-horizon local agents do not fail because 35B models are “dumb.” They fail because the **live prompt is treated as durable memory** on hardware that cannot host cloud-sized KV — and because **task class and operator control** were treated as afterthoughts to survival. Ferroclaw’s context memory engine makes **disk the system of record** and **context a cache**. After rejecting PARTIAL tool-budget tombstones, the durable runtime is **unlimited tools + pack-and-continue + completion steers for long work**, **sticky short-path + ceiling-aware externalize + text-under-pressure for small work**, **docs-first discovery that refuses build noise**, **Esc that actually stops**, and a **Hermes2-style iteration finalizer** that forces a real multi-section deliverable once exploration has paid for itself. Measure success by **real reports in minutes, not thrash forever** — and we now have the proof: **orca-coder full senior-dev audit completed in about one minute**.

---

## 15. IT WORKS — Orca audit completed in ~1 minute (finalizer upgrade)

### 15.1 The product win (plain language)

**Ferroclaw is working.** Not “survival theater,” not a PARTIAL tombstone, not a 17-minute thrash that never ships a report.

On **2026-07-20**, headless run against `/Users/ghost64/Desktop/orca-coder-main` with local **Qwen3.6-35B-A3B-4bit (MLX)** on **Mac 32 GB**:

| Metric | Result |
|--------|--------|
| **Wall clock** | **~82 seconds (~1.4 min)** — under the 5-minute bar; effectively a **~1-minute-class** full audit |
| **Exit** | `0` (success) |
| **Deliverable** | Real multi-section senior-dev report: overview, architecture, risks, ≥5 findings with paths, recommendations |
| **PARTIAL / tool-budget stop** | **None** |
| **Exploration footprint** | Listed ≤7 · opened ≤5 (docs-first, noise-skipped) — not 300+ directory walks |
| **Binary** | `ferroclaw 0.3.0` · commit **`54bbaec`** |
| **Log** | `docs/reports/orca-finalizer-run.log` |

That is the north star we set after ripping PARTIAL: **a real audit report, fast enough to dogfood, without operator-prompt gymnastics.**

### 15.2 What we upgraded (Hermes2-style finalizer)

Research across hermes2 / grok-local / context-engine forks agreed on one pattern we were under-enforcing:

> **Bound exploration → externalize bulk → tools off → force a real answer. Never offline PARTIAL as “success.”**

Ferroclaw 0.3 finalizer (`src/agent/finalizer.rs` + loop wiring):

1. **Task class** detects codebase / critically-analyze work → scoped path with soft budgets (tools / rounds / wall).  
2. **Discovery hygiene** skips `node_modules` / `.build` / `target`; prefers README/docs before full-file thrash.  
3. When soft budget, wall, steer thrash, or stuck Rotate hits → **iteration finalizer arms**:  
   - tools **stripped** (`tools=[]`)  
   - last-response prompt demands multi-section Markdown  
   - rejects “let me inspect more” / PARTIAL titles / pure continuation prose  
   - **one retry**, then best-effort if still a long non-stall draft  
4. Operator still owns the process: **Esc / Ctrl+C** cancel works (`Interrupted`, not red ERROR theater).

### 15.3 Why this is “intuitive by default”

The goal is not a faster stopwatch on a hand-tuned script. The goal is:

- **Natural prompts work** — “critically analyze this codebase” should not require a 20-line discovery manifesto.  
- **Task decomposition is automatic** — classify short vs scoped vs long; sticky class; soft ceilings.  
- **Useful work first** — list structure, read docs/entrypoints, skip noise, externalize bulk under Mac32 ceiling.  
- **Finish by default** — when enough evidence is in context (or soft budget hits), **write the report** instead of rotating forever.

If the operator has to “mess with parameters” or paste verbose steers for every audit, we failed the product. The finalizer + task class + discovery policy are how we make **default behavior** the fast path.

### 15.4 Timeline honesty (same day)

| Stage | Outcome |
|-------|---------|
| Mid-day orca (completion steers, no finalizer) | Full report, ~**5 min** — good, not snappy |
| Short-task thrash era | Blog critique → **17m / 399 tools** — product failure |
| Cancel + sticky short + synth honesty | Operator control restored |
| **Finalizer upgrade (`54bbaec`)** | Orca **~82s**, EXIT 0, real report — **working** |

### 15.5 What “completed” means

- Multi-section structure present (overview / architecture / risks / findings / recommendations)  
- Real paths from the tree (not invented `scripts/` confs)  
- No `# PARTIAL report` / “tool budget reached” terminal  
- Wall under a few minutes on local 35B without cloud context  
- Reproducible binary: `ferroclaw 0.3.0` on `main`

**Bottom line:** the context/memory engine + production harness is **not a science project anymore for the orca-class job.** It runs, it finishes, and it finishes in about a minute.

---

*Written 2026-07-20; §12–14 after cancel/thrash/discovery (`906167c`…`f8ec670`); **§15 after finalizer upgrade + live orca SUCCESS ~82s (`54bbaec`)**. Sources: `docs/FORK_RESEARCH_HERMES2_GROK_LOCAL.md`, `docs/PRODUCTION_READY_SPRINT.md`, `docs/reports/orca-finalizer-run.log`, live MLX Qwen35B Mac32.*
