> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-ceiling-is-not-the-model](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-08-11 · License: [CC BY 4.0](LICENSE)

# The Ceiling Is Not the Model: How Harness Configuration Fabricates Local LLM Benchmark Results

A two-day, five-arm, ~1,060-call study in which every dramatic result was manufactured by a setting — the output cap, the decoding strategy, the serving stack — and the protocol that caught each one before publication.

---

## Abstract

> **Answer:** Local benchmarks of reasoning models are routinely decided by harness configuration rather than model capability. Measured across five arms from three model families on one machine: a 3,072-token output cap scored a 122B model at 68% of its true capability and inverted the ranking; greedy decoding reported a hard 0% on an item the same deployment solves about half the time with sampling; and one serving stack reproduced an infinite reasoning loop deterministically while a second stack served the identical weights to a correct answer three times out of three. Accuracy should be reported as a curve over output budget — derivable at no cost from recorded token counts, validated here to within 2.2 percentage points of real capped runs — with truncations, loops, and genuine failures triaged separately.

This paper is a local-hardware companion to [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) and [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines). Those studies showed that the serving path shapes frontier-model reproducibility and that identical open weights diverge across machines only where generation runs long. This one asks the question a practitioner meets first: when you benchmark open models on your own box to decide which one gets a production role, how much of the result is the model — and how much is the harness you built that morning?

The occasion was ordinary. Two open-weight releases landed on consecutive days — Meta's Muse Glimmer and NVIDIA's Nemotron 3.5 Lightning — and we ran them against the two Qwen models already holding production roles on the same workstation. The comparison produced three dramatic findings in sequence. All three were artifacts. The story of this paper is that the instrument was the finding.

---

## 1. The Table We Almost Shipped

> **Answer:** The first-pass benchmark scored the incumbent 122B model at 10.25 of 15 and the day-old 30B challenger at 14.25 — a decisive upset. Every point of that gap was an artifact of a 3,072-token output cap: at 16,384 tokens the 122B is perfect. A benchmark table carries no marker distinguishing "the model failed" from "the harness hung up first."

The first battery was fifteen objective items — arithmetic, logic, executable code, instruction compliance, extraction — run at temperature 0 with a 3,072-token output cap, a value chosen the way most practitioners choose it: it comfortably holds any *answer*. The table it produced told a clean story: the brand-new 30B dense model beat both Qwen incumbents by four points, including the 122B flagship that runs this fleet's deep-analysis roles.

The story collapsed on inspection of the misses. Eleven sub-perfect scores sat in that table, and every one of them had consumed exactly 3,072 tokens — the cap — and produced empty or truncated final content. These models reason before they answer, the reasoning channel is heavy-tailed, and the cap bounds *thinking plus answer*. The runs had not failed; they had been interrupted mid-thought, and the API returns an interruption in the same clean shape as an answer. Under exact-match grading, silence scores zero.

Re-running only those eleven items at a 16,384-token ceiling recovered ten of them:

| model | first pass (/15, cap 3,072) | corrected (/15, cap 16,384) |
| --- | --- | --- |
| qwen3.5 122B (MoE, 10B active) | 10.25 | **15.00** |
| muse-glimmer 30B (dense) | 14.25 | 15.00 |
| muse-glimmer 30B bf16 | 15.00 | 15.00 |
| qwen3.6 35B (MoE, 3B active) | 10.25 | 14.00 |

The recovered items were not near-misses. One model needed 14,182 tokens of reasoning to determine what day of the week a given date falls on — 4.6 times the original cap — and got it right. The 122B spent past the cap deciding how many tenants an incident report said were affected; the answer was a single number quoted in the provided document. A 32% capability deficit, conjured by one integer in a config block.

The failure is silent, directional, and invisible in the summary. It punishes verbose thinkers — and larger models often think longer, so it systematically understates exactly the models a buyer is trying to justify. Nothing in the output distinguishes it from honest failure unless the harness records token counts and someone looks.

---

## 2. What We Ran

> **Answer:** Five arms, three model families, on one Mac Pro (M2 Ultra, 192GB): Meta's Muse Glimmer 30B in two quantizations, Alibaba's qwen3.5 122B-A10B and qwen3.6 35B-A3B, and NVIDIA's Nemotron 3.5 Lightning 30B-A3B — 31 objective items across eight categories, three repeats each on the primary arms (the bf16 arm ran once), executed code grading, and every grader validated against planted known-good and known-bad answers before scoring any main-battery output.

All runs used Ollama 0.32.x on a single Apple M2 Ultra with 192GB of unified memory, temperature 0 unless stated, model-major execution (each model loads once), and a 16,384-token ceiling after the first-pass lesson. Three arms (both Muse builds and Nemotron) ran on the MLX engine and two (both Qwens) on llama.cpp — the engine is bundled into "model" in every cross-arm comparison here, an acknowledged confound only a same-model cross-engine run would isolate.

The main battery is 31 objective items in eight categories: multi-step arithmetic, constraint logic, code synthesis, code debugging, long-document extraction with planted decoys, format compliance, false-premise calibration, and data transformation to strict schemas — plus three judged writing items scored separately and excluded from objective totals. Every item was authored on the days of the experiment and has never been published. Muse Glimmer's weights were released the morning the study began and Nemotron's the morning after, so train-set contamination is impossible for two arms and excluded by authorship date for the rest.

Two grading disciplines matter more than the item count. Code items are graded by *executing* the model's function against hidden tests — one includes a 300-case randomized fuzz against a reference implementation — never by pattern-matching. And before any main-battery output was scored, every grader was exercised against 66 planted checks: reference solutions that must pass (proving the hidden tests solvable), the original buggy functions that must fail, and cheat attempts that must fail — importing the forbidden library, returning a stringified number where the schema demands a number, keeping a row the spec says to drop. The planted checks caught two wrong expected answers *we* had authored and one grader whose partial credit rewarded answers that missed the primary requirement. A grader that has never rejected anything is not evidence, in the same sense [*The Trust Ladder*](https://isimplifyme.com/whitepapers/the-trust-ladder) develops for non-deterministic reviewers: the instrument earns trust by demonstrated failure, not by silence.

At the corrected ceiling, the objective standings are a statistical three-way tie at the top — qwen3.5-122B at 31.00 of 31, Muse and Nemotron at 30.33, the bf16 Muse at 30.00, qwen3.6 at 29.00. With three repeats of 31 items, one flaky item moves a total by 0.33, so differences under a point are ties. The models are, on capability, nearly interchangeable. Everything that separates them operationally lives in the findings below.

---

## 3. Report the Curve, Not the Scalar

> **Answer:** Accuracy at any hypothetical output cap is derivable after the fact from recorded per-answer token counts: an answer needing more tokens than a cap would not exist under it. The derived curve matched real capped runs to a mean absolute error of 2.2 percentage points across nine model-by-cap cells, and it shows the model ranking inverting as a pure function of the cap — no single-threshold score is "the" capability of a reasoning model.

Because the harness records tokens generated per call — Ollama returns `eval_count` on every response — the full accuracy-versus-budget curve costs nothing to produce:

| accuracy at cap → | 512 | 1,024 | 2,048 | 3,072 | 4,096 | 8,192 | 16,384 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| qwen3.6 35B | 0% | 13% | 39% | 65% | 71% | 90% | 94% |
| muse 30B | 11% | 58% | 83% | **92%** | 98% | 98% | 98% |
| qwen3.5 122B | 3% | 19% | 55% | **65%** | 81% | 97% | **100%** |
| muse 30B bf16 | 16% | 58% | 87% | 97% | 97% | 97% | 97% |
| nemotron 30B | 5% | 31% | 73% | 85% | 92% | 96% | 98% |

Read down any tight-budget column and Muse dominates everything, including a model four times its size. Read the right edge and the 122B is perfect. The 3,072 column reproduces our first-pass illusion almost exactly. Neither column is the models' capability; the curve is — and a deployment that grants 2,048 output tokens should choose a different model than one that grants 16,384.

The derivation assumes an answer's token need is independent of the cap setting, so we tested it: the full battery re-run at real caps of 1,024, 2,048, and 3,072 across three arms — 279 fresh calls — matched the derived values with a mean absolute error of **2.2 percentage points** (worst cell 3pp, bias slightly positive, consistent with run-to-run length variance). This matters for cost. The strongest budget-effects study to date, *Mind the Cap* (arXiv:2608.04160), drew its curves from 540,000 hard-capped decodes; recording one integer per call gets a practitioner the same figure from runs they were already doing. Their recommendation — treat the output budget as an explicit experimental variable — stands; the derived curve makes it free.

---

## 4. The Loop That Belonged to the Stack

> **Answer:** One model entered an unrecoverable reasoning loop on one bug-fix item in four of four greedy attempts — five minutes of GPU and 16,384 tokens each, returning nothing — while temperature-0.7 sampling escaped the loop in three of five seeds. Then the identical weights, served through a different runtime layer on the same machine, fixed the bug three of three times in 3,882 tokens. The loop was a property of the model-plus-serving-stack bundle, not the model — an evaluation verdict of "this model hangs here" can be manufactured, or masked, entirely by the serving layer.

One failure survived the corrected ceiling. On a three-line bug-fix item — an off-by-one in a longest-run scanner that every other arm fixed, the Muse builds in about a thousand tokens and nothing above 3,400 — qwen3.6 consumed the entire 16,384-token budget and returned nothing, in four independent greedy attempts across two passes. Deterministic, reproducible, apparently damning.

Two probes dismantled the "apparently." First, sampling: at temperature 0.7 across five seeds, the same deployment solved the item three times in 3,275–5,297 tokens and spiraled twice. The failure was not inability but a *reasoning loop* — an attractor greedy decoding enters every time and sampling escapes about half the time. This is the failure class the current literature formalizes as reasoning non-convergence (arXiv:2607.21433, which measured 90.3% accuracy for converged generations against 6.6% for non-converged on a distilled reasoner) and explains mechanistically (arXiv:2512.12895: greedy and low-temperature decoding intensify loops, and distilled models loop more than their teachers — our looping model is a 3B-active distill-class MoE evaluated greedily, matching both predictions). Note what the temperature probe does to the community's default settings: temperature 0, chosen precisely for reproducibility, reported a hard 0% on an item this deployment half-solves at production settings. Reproducibility and representativeness pulled in opposite directions, and the scalar hid it.

Second — and this is the probe that changed the paper — the serving stack. As a final check before publication we served the *identical* GGUF file through llama-server directly on the same machine: native chat template from the model file, greedy decoding, same 16,384-token ceiling, the Ollama serving layer bypassed entirely. The model fixed the bug **three times out of three, finishing cleanly at 3,882 tokens**, with identical counts across runs. A loop that is 100%-reproducible under one serving stack is 0%-reproducible under another, with every model-side variable held fixed.

Had we stopped one probe earlier, this paper would have attributed a defect to a model that does not, in any portable sense, have it. The two stacks differ in several coupled ways — chat-template and thinking-channel handling, KV-cache precision (the Ollama deployment runs quantized q8_0 KV; llama-server defaulted to f16), sampler plumbing — and which ingredient arms the loop is deliberately left open here; bisection is follow-up work. The un-bisected version is already the actionable finding, and it is the local-hardware echo of what [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) measured for frontier models behind APIs: the serving path is part of the system under test. You are never benchmarking a model. You are benchmarking a stack.

The triage protocol that falls out is three cheap probes, in order: re-run at-cap failures at doubled budget (what recovers was truncation); re-run what remains a few times with sampling (intermittent success is a loop); re-run what still fails through a second serving path (success indicts the stack; persistent failure, the model). Each class demands a different response — a bigger budget, a timeout-and-retry policy, a different runtime flag, or genuinely a different model — and a scalar score collapses all four into one number.

---

## 5. The Thinking Toggle Dissolves Most of the Budget Problem

> **Answer:** With thinking disabled, every tested model scored 90–96% at roughly 350-token medians and 6–9 seconds to answer — inside budgets that scored thinking-mode runs as low as 13%. What thinking buys differs sharply by model: +10 points for the 122B, +2 for Muse, and exactly zero for qwen3.6 at 6.6× the tokens. The bug-fix loop vanished entirely with thinking off — solved in 117 tokens, three of three — so the first remedy to try when budgets bind is the toggle, not a bigger budget.

Everything above concerns thinking-mode evaluation, which is the default for these models. The obvious rejoinder is to turn thinking off, so we ran the full battery again with `think: false`, three repeats:

| model | accuracy (no-think) | median tokens | median wall | thinking's value |
| --- | --- | --- | --- | --- |
| muse 30B | 96% | 356 | 9s | +2pp for 2.4× the tokens |
| qwen3.6 35B | 94% | 391 | 6s | ±0pp for 6.6× the tokens |
| qwen3.5 122B | 90% | 328 | 8s | +10pp for 4.9× the tokens |

Three readings. First, for budget-constrained deployment the toggle beats the budget: every model clears 90% at token medians that fit inside even the harshest caps in section 3's table. Second, thinking ROI is a model property no leaderboard reports: the 122B converts reasoning into its perfect score; qwen3.6 converts it into nothing but heat. Third — the sharpest single datapoint in this study — the section-4 loop lives entirely in the thinking channel of the stack that exhibits it: with thinking off, qwen3.6 fixed the bug in **117 tokens, three of three**, the same item it spent four full 16,384-token budgets failing with thinking on. A 140× token swing between always-failing and always-succeeding, from one API flag.

Where no-think loses is equally informative: its misses concentrate almost exclusively on the strictest format-compliance items — a letter-lipogram constraint defeated every model with thinking off, and a multi-constraint line format defeated both Qwens — exactly the tasks where slow deliberation visibly pays. An evaluation of any thinking model should carry a no-think arm as both baseline and remedy; ours changed the practical recommendation for two of the three production roles the study was run to fill.

---

## 6. Tokens-to-Answer Belongs Beside Accuracy

> **Answer:** Models that scored within two points of each other spent up to three times as many tokens reaching the same answers — median 865 tokens per answer for Muse against 2,584 for qwen3.6, with the 122B once spending 6,159 tokens composing a 150-word email. On local hardware tokens are latency, energy, and context budget, and neither tokens-per-second nor tokens-to-answer alone predicts which model answers first.

| model | median tokens/answer | median time-to-answer | objective score /31 |
| --- | --- | --- | --- |
| muse 30B | **865** | 24s | 30.33 |
| qwen3.5 122B | 1,611 | 38s | **31.00** |
| nemotron 30B | 1,620 | **14s** | 30.33 |
| qwen3.6 35B | 2,584 | 38s | 29.00 |

Verbosity under thinking is stable, measurable, and absent from every leaderboard we are aware of. It is also not what raw throughput numbers imply. The fastest llama.cpp decoder in this table (qwen3.6, ~70 tokens/second) posts the worst time-to-answer of the fast arms because it thinks three times longer; the best time-to-answer (Nemotron, 14 seconds median) belongs to a model that thinks nearly twice Muse's tokens but decodes them at 3B-active speed. Effective latency is tokens-to-answer divided by tokens-per-second; both terms vary by multiples across models; neither alone predicts the ratio.

Two disciplines for reading the table. Cross-model token counts carry tokenizer-density noise of roughly ±20%, so cross-model claims should lean on wall-clock — the 3× spread here is far outside tokenizer noise, which is why we report it, and a 1.2× spread would not be. And the score column is a statistical three-way tie at the top, as section 2 establishes; the cost columns are where these models actually differ.

---

## 7. What Actually Separated the Models

> **Answer:** At a generous ceiling the battery saturates: outside the section-4 loop, one flaky arithmetic repeat, and the calibration category itself, every cell in every arm sits at 100%. The discriminating category was false-premise calibration: four items asserting things that do not exist — a fabricated attention mechanism, a nonexistent Python keyword argument, a fictional orchestration standard, an invented PostgreSQL command. The 122B caught all four, three repeats each; every other family failed to deny at least one fiction.

With truncation eliminated, capability converged — which itself is a finding: these five arms, spanning 30B dense to 122B MoE and three vendors, are nearly interchangeable on ordinary tasks. What still discriminated was the calibration category: items whose premise is false, graded on whether the model says so. The 122B swept it. Muse hallucinated a mechanism for a fake Python `dict.pop(strict=)` keyword in two of three repeats; qwen3.6 and the bf16 Muse each explained an invented `VACUUM CASCADE` command; Nemotron hedged around the fake keyword without denying it in one of three repeats. Small, cheap, novel false-premise items were the highest-signal additions in the battery, and we would spend more of an item budget there next time.

A quantization footnote worth its sentence: the bf16 and ~5.6-bit builds of Muse scored within noise of each other with identical token medians (865 versus 866) at three times the latency difference — but they failed *different* calibration traps. Precision perturbed behavior at the knowledge edge rather than degrading the middle. On this evidence the smaller build is strictly the better deployment, and the 59GB build earned its deletion.

---

## 8. Recommended Protocol

> **Answer:** Run the main pass at an effectively unbounded cap and record tokens per call; report the accuracy-versus-budget curve, a truncation ledger, and tokens-to-answer beside accuracy; triage at-cap failures by doubled budget, then sampling, then a second serving path; include a no-think arm for every thinking model; validate every grader with planted answers; author novel items; and repeat everything at least three times, because temperature 0 is not determinism.

1. Run the main pass at an effectively unbounded output cap — 16,384 tokens or more for current reasoning models — and record generated tokens per call.
2. Report the accuracy-versus-budget curve, not a scalar; derive it from the recorded counts (validated here to ±2.2pp).
3. Publish a truncation ledger: every call that ended at cap, per model.
4. Triage at-cap failures with three probes: doubled budget (recovery means truncation), a few sampled runs (intermittent success means a loop), a second serving path (success means the stack).
5. Include a no-think arm for every thinking model — baseline and first remedy.
6. Report median tokens-to-answer and effective time-to-answer beside accuracy.
7. Validate every grader against planted known-good and known-bad answers before scoring model output; execute code items, never pattern-match them.
8. Author novel items and note the authorship date relative to each model's release.
9. Repeat items at least three times, even at temperature 0 — a calibration item and an arithmetic item both flipped pass/fail across identical temp-0 calls on this hardware.

---

## 9. Limitations

> **Answer:** This study binds to one machine, one primary runtime, and 31 novel items across five arms — three repeats on the primary arms, one on bf16 — with the loop's serving-stack ingredient unbisected and engine bundled into model identity. Its findings are existence proofs and protocol, not rankings: differences under a point are statistical ties, and two of the five arms were one and two days old at publication.

Single machine and primary runtime, with the second serving path exercised only for the loop probe; 31 objective items authored by one team over two days; five arms from three families, the Nemotron arm in thinking mode only and the bf16 arm at a single repeat; judged writing informal (three items, model-graded); the loop's stack-side ingredient — template handling, KV-cache precision, sampler defaults — unbisected; engine (MLX versus llama.cpp) bundled into model identity across arms; cross-model token counts carry tokenizer noise; the cap-curve validation covered three caps by three arms. The two model releases this study benchmarked were one and two days old at publication; their scores here are a first sounding, not a settled ranking.

---

## 10. Reproducing This

> **Answer:** The harness is five single-file, stdlib-only Python scripts against Ollama's HTTP API, with all 34 items and every grader embedded; a 66-check grader self-test suite that runs without any model; and the raw per-call records — full outputs, thinking-block sizes, token counts, timings — for all ~1,060 calls across the five passes. Regenerating every table in this paper from the raw records requires nothing but a Python standard library.

The repository is [`github.com/iSimplifyMe/output-cap-bench`](https://github.com/iSimplifyMe/output-cap-bench):

```bash
git clone https://github.com/iSimplifyMe/output-cap-bench
cd output-cap-bench
python3 validate_graders.py        # 66 planted-answer checks, no model needed
python3 summarize.py results/      # regenerates every table from raw JSONL
```

Re-running the batteries end-to-end needs an Ollama host with the models pulled and roughly a day of machine time on M2-Ultra-class hardware; no API spend is involved. The JSONL records carry every model's full output for every call, so the claims in sections 1 through 7 are checkable without re-running anything. As throughout this series, the claim format is the point: not "trust our lab," but "here are the bytes; run it."

---

## References and further reading

- [*The Same Model, Three Doors: Attributing Inference Reproducibility to the Serving Path*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) — the frontier-API version of section 4's lesson: the serving path is part of the system under test.
- [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) — the same workstation and engine lineage under full determinism control.
- [*The Trust Ladder*](https://isimplifyme.com/whitepapers/the-trust-ladder) — when a non-deterministic instrument may gate a merge; the planted-failure discipline section 2 applies to graders.
- *Mind the Cap: Output-Budget Regimes Change the Measured Multilingual Reasoning Gap.* arXiv:2608.04160 (2026).
- *Token Budget Saturation and Mechanistic Early-Detection of Reasoning Non-Convergence in Chain-of-Thought Models.* arXiv:2607.21433 (2026).
- *Wait, Wait, Wait... Why Do Reasoning Models Loop?* arXiv:2512.12895 (2025).
- Ganapaneni, S., *How Token Budgets Can Shift LLM Benchmark Accuracy by 70 Percentage Points.* Medium (2026).
- OpenAI, *How enabling two settings tripled our scores on the ARC-AGI-3 benchmark* (2026).
- [`output-cap-bench`](https://github.com/iSimplifyMe/output-cap-bench) — harness, items, graders, self-tests, and raw records for every call in this study.
