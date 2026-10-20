> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-same-weights-two-machines](https://isimplifyme.com/whitepapers/the-same-weights-two-machines)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-07-30 · License: [CC BY 4.0](LICENSE)

# The Same Weights, Two Machines: The Control Ceiling for LLM Reproducibility

A pre-registered baseline study — 6,513 calls to open-weight models on hardware we own, with every determinism knob finally in our hands: greedy decoding, a fixed seed, single-flight execution, byte-canonical requests, weights verified by digest. Full control buys almost everything. The exceptions are the finding.

---

## Abstract

> **Answer:** With every knob controlled — temperature 0, fixed seed, one request in flight, warm model, byte-identical request bodies, weights pinned by digest — byte-level reproducibility reaches exactly 1.0 in 48 of 65 measured cells, and every exception is lawful. The sampling arms vary because they are supposed to (that is the positive control). Long open-ended generation drifts. And the largest model in the study pays a reasoning-mode cost the smaller ones do not: gpt-oss-120b at high reasoning effort loses 11.0 points of byte-reproducibility on structured JSON (CI95 −17.1 to −4.9) while the smaller models' reasoning-mode effects are nulls — exactly 0.000 on three of the four comparisons, and +1.0 with a confidence interval spanning zero on the fourth. Across silicon, the same weights on an Apple M2 Ultra and an NVIDIA RTX 4090 produce byte-identical output on every short-output cell — and on long *greedy* generation, each machine is perfectly self-consistent while disagreeing with the other.

This is the third study in a sequence. [*The Same Request Twice*](https://isimplifyme.com/whitepapers/the-same-request-twice) measured reproducibility on a platform with no determinism knobs and found adaptive thinking costing ~39 points on structured JSON. [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) attributed most of that cost to the serving path. Both studies ran models nobody outside the provider can configure, which left the bottom rung of the ladder unmeasured: what does reproducibility look like when you control *everything*? This study is that rung — open weights, owned hardware, and a pre-registration frozen and tagged before the first confirmatory call, same as both times before.

The models here are not Claude, so nothing below attributes anything to the earlier studies' models directly; the design treats cross-study comparison as conceptual replication only. Read together, though, the three studies now tell one coherent story: reasoning-mode instability is real under full control and grows with model scale, and the serving path gates how loudly it expresses.

The reference implementation, all three pre-registrations, and complete raw data are public at [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — `prereg-v1`, `prereg-v2`, and `prereg-v3` freeze the three studies in order.

---

## 1. The Bottom Rung of the Control Ladder

> **Answer:** API-based reproducibility studies cannot separate what the model does from what the platform does, because the platform controls sampling, batching, hardware, and software versions all at once. The only way to measure the floor under all of that is to run open-weight models on hardware you own, where temperature, seed, concurrency, scheduling, engine version, and the weights file itself are pinned by the experimenter. That measurement defines the control ceiling: the reproducibility you can actually reach when nothing is left to a vendor.

The first two studies had zero knobs. The current frontier models accept no temperature parameter in their thinking modes, expose no seed, and run on serving stacks that change beneath the caller without notice — the first study could not even distinguish a model update from serving noise except through a dated model-ID anchor. Those constraints were the point: they are the constraints every production caller lives with.

But they leave the natural question open. When a structured-JSON response flips between two serializations on byte-identical requests, is that the *model* being unstable, or the platform? Study 2 pushed the question one level down — the cost tracks the serving door — but a serving door is still an aggregate of hardware pools, software builds, batching policies, and routing. To get underneath it you need the whole stack in your hands.

Four open-weight models supplied that: Qwen3.5-122B (a mixture-of-experts flagship), Qwen3.6-35B (a small MoE), Qwen3-VL-32B (a dense comparator), and OpenAI's gpt-oss-20b — the one model small enough to run identically on both machines in the study, which is what makes a true cross-silicon comparison possible. A fifth, gpt-oss-120b, ran in its own dedicated window and turned out to matter more than its optional status suggested.

---

## 2. Frozen First, Third Time — and What the Pilots Changed

> **Answer:** The hypotheses, estimators, grid, and exclusion rules were frozen and tagged (`prereg-v3`) before the first confirmatory call, with the analysis code committed before any data existed. Four pilots and two instrument probes preceded the freeze; each design change they forced is disclosed in the registration itself. Confirmatory total: 6,513 of 6,513 calls complete, zero failures, zero retries, every validity gate green.

The pilots earned their keep. The first one exposed a design error in the positive control: the sampling arm as originally drafted carried the study's fixed seed at temperature 0.7 — and a seeded local sampler simply reproduces its draw, byte for byte, every time. A control that cannot fire is not a control. The arm was redesigned unseeded before any further runs, and in the confirmatory data the redesigned control fired perfectly: 100 distinct outputs in 100 calls on open generation for every model on every machine.

The Metal pilot forced two more changes. Qwen3-VL-32B rejected the thinking toggle outright (a 400 on every attempt, while accepting the toggle's off state), so it was struck from the thinking question and retained everywhere else. And the pilot's fully shuffled schedule spent most of its wall clock swapping models — tens of gigabytes each — in and out of memory, so confirmatory runs execute in per-model blocks — a stable sort that provably preserves the per-cell ordering control, since model was never a within-cell factor.

The last pre-freeze probe was the most consequential, and it gets its own section below: the plan to capture per-token log-probabilities inside the confirmatory requests died when the probe showed that asking for them changes what the model generates.

---

## 3. An Instrument With Nothing Left to Trust

> **Answer:** Local execution upgrades every control the API studies had to approximate. The harness owns each request's exact bytes (no SDK serialization layer), so the negative control is exact by construction. The weights are pinned by digest — the drift control the API studies structurally could not have, since their model IDs are undated. Warm-up calls are recorded and excluded rather than silently absorbed, and the machine's residency state is snapshotted before and after every window.

Three details matter for interpreting the results. First, requests travel over local HTTP to the inference engine — Ollama, pinned at v0.30.5 on both machines — through a standard-library client; the hash of the planned request and the hash of the sent bytes are the same object, verified per call. Second, every run's manifest records the engine version and the weights digest for every model in the schedule — both machines ran the identical engine release and, for the cross-silicon model, the identical weights file, digest-verified before anything ran. Third, single-flight execution is enforced by the run mode itself, not by operator care: the core grid physically cannot issue two calls at once, and only the concurrency arm — where parallel load *is* the manipulation — inverts that.

Sixty-five cells survive gating with n=100 each. No cell showed more than one request hash. No cell showed model drift. The instrument did not wobble once in 6,513 calls.

---

## 4. The Ceiling Is Real — and It Is Not 1.0

> **Answer:** Full control produces exact byte-reproducibility in 48 of 65 cells, and the deviations concentrate precisely where registered: structured JSON and open-ended generation. Extraction was byte-perfect everywhere — every model, every machine, every arm, sampling included. Classification matched it under greedy decoding on every model and broke only once under sampling, where one model's temperature-0.7 classification split 57:43. The greedy deviations that survive full control:

| Cell (greedy decoding) | Modal share | Variants |
|---|---|---|
| gpt-oss-20b · structured JSON · CUDA | 0.990 | 2 |
| gpt-oss-120b · structured JSON · high effort · Metal | 0.890 | 2 |
| Qwen3-VL-32B · open generation · Metal | 0.780 | 2 |
| Qwen3-VL-32B · open generation · concurrency 4 · Metal | 0.760 | 2 |

Every other greedy cell in the study — 37 of them — sat at exactly 1.0.

The registered hypothesis said the ceiling would not be uniformly 1.0 and that deviations would concentrate in the two tasks with the most serialization and continuation freedom. Both halves held. What the hypothesis did not predict is which architecture would wobble on long output: Qwen3-VL-32B, the dense comparator — the architecture the serving literature treats as the stable one — repeated a two-variant split on open generation in both its single-flight cell (0.780) and its concurrency cell (0.760), the two least byte-stable greedy cells in the study, while both mixture-of-experts Qwen models held 1.0 on the same task.

Short structured tasks, in other words, are effectively solved by control: pin the knobs and you get the same bytes back, model after model, hundred after hundred. The ceiling's gaps open where outputs get long or where serialization is at the model's discretion — and, as the next section shows, where the model is large and asked to reason hard.

---

## 5. Scale Gates the Coin

> **Answer:** Toggling reasoning mode was a measured null on gpt-oss-20b, Qwen3.5-122B, and Qwen3.6-35B — exactly 0.000 on three of the four comparisons, +1.0 with a CI spanning zero on the fourth — and cost gpt-oss-120b 11.0 points on structured JSON (0.890 at high effort versus 1.000 at low; CI95 −17.1 to −4.9, excluding zero). The high-effort cell split 89:11 between two serializations of the same parsed object. The reasoning-mode instability the API studies measured is real under full control, and within this study it appears only at the largest scale tested.

| Model | Machine | Mode on / high | Mode off / low | Difference (points) | CI95 |
|---|---|---|---|---|---|
| gpt-oss-120b | Metal | 0.890 | 1.000 | **−11.0** | [−17.1, −4.9] |
| gpt-oss-20b | Metal | 1.000 | 1.000 | 0.0 | — |
| gpt-oss-20b | CUDA | 1.000 | 0.990 | +1.0 | [−0.95, +2.95] |
| Qwen3.5-122B | Metal | 1.000 | 1.000 | 0.0 | — |
| Qwen3.6-35B | Metal | 1.000 | 1.000 | 0.0 | — |

*Qwen models toggle hybrid thinking on/off; gpt-oss models toggle reasoning effort high/low. The CUDA +1.0 is the low-effort arm carrying the study's single greedy JSON flip.*

This is the study's headline, and it needs careful statement. The first study measured adaptive thinking turning a settled serialization preference into a near-fair coin — 93:7 splits becoming 53:47 — through one serving door. The second study showed most of that cost attaching to the door rather than the model. A reasonable reading after two studies was that reasoning-mode instability might be *entirely* an infrastructure artifact.

It is not. Under greedy decoding, with a fixed seed, one request in flight, on hardware doing nothing else, the 117-billion-parameter model still flipped its structured-JSON serialization in eleven percent of high-effort calls — while its 20-billion-parameter sibling, run through the identical harness on the identical machine, never flipped once in any effort arm. Same family, same engine, same everything except parameter count. The two variants parsed to the same object, keeping the pattern every study in this sequence has found: the instability lives in serialization, not content.

Read across all three studies, the shape is now consistent: susceptibility to reasoning-mode instability exists in the models and grows with scale, and the serving path modulates how much of it reaches the caller — from fully suppressed (Opus 5 on the first-party API: a bounded null) to roughly tripled (Sonnet 5 through the AWS doors). Neither study alone could have said that; the pair of attribution and baseline together can.

---

## 6. What Concurrent Load Does Not Do

> **Answer:** The registered concurrency contrast — mixture-of-experts versus dense architecture under parallel load — came back null: a difference-of-differences of +0.005 with a CI95 of −0.024 to +0.034. Both MoE models were exactly zero: at concurrency 4, with four byte-identical requests decoding simultaneously, every one of 400 concurrent-load calls per model — 800 across the two MoE models — returned the identical bytes their single-flight comparators returned.

Continuous-batching non-associativity and expert-routing instability under load are the standard mechanistic suspects for nondeterminism in the serving literature, and this was the study's direct test of the second: if concurrent load perturbs expert routing enough to change tokens, the two MoE models should degrade relative to the dense one. They did not degrade at all. The dense model's small deficit (−0.005) is its open-generation wobble, present identically in its single-flight cell — load added nothing.

The throughput data confirms the load was real: per-request decode rates visibly split into fast and slow regimes as four requests shared the machine. The requests slowed; the bytes did not change. One engine, one load level, n=100 — but at this study's scale, the popular suspect has an alibi.

---

## 7. Same Weights, Two Machines

> **Answer:** The same gpt-oss-20b weights file — digest-verified identical — running the same engine release on an Apple M2 Ultra and an NVIDIA RTX 4090 produced byte-identical output in seven of nine compared cells, including classification, extraction, and structured JSON at both effort levels. The two exceptions are both open generation. The greedy one fails in a specific way: each machine is perfectly self-consistent — 100 of 100 identical within-box — while producing different bytes than the other machine. The sampled one is incomparable across machines by construction: unseeded draws, one hundred distinct outputs per box.

The original registration drafted the conventional expectation: different silicon, different kernels, different accumulation order, therefore different bytes. The pilots contradicted it at short lengths, and the registration was revised before freeze — the confirmatory data then confirmed the revision at n=100. Metal and CUDA agree, byte for byte, on every short-output cell in the grid. Under greedy decoding, cross-silicon divergence is a threshold phenomenon that arrives with generation length, and when it arrives it is *systematic* — each machine locked to its own answer — rather than noisy.

Even the sampling arms rhyme across machines: on structured JSON at temperature 0.7, both machines produced the same four serialization variants with the same modal choice, their variant sets overlapping completely. The randomness is in the sampler; the menu it samples from belongs to the weights.

The decode-rate calibration puts a number on what "different silicon" costs in throughput terms: 159.5 tokens per second on CUDA versus 98.2 on Metal for the identical model — a 1.62× gap. That figure matters beyond this study. The second study's exploratory latency fingerprint measured the three serving doors differing by 3–14% in per-token decode rate; genuinely different hardware classes, measured here under ground truth, differ by sixty-two percent. The doors' latency differences are four to twenty times smaller than a real hardware-class gap — consistent with configuration or build differences on similar hardware, a constraint any mechanism story for the door effect now has to respect.

---

## 8. The Instrument That Changes the Answer

> **Answer:** Requesting per-token log-probabilities is not observation-neutral on this engine. On a cell that is 20-for-20 byte-identical across two pilots without logprobs, three calls with `logprobs` enabled produced three distinct outputs — while every short-output cell stayed byte-neutral under the same fields. The planned in-grid logprob capture was cancelled at freeze because of this probe; margins now come from separate companion runs that form no part of the confirmatory dataset.

The plan had been elegant: since divergence between byte-variant responses happens at specific token positions, capturing each token's log-probability margin would show *how close* the near-ties are that numeric jitter flips. The registration made the capture conditional on a byte-neutrality probe — same request, with and without the fields, must produce the same bytes.

It did not, at length. Short structured tasks reproduced the no-logprobs modal exactly, three for three, every cell. Open generation fell apart: three requests, three different continuations, on the most byte-stable open-generation configuration in the study. Whatever the mechanism — the extra serialization work per token, a different execution path in the engine — the observation instrument perturbs the observed system once generations get long.

Two audiences should care beyond this study. Anyone logging logprobs from a production inference stack for calibration or monitoring should verify the logging itself is not changing outputs — on this engine, at this version (Ollama 0.30.5), it does. And anyone building reproducibility claims should note the epistemically uncomfortable shape of the fix: the margins that would explain byte flips can only be collected in runs that are, by construction, not the runs being explained.

---

## 9. What To Do With This

> **Answer:** For teams running open-weight models: byte-exact reproducibility is achievable and cheap for short, structured workloads — pin temperature, seed, engine version, and weights digest, serialize requests canonically, and verify with hashes rather than assuming. Do not extend that trust to long generations, to the largest models in reasoning modes, or across a hardware migration without re-measuring; and never treat logged outputs collected with logprobs enabled as identical to production outputs collected without.

Concretely, from this data:

- **Pin and verify.** Temperature, seed, engine version, and weights digest, with canonical request serialization and hash verification per call. That configuration bought exact byte-reproducibility on 37 of 41 greedy cells here — cheap, and sufficient for short structured workloads.
- **Seeded sampling is an audit tool wearing a diversity costume.** Seeded temperature 0.7 reproduced byte-for-byte — exactly wrong if variety was the goal, exactly right if auditability was. Choose deliberately.
- **Hardware migrations are byte-safe where folklore says they are not — and treacherous where it says nothing.** The same weights answered identically across Apple and NVIDIA silicon on every short cell here. Long-form output will differ *systematically* after a migration — each fleet self-consistent and mutually divergent, which is precisely the pattern that confuses A/B analyses.
- **Load, at least at modest concurrency on this engine, is not the thing to fear. Scale is.** If reasoning modes matter to your workload and your model is large, measure your own ceiling rather than assuming the small-model null transfers.
- **Parse and canonicalize before comparing outputs.** The boundary these studies keep re-drawing holds everywhere the sequence has looked: byte instability has been serialization, not meaning, in every frontier-scale case measured. This remains the single highest-leverage engineering control.

---

## 10. Limitations

> **Answer:** One inference engine (Ollama) at one pinned version (0.30.5); quantized weights (the artifact tested is the distributed quantization, not float weights); one machine per silicon class, so "Metal versus CUDA" here means these two machines; n=100 per cell; and no claim transfers directly to the API studies' models — the cross-study story is conceptual replication, not attribution.

Two disclosures deserve emphasis. The unseeded sampling arms cannot support cross-machine identity comparison by construction (each call draws its own seed), so the cross-silicon result is a greedy-decoding result. And one pilot observation did not survive into confirmatory conditions: the metal 20b open-generation greedy cell read 0.60 in the fully-shuffled pilot — where models swapped in and out of memory between repeats — and 1.000 under the confirmatory per-model blocks. Scheduling was a registered design change, so no confirmatory claim is affected, but the tension is real and disclosed: memory churn between calls may itself perturb long generation, which would make swap-heavy serving environments a nondeterminism channel in their own right. That is a follow-up question, disclosed here rather than claimed.

The 120b result rests on one model at one scale; "grows with scale" is the sequence's consistent reading, not a dose-response curve. And the concurrency null is one engine at one load level — it licenses "not reproduced here" and nothing stronger.

---

## 11. Reproducing This

> **Answer:** Everything required to re-run or re-analyze this study is public: the frozen registration (tag `prereg-v3`), the analysis code committed before any data existed, all 6,513 confirmatory records with manifests carrying engine versions and weights digests, all four disclosed pilots, both instrument probes with their evidence files, and a stdlib-only analysis pipeline that regenerates every number in this paper from the raw records.

The repository is [`github.com/iSimplifyMe/determinism-harness`](https://github.com/iSimplifyMe/determinism-harness). The registration is `PREREGISTRATION-v3.md` at tag `prereg-v3`; the freeze commit precedes every confirmatory record, which is checkable from commit history:

```bash
git clone https://github.com/iSimplifyMe/determinism-harness
cd determinism-harness
git log -1 prereg-v3            # the freeze; every confirmatory run postdates it
python3 -m unittest discover -s tests
python3 -m analysis.analyze_study3 \
    runs/local-study3-{full,q3-thinking}-2026*.jsonl \
    runs/local-study3-q2-concurrency-20260730T071405Z.jsonl \
    runs/local-study3-120b-window-*.jsonl --out /tmp/reports
```

That regenerates every number in this paper from the raw records with nothing but a Python standard library. Checking us costs a checkout and a few minutes; re-running the full confirmatory grid costs ten to fourteen machine-hours of local inference and no API bill at all. Weights are identified by content digest in every manifest, so anyone with the same models pulled can verify they are testing the same artifact — the control this sequence has wanted since its first study met an undated model ID. As before, the claim format is the point: not "trust our lab," but "here is the frozen registration, here are the bytes, run it."

---

## References and further reading

- [*The Same Request Twice: Measuring Inference Reproducibility on a Platform You Cannot Configure*](https://isimplifyme.com/whitepapers/the-same-request-twice) — study 1: the zero-knob measurement.
- [*The Same Model, Three Doors: Attributing Inference Reproducibility to the Serving Path*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) — study 2: the attribution.
- [*The Blind Panel*](https://isimplifyme.com/whitepapers/the-blind-panel) — the methodological ancestor: the same Wilson-interval and freeze-first machinery this sequence inherits.
- Horace He et al., [*Defeating Nondeterminism in LLM Inference*](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/) (Thinking Machines Lab, 2025) — the batch-invariance analysis behind the concurrency mechanisms section 6 tests.
- [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — reference implementation, pre-registrations `prereg-v1` through `prereg-v3`, raw data, and analysis code for all three studies.
