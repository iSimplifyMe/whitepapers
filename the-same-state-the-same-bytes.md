> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-same-state-the-same-bytes](https://isimplifyme.com/whitepapers/the-same-state-the-same-bytes)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-07-31 · License: [CC BY 4.0](LICENSE)

# The Same State, The Same Bytes: State Mixture Masquerading as Nondeterminism Under Full Control

The third study in this series ended with a number it could not explain: a cell that measured 0.60 in a shuffled pilot and a perfect 1.000 under confirmatory conditions, on identical requests. Six follow-up runs later, the explanation is measured and manipulated: the nondeterminism was never noise. It was a mixture of hidden serving states — and every one of them is deterministic.

---

## Abstract

> **Answer:** Six companion runs resolve the reproducibility tension [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) disclosed — 1,160 recorded calls against a digest-pinned open-weight model on owned hardware, every plan committed publicly before its data. The long-generation instability that looked like nondeterminism was a mixture of three prompt-cache states, each internally byte-deterministic at every sample size measured (modal share 1.0 at n up to 100 per state). Which state a call lands in is selected by instance history: a freshly loaded model instance serves one byte-stream; an instance that has answered even one different prompt serves another, permanently, until reload. That claim survived a registered manipulation — five fresh-instance cycles in which a single interposed different-prompt call flipped the output bytes at exactly that call, in every cycle, under a passing gate with zero overlap between state signatures. One boundary was found and is disclosed: the bytes each state produces held across four server sessions and then changed in a fifth, so per-state bytes are session-scoped — stable across five unload-reload cycles within a session, not guaranteed across server restarts. The engineering consequence is a state-pinning pattern for byte-reproducible long-form generation: both pinned regimes measured 100-for-100, on the output surface no validator can cover.

This is the mechanism paper the series has been walking toward. [*The Same Request Twice*](https://isimplifyme.com/whitepapers/the-same-request-twice) measured instability through one serving door; [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) attributed most of it to the serving path; [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) measured the ceiling under full control and disclosed the one tension full control did not close; [*The Same Object, Different Bytes*](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) showed which instability a validator absorbs — and named free prose as the surface no output gate covers. This paper closes the disclosed tension, and its answer lands on exactly that uncovered surface.

The companions are exploratory-directional, not pre-registered studies: each plan, gate, and endpoint was committed to the public repository before its data was collected, at addendum scale. Three of the five manipulation attempts voided their own gates. That iteration is reported in full, because the failures are what located the variable.

---

## 1. The Tension the Ceiling Left Behind

> **Answer:** In the third study's shuffled pilot, the cell `gpt-oss-20b | open generation | greedy | low effort` measured modal share 0.60 with three distinct outputs in ten calls on the Metal box — the only sub-ceiling greedy cell in any pilot. The confirmatory run measured the identical cell, identical request bytes, at 1.000: one hundred calls, one byte-stream, 811 output tokens every time. The study disclosed rather than resolved it: "memory churn between calls may itself perturb long generation, which would make swap-heavy serving environments a nondeterminism channel in their own right."

The disclosure was careful because the confound was real. The shuffled pilot differed from the blocked confirmatory run in two mechanically distinct ways at once: models were unloaded and reloaded between calls, and other models were competing for memory. Either could be the channel; so could small-n pilot noise. The fixture itself is the series' longest divergence surface — roughly 400 words of prose explaining why floating-point addition is not associative, generated greedily with a fixed seed on a digest-pinned gpt-oss-20b under Ollama 0.30.5.

The companions attacked the tension the way the series attacks everything: one manipulated variable at a time, a gate that voids the run if the manipulation did not physically happen, and the plan committed in public before the first call. Six runs were needed. Schedule noise and memory pressure both washed out. What remained was state.

---

## 2. The Inversion: Reload Does Not Degrade Determinism

> **Answer:** The first companion tested the obvious hypothesis — that unload/reload churn between calls degrades long-generation reproducibility — with 100 calls per arm per box: a `blocked` arm holding the model resident versus a `churn` arm that unloaded the model, confirmed its absence from the serving roster, and reloaded before every single call. The registered result came out backwards: churn-minus-blocked reproducibility was **+0.100 (CI95 +0.041 to +0.159), identical on both boxes**. The fresh-reload arm was byte-perfect, 100 of 100, on each machine. The undisturbed arm carried the instability.

The recorded per-call telemetry explains the inversion, and the explanation dissolves the original question. The blocked arm's 100 calls were not one condition — they were a mixture of prompt-cache states. On the CUDA box: ninety calls ran with a warm prefill (about 17 ms) and produced one variant, ninety for ninety; ten calls — the first block, right after the schedule's warmup — ran at an intermediate prefill (about 37 ms) and produced a second variant, ten for ten; and the churn arm's hundred cache-cold calls (prefill about 199 ms) produced a third, one hundred for one hundred. Three prefill classes, three byte-streams, *zero within-state variation anywhere*, with the two blocked-arm variants forking at character 230 and never earlier. Metal drew the same 90:10 blocked split with its own per-box signatures — its first-block calls ran near its fresh-load timing (~106 versus ~118 ms) and produced its fresh-load bytes, two byte-streams in total, forking at character 46 — a first sight of something the confirmation later sharpens: the state *structure* travels; the exact classes, timings, and bytes are per-box facts.

Read back against the pilot, the tension resolves without residue. The shuffled pilot's 0.60-with-three-variants cell was three cache states sampled in one bucket — each state plausibly deterministic even then. The blocked confirmatory run's 1.000 came from a schedule whose interleaved tasks kept the prefix cache in one state for all one hundred calls. Reload does not degrade determinism. **State mixture masquerades as nondeterminism** — and a reproducibility number computed across a state boundary measures the mixture, not the model.

---

## 3. Three States, Physically Identified

> **Answer:** The serving engine — Ollama 0.30.5's llama.cpp serving layer — names the states in its own logs, and prefill time separates them cleanly, by an order of magnitude at the extremes on the box where all three classes appeared. **Full evaluation**: no cached prefix, all 116 prompt tokens evaluated — 216 ms in the logged instance. **Full-KV reuse**: 112 of 116 prompt tokens restored from cache, 5 evaluated (the engine re-evaluates the boundary token) — 17.8 ms. **Template checkpoint**: a persistent 39-token cached prefix restored, the remaining 77 tokens evaluated — 46 ms in the logged instance, medians 33–37 ms across the measured runs, individual calls tailing to ~102 ms. Each state, every time it was sampled in five server sessions, produced exactly one byte-stream at that session.

The discriminator is free: `prompt_eval_duration` arrives on every response (in nanoseconds — `prompt_eval_duration_ns` in the response schema — quoted here in milliseconds), with the checkpoint class sitting cleanly between full evaluation and full-KV reuse. The class boundaries are a per-box calibration — roughly 10x separation at the extremes on the CUDA box, narrower on Metal — not universal constants. No instrumentation beyond reading a field the API already returns is needed to know which state served a call.

The transition is equally visible. A qualification loop before one companion — four calls, committed as evidence rather than analysis data — recorded the prefill trajectory [216.0, 17.8, 16.3, 17.9] ms — and the byte flip lands on the same call as the timing flip: call zero (full evaluation) produced the fresh-load byte-stream, calls one through three (full-KV) produced the cached byte-stream, with both matching the byte-streams recorded in earlier sessions. A second qualification loop in a fourth session reproduced the same pair. The state changes and the bytes change at the same call boundary; nothing about it is gradual.

---

## 4. Two Hypotheses Die by Their Own Gates

> **Answer:** Locating which lever selects the state took three manipulated A/B designs, and the first two voided their own manipulation gates — which is the honest cost of gates that check whether the manipulation physically happened. A cache-eviction design (flush the prefix with a different-prompt call) voided: warm-arm prefill 35.7 ms versus cold-arm 36.3 ms — there was no cached state to evict, so the arms never separated. A prewarm-qualified retry voided the same way: the session qualified into the full-KV state in four calls, then every A/B call fell to the checkpoint state regardless of arm (35.8 versus 36.3 ms). A timing design — back-to-back calls versus 600 ms gaps, jitter suppressed — *falsified timing outright*: adjacent median 36.14 ms, gapped 36.31 ms, indistinguishable. Inter-call spacing does not select the state.

The voids were not wasted runs; each narrowed the hypothesis space and two produced standing evidence on the way down. The eviction companion's hundred measured calls, all landing in one state, were byte-identical to the same state's byte-stream from a *different server session* — the first cross-session state-level byte match, which recast the launch smokes' apparent "cross-session drift" as state mismatch rather than drift. And the prewarm companion's trajectory (section 3) captured the state transition and the byte flip on the same call.

What survived every run — five sessions, every prewarm, every void — was a single pattern: **the full-KV state appears only on a model instance that has never served a different prompt.** The first different-prompt call an instance answers forms the persistent 39-token template checkpoint the serve logs show being restored, and every subsequent same-prompt call lands in the checkpoint state — regardless of gap, order, or repetition — until the instance is reloaded. Not eviction, not availability, not timing: *history*. That is a manipulable claim, so the series' rules required manipulating it.

---

## 5. The Confirmation: One Call Flips the Instance, Five Times in Five

> **Answer:** The confirming design made the instance the unit: five cycles, each beginning with an unload confirmed against the serving roster, then a burn-in call (the measured prompt itself — the fresh-load state, excluded like every control), then ten same-prompt calls ("pure"), then **one** different-prompt call (a short classification fixture, excluded), then ten more same-prompt calls ("contaminated") — 110 calls, fixed schedule, no warmups anywhere, because a warmup's different prompt is itself the checkpoint trigger under test. The gate demanded physical separation: every pure call's prefill under 25 ms, every contaminated call's over 30 ms, every cycle's reset confirmed. It passed with zero overlap — pure 16.2–23.9 ms (median 17.3), contaminated 31.5–35.8 ms (median 33.2), resets five for five. All three registered endpoints held: **both arms internally byte-deterministic (modal share 1.0, fifty of fifty each); the arms' byte-streams differ; and the flip sits at the interposed call in every one of the five cycles.** Nothing flipped early; nothing flipped late.

| Registered endpoint | Prediction | Result |
|---|---|---|
| Within-arm determinism, pooled across cycles | modal share 1.0, both arms | 1.0 and 1.0 (50/50 each) |
| Cross-arm modal equality | arms differ | differ |
| Flip located at the interposed call, per cycle | all 5 cycles | 5 of 5 |
| Manipulation gate | prefill classes separate; resets confirmed | zero overlap; 5/5 confirmed |

One different-prompt call is sufficient to move an instance out of the full-KV state, and the move is permanent for the life of the instance: ten subsequent identical prompts in every cycle failed to restore it. Under the gate, with the reset confirmed each cycle, instance history is the lever.

The fork itself deserves its sentence. The pure and contaminated byte-streams run identically for 287 characters and then split mid-explanation — "…the exact operands and **the order of operations.**" versus "…the exact operands and **the intermediate results**" — two defensible continuations of a prompt about why floating-point addition is not associative. The instrument is explaining the mechanism that moves it.

---

## 6. The Bytes Belong to the Session

> **Answer:** The confirmation also found the claim's boundary, and it is disclosed rather than absorbed: none of the three states reproduced the bytes they had produced in the previous four server sessions. Across sessions one through four, each state's byte-stream was stable — the same fresh-load bytes, the same full-KV bytes, the same checkpoint bytes, recognized by hash across four independent server sessions, all on the CUDA box (per-state bytes never crossed machines to begin with: the parent study measured cross-box divergence at exactly this generation length). In the fifth session, on a request verified byte-identical (same request hash, same engine version, same weights digest), all three states produced *new* byte-streams — zero matches against the four on-record variants — while reproducing the state *structure* exactly: three prefill classes, each perfectly deterministic, flipped by one interposed call.

Within the session, the new bytes were rock-stable: each state's stream survived five independent unload-reload instance cycles unchanged — instance resets do not reroll the bytes; something session-scoped selects them, and this series does not yet know what. So the honest statement has two layers. **Within a server session, output bytes are a deterministic function of prompt-cache state, and the state is a deterministic function of instance history** — manipulated, gated, five for five. **Across server sessions, the per-state bytes held four times and changed once** — observed, unexplained, and now part of the claim's stated scope rather than a footnote. History selects the bytes, and the server session is part of the history.

---

## 7. Near-Ties Are Where States Flip Bytes

> **Answer:** A logprob-margin companion measured *where* a state shift can express itself, and the answer matches everything this series has found about instability: only at near-ties. On the cells that moved in the third study, the minimum top1-minus-top2 margin along the generated path was razor-thin — 0.0034 on the 120B model's unstable high-effort cell (a 49:1 split at n=50) against **0.846** on its own stable low-effort twin, a ~250x gap on the matched pair; 0.0014 on the least-stable dense model's long-generation cell (19:1). Every stable cell measured a minimum margin of at least 0.011. The observed forks sat at margins of roughly 0.005–0.019: token 213, " output" versus " JSON"; token 199, " adding" versus " sum".

This is the same law wearing a third face. The first study found flips concentrating where the judgment is closest; the validator reanalysis found the one semantic fork living on a genuinely ambiguous phrase; this companion measures the mechanism's precondition directly. A serving-state change — different cache path, different kernel schedule, different accumulated numerics — perturbs the computed distribution slightly. Where the top two continuations are separated by a wide margin, the perturbation changes nothing and the bytes hold. Where they sit a few thousandths apart, any perturbation can pick the coin up and put it down on the other face — deterministically, for as long as that state persists. One caveat carries: the margin instrument itself is generation-affecting at length on this engine (measured in the third study's freeze gate), so margins describe logprob-instrumented trajectories, and the companion's cells are suffixed so they can never be mistaken for frozen-cell records.

---

## 8. The State-Pinning Pattern

> **Answer:** The validator paper ended with an uncovered surface: free-form generation admits no deterministic acceptance rule, so its instability has no output-side fix. The state findings supply the input-side fix for owned hardware: **byte-reproducible long-form generation is achievable today by pinning the serving state, not just the request.** Both pinned regimes measured perfect in these runs — always-fresh (reload before every call: 100/100 on both boxes) and steady-state (hold one instance in one state: 90/90, 100/100, and 50/50-per-arm wherever a single state persisted). What is not reproducible is a *mixture* — and mixtures are what naive schedules produce.

Concretely, for teams running open-weight models on their own hardware:

1. **Pin the regime.** Same weights, same request, same greedy decoding, same machine still admit three byte-streams. Choose deliberately: reload per call where per-call cost tolerates it (the churn arm's load median bought perfect reproducibility at 5.4 s per call on CUDA, 1.6 s on Metal), or dedicate a warm instance to one prompt family and never let it serve anything else.
2. **Read the state on every call.** `prompt_eval_duration` is free telemetry (nanoseconds in the response schema; the figures here are milliseconds); calibrate the class thresholds per box — ~10x separation at the CUDA extremes, narrower on Metal. Record the class next to the response hash; alarm on a class jump the way [the validator paper](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) alarms on fence rate. A golden-output comparison that ignores serving state is comparing across a hidden variable.
3. **Treat server restarts as byte boundaries.** Per-state bytes held for four sessions and changed in a fifth on identical requests. Re-baseline golden outputs after any engine restart, upgrade, or host change — the state structure will survive; the exact bytes may not.
4. **Keep one different-prompt call away from pinned instances.** One is all it takes, and the flip is permanent for that instance. Health checks, warmup pings, and "quick test" prompts against a pinned instance are state contamination with a measured signature.
5. **Canonicalize what parses; pin what does not.** Structured output already has its answer — parse and canonicalize, per the validator reanalysis. State-pinning is for the surfaces that paper could not cover: prose, code, long-form anything.

| Regime | How | Measured |
|---|---|---|
| Always-fresh | confirmed unload + reload before each call | 100/100, both boxes |
| Steady-state | one instance, one prompt family, no interlopers | 90/90 · 100/100 · 50/50 per arm |
| Mixed (naive schedule) | states sampled unawares | 0.60–0.90 modal share |

None of this is claimed for API-door inference. The serving stacks behind managed endpoints surely have their own state structure — the serving-path study measured behavior consistent with that — but nothing in these local runs licenses a claim about infrastructure this series cannot instrument, and the pattern above requires owning the box.

---

## 9. Limitations

> **Answer:** These are exploratory companions with pre-committed plans, not pre-registered studies: one engine (Ollama 0.30.5, pinned), primarily one model (gpt-oss-20b) and one fixture (the 400-word floating-point prose task) for the state work, two machines, and a manipulated confirmation run on the CUDA box only. Three of five manipulation designs voided their own gates and are reported as voids. The registered churn contrast and the fresh-instance confirmation are the two results that cleared a passing gate; everything else is descriptive. The session-scoping of per-state bytes rests on one observed break in five sessions, with no mechanism identified. The margin measurements are observer-affected at generation length by construction. And the three-state taxonomy is this engine's cache architecture speaking — other engines will have their own state structures, plausibly with the same moral and different details.

The scope limits inherited from the parent study apply unchanged: open-weight models on owned hardware, English fixtures, fixed windows, one engine version. Nothing here modifies any registered answer from the three studies; the companions bracket their frozen cells with suffixed cell keys so no record can collide with the confirmatory datasets.

---

## 10. Reproducing This

> **Answer:** Every run, plan, gate, and report in this paper is public, and the commit ordering is third-party checkable: each companion's plan section was committed to the harness repository before its first call, and the raw records, manifests, and analyzer reports were committed after. The manipulated confirmation reruns with one command against a local Ollama instance; the analyzers rerun from the committed records with zero model calls.

The runner modes are `study3-churn-ab`, `study3-margins`, `study3-cache-ab`, `study3-cache-timing`, and `study3-cache-instance` in [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness); the plans and results live in `FOLLOWUP-COMPANIONS.md`; the reports (`churn-ab-report-*`, `margins-report-*`, `cache-ab-report-*`, `cache-timing-report-*`, `cache-instance-report-*`) carry every number quoted here; the serve-log excerpts identifying the three states are committed as evidence alongside the runs that produced them. The analyzers are standard-library Python with no model anywhere in the loop — the same measurement stance as every paper in this series. Anyone with a consumer GPU and an afternoon can check whether their engine has the same three states; a different state structure with the same within-state determinism would itself be a finding.

---

## References and further reading

- [*The Same Request Twice: Inference Reproducibility on Models With No Sampling Controls*](https://isimplifyme.com/whitepapers/the-same-request-twice) — study 1: instability concentrates at near-ties.
- [*The Same Model, Three Doors: Attributing Inference Reproducibility to the Serving Path*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) — study 2: the serving path gates what reaches the caller.
- [*The Same Weights, Two Machines: The Control Ceiling for LLM Reproducibility*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) — study 3: the control ceiling, and the disclosed tension this paper closes.
- [*The Same Object, Different Bytes: What a Deterministic Validator Catches — and What It Cannot See*](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) — the validator reanalysis whose uncovered surface the state-pinning pattern addresses.
- Raw data, pre-committed plans, analyzers, serve-log evidence, and reports: [`github.com/iSimplifyMe/determinism-harness`](https://github.com/iSimplifyMe/determinism-harness).
