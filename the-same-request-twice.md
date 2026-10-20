> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-same-request-twice](https://isimplifyme.com/whitepapers/the-same-request-twice)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-07-28 · License: [CC BY 4.0](LICENSE)

# The Same Request Twice: Inference Reproducibility on Models With No Sampling Controls

A pre-registered measurement of AWS Bedrock inference reproducibility — 13,100 byte-identical requests across three Claude models, two routing scopes, two thinking modes, and three load windows, with the methods frozen in public before the data existed and every number re-derivable from the committed raw records.

---

## Abstract

> **Answer:** On current Claude models, the sampling controls are gone — `temperature`, `top_p`, and `top_k` return a validation error — so whatever variation remains is the platform's own, and we measured it. Sending the same request 100 times: field extraction and closed-set classification reproduced byte-identically in 5,988 of 5,988 valid calls; 400-word prose never reproduced even once; structured JSON sat in between, and turning adaptive thinking on cost 39 points of byte-level reproducibility on that task while leaving the parsed content identical. The frontier models varied their formatting; the budget-tier model varied its answers.

The question every regulated buyer eventually asks an AI vendor is some version of *will it say the same thing tomorrow?* This paper is the measured answer for one production platform — AWS Bedrock — across the current Claude family, under a pre-registration that was frozen, tagged, and pushed to a public repository before the first confirmatory call was made. The design, the equivalence margin, the exclusion rules, and the analysis code were all committed in advance; the raw per-call records are in the same repository; and one of our four pre-registered questions came back *inconclusive* under our own frozen test, which we report as such rather than repairing after the fact.

This is the empirical companion to [*The Finite Chain*](https://isimplifyme.com/whitepapers/the-finite-chain) and [*The Blind Panel*](https://isimplifyme.com/whitepapers/the-blind-panel): the same commitments — verification by deterministic code, guarantees by construction, statistics chosen before data — applied to a question about the inference substrate itself.

The reference implementation, pre-registration, and complete raw data are at [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — stdlib-only measurement, MIT, tag `prereg-v1` is the freeze.

---

## 1. The Knob Is Gone

> **Answer:** On the Claude 5 family, `temperature`, `top_p`, and `top_k` are removed — sending any of them returns a `ValidationException`, verified live on Bedrock during instrument checks. The folk method for determinism, `temperature = 0`, never guaranteed identical outputs and now cannot be attempted at all. The reproducibility of these models is therefore a fixed property of the serving platform, measurable but not configurable, and nobody had published the measurement.

For years, practitioners treated `temperature = 0` as a determinism switch. It never was one — greedy decoding still runs on floating-point arithmetic whose reduction order changes with server-side batch composition, and floating-point addition is not associative. A microscopic difference in low-order bits occasionally flips which token wins at a boundary where two continuations are close in probability, and everything downstream of that token follows the new branch.

What changed is that the switch is no longer there to flip. Our instrument checks confirmed the removal empirically on Bedrock: `temperature: 0.7` on Opus 5 returns a validation error, as does `thinking: disabled` combined with effort levels above `high` — both recorded in the repository's committed evidence files. On models like these, a deployment cannot configure its way toward determinism. It can only measure what the platform gives it, and design for that number.

One more discovery from live verification reshaped the study before it began: for all three models measured, Bedrock offers **no single-region on-demand invocation at all** — `inferenceTypesSupported` reports `INFERENCE_PROFILE` only. Every current deployment therefore chooses between routing scopes: the `us.` profile, which routes within United States regions, or the `global.` profile, which routes worldwide. That choice became our second question.

---

## 2. Methods Frozen Before the Data Existed

> **Answer:** The pre-registration — hypotheses, equivalence margin, sample size, exclusion rules, and the analysis script itself — was committed and tagged (`prereg-v1`) in a public repository 45 minutes before the first confirmatory call. Four validity gates were pre-registered: a positive control that must fire, a negative control on request hashes, a version-drift check on returned model IDs, and freeze-before-data ordering. All four held. A result produced this way can be checked; a result produced any other way must be trusted.

Evaluation claims in this industry are ordinarily unfalsifiable in practice: the methods are described after the results are known, which permits — and therefore invites — quiet adjustment until the numbers cooperate. The discipline this paper borrows is older than the industry: state what you will measure, how you will test it, and what would count as failure, in public, first.

Concretely, before any confirmatory data existed, the frozen document specified:

- **The four questions.** Q1: exact-match reproduction rate by model and task. Q2: does worldwide routing (`global.`) measurably change reproducibility versus US-bounded routing (`us.`), tested as equivalence with a 2-percentage-point margin. Q3: does adaptive thinking change final-answer reproducibility versus disabled thinking. Q4: does divergence correlate with time-of-day load.
- **The primary endpoint.** Per-cell *modal share*: the fraction of valid calls whose response text is byte-identical to the cell's most frequent response. Reported with 95% Wilson score intervals — the same machinery as [*The Blind Panel*](https://isimplifyme.com/whitepapers/the-blind-panel), deliberately.
- **The exclusion rules.** Errored calls and truncated stops excluded and counted, never silently dropped; any cell whose request hashes disagree is invalidated outright, because that would be harness variance rather than platform variance.
- **The controls.** A positive control on the one grid model that still accepts sampling parameters (Haiku 4.5 at `temperature 0.7`) that must show divergence for any null elsewhere to be interpretable; scheduled shuffling with a recorded seed and a rule that no two calls from the same cell are ever in flight concurrently, so a cell's repeats cannot ride in the same server-side batch and agree for the wrong reason.
- **The measurement code.** Committed at the freeze. No model judges anything anywhere in the pipeline — every metric is deterministic arithmetic, consistent with the stance published in [*The Finite Chain*](https://isimplifyme.com/whitepapers/the-finite-chain).

All measurement gates held: the positive control produced 100 distinct outputs in 100 calls; every one of the 121 analyzed cells had uniform request hashes; no returned model ID changed mid-run; no prompt-cache activity appeared anywhere. Twenty-one calls of 13,100 failed with server-side `ServiceUnavailableException` during one overnight window and were excluded and counted — no cell lost more than 4 of its 100 calls.

---

## 3. The Design

> **Answer:** A factorial grid: three models (Opus 5, Sonnet 5, and dated Haiku 4.5 as anchor), four tasks ordered by expected fragility (extraction, classification, structured JSON, open generation), both routing scopes (`us.` and `global.`), and — on the models that support it — adaptive versus disabled thinking, at 100 repeats per cell in each of three UTC load windows. 12,000 grid calls plus controls, all byte-identical within their cells by construction, with the SHA-256 of exactly the bytes sent recorded on every call.

The task ladder is what makes the result usable rather than merely interesting. A single pooled reproducibility number would average a task that always reproduces with one that never does, and describe nothing. The four rungs were chosen to isolate where determinism breaks:

| Factor | Levels |
|---|---|
| Model | `us./global.anthropic.claude-opus-5` · `claude-sonnet-5` · `claude-haiku-4-5-20251001-v1:0` |
| Task | extraction (one field from a fixed document) · classification (one label, closed set) · structured JSON (fixed schema) · open generation (~400 words of prose) |
| Routing scope | `us.` vs `global.` inference profile |
| Thinking | adaptive vs disabled (Claude 5 family only) |
| Load window | 15:00–19:00, 00:00–03:00, 07:00–10:00 UTC |

Three design constraints were forced by the API rather than chosen, and are worth recording because anyone replicating this will hit them. Effort is pinned at `medium` on the thinking arms because disabling thinking above effort `high` is rejected. Haiku 4.5 runs a single arm because adaptive thinking does not exist there — its roles are positive-control host and *dated anchor*: it is the only model in the grid whose returned model ID carries a version date, which makes it the only place a silent model roll would be visible. The Claude 5 models return undated IDs, so the version-drift control is blind on them — a limitation we mitigated by compressing each window's run into under two hours and disclose rather than paper over.

Sample size and the equivalence margin were set from a pilot, exactly as the draft pre-registration reserved: at n=100 per cell, the margin the study could support moved from the drafted 1 percentage point to 2, recorded with its power computation before the freeze. Total confirmatory cost, measured from per-call token usage at list prices: **$75.45**. The pilot cost $3.85. Rigor at this grain is not expensive.

---

## 4. The Ladder

> **Answer:** Reproducibility is not a property of a model; it is a property of a model *and a task shape*. Constrained outputs reproduced perfectly: 5,988 of 5,988 valid extraction and classification calls matched their cell's modal response exactly, across every model, scope, thinking mode, and window. Open-ended prose never reproduced: in every one of the 30 open-generation cells, no two of the ~100 responses were identical. Structured JSON fell in between — 53% to 97% — and which end of that range you get is a configuration choice.

The confirmatory numbers, pooled across windows and routing scopes (byte-identical modal share; n per arm ≈ 600):

| Task | Opus 5, thinking on | Opus 5, thinking off | Sonnet 5, thinking on | Sonnet 5, thinking off | Haiku 4.5 |
|---|---|---|---|---|---|
| extraction | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 |
| classification | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 |
| structured JSON | **0.533** | **0.928** | **0.590** | **0.975** | 0.798 |
| open generation | 0.010 | 0.010 | 0.010 | 0.010 | 0.010 |

Three readings of that table, in increasing order of consequence.

First, the perfect rows are genuinely perfect, and that is itself news. The classification fixture was deliberately written to straddle two categories — a support ticket that is arguably billing and arguably technical — precisely to probe near-boundary token decisions. Across 1,796 valid classification calls on that deliberately ambiguous input, the label never flipped once. When a decision is clear of the boundary at the final token, current Bedrock inference lands it identically every time.

Second, the open-generation row is a floor, not a dip. A modal share of 0.010 means the most common output appeared once — every response was unique. Divergence begins almost immediately (median first divergence at character 15 of roughly 1,800) and is structural rather than cosmetic: mean normalized edit distance from the modal response was 0.165 across cells. Long-form generation on these models is effectively a one-time function of the request. Any architecture that stores, diffs, caches, or legally relies on generated prose should assume zero call-to-call reproducibility, because that is the measured value.

Third, the structured JSON row is where engineering decisions live, and it is examined for the next two sections.

---

## 5. The Thinking Result

> **Answer:** Enabling adaptive thinking cost 39.5 points of byte-level reproducibility on structured JSON on Opus 5 (0.928 → 0.533) and 38.5 points on Sonnet 5 (0.975 → 0.590) — and the mechanism is not what the word "thinking" suggests. In the affected cells the models spent approximately zero thinking tokens; the reasoning trace did not push the answer around. The thinking *mode* itself destabilized a formatting decision, turning a 93-to-7 serialization preference into a near-fair coin.

Pooled across all tasks, the thinking effect is −9.9 points on Opus 5 (95% CI −12.5 to −7.3) and −9.6 on Sonnet 5 (−12.2 to −7.0) — but pooling understates what is happening, because the entire effect concentrates in one row of the ladder. On the tasks that always reproduce, thinking changes nothing. On the task that never reproduces, thinking changes nothing. On the production-shaped task in between, it is decisive.

Reading the actual variants explains it. Across 600 calls, Opus 5 with thinking enabled produced **exactly two** distinct outputs: the same JSON object serialized compactly (`"sku":"CS220-BDG-K"`) 320 times, and with spaces after separators (`"sku": "CS220-BDG-K"`) 280 times. With thinking disabled, the same model produced the same two variants at 557 to 43. One near-boundary decision — emit a space after the first colon or not — with everything downstream committing to whichever style won, by ordinary self-consistency. The thinking mode did not add reasoning that changed the answer; per-call token accounting shows the affected cells spent essentially no thinking tokens at all. The mode changed the sampling landscape enough to move one marginal formatting token from a settled preference to a coin flip.

Sonnet 5 adds a sharper version of the same phenomenon: among its variants were outputs wrapped in markdown code fences — **despite the prompt explicitly instructing no fences**. An instruction to a language model is a probability shift, not a constraint, and this dataset measures the residual rate at which a strongly-discouraged branch still fires under infrastructure noise. We note, with some satisfaction of the unwelcome kind, that during this study's own preparation week an unrelated production system in our fleet hit exactly this failure — intermittently fenced JSON that a downstream parser swallowed into an empty HTTP 200 — before this measurement existed to predict it.

---

## 6. Bytes Versus Meaning

> **Answer:** Every one of the 2,395 valid structured-JSON responses from the Claude 5 models — every variant, fenced or not, spaced or compact — parsed to the byte-identical object. At the semantic level, frontier-model reproducibility on this task was 100%. Haiku 4.5 was different in kind: in 7% of its calls, a *field value* changed — the same request nondeterministically extracted a different answer. On current pricing tiers, the frontier models put formatting at risk; the budget tier puts content at risk.

The primary endpoint of this study is byte-identity, deliberately: it is the strictest standard, the one a hash check or an evidentiary reproduction demand would apply, and it requires no judgment calls about which differences "matter." But the natural follow-up question is whether the divergence ever touched meaning, and the answer splits cleanly by model tier.

For Opus 5 and Sonnet 5, it never did. Parse every variant and compare objects: identical, all 2,395 of 2,395, across both thinking modes, both routing scopes, and all three windows. A pipeline that canonicalizes JSON at its boundaries — parse, then re-serialize — would have observed perfect reproducibility from the frontier models throughout this study.

Haiku 4.5 produced four distinct outputs, and two of them differ in substance: 558 calls extracted the item name one way, and 42 calls — 7.0% — extracted it with an extra leading word carried in from the source document's phrasing. Both readings are defensible; the fixture's wording is genuinely ambiguous at that boundary. Which is precisely the point, and the least comfortable finding in this study: **the flips concentrate where the judgment is closest.** A token only flips between runs when two continuations are near-tied in probability, and continuations are near-tied exactly when the underlying call is genuinely contestable. The clear-cut decisions in this dataset never flipped in 5,988 attempts. The arguable one flipped 7% of the time.

For any workflow where an extracted value feeds something that must be defended later — a filing, a valuation, an audit — this inverts the intuitive cost model. The nondeterminism does not sprinkle noise uniformly; it lands selectively on the calls a human reviewer would also have flagged as close. A system that runs judgment-bearing extractions once, on the cheapest tier, and treats the output as *the* answer has quietly built "which run did we happen to keep" into its evidence chain.

---

## 7. Routing, Load, and an Equivalence Test That Came Up Short

> **Answer:** Routing scope made no practically visible difference — observed us-versus-global gaps were 0.25 to 0.32 points with 90% confidence intervals inside ±2.6 points — but our pre-registered equivalence test did not formally conclude equivalence at its 2-point margin, and we report that outcome as registered: inconclusive. The cause is an estimator mismatch we froze into our own design. Time-of-day showed nothing: pooled reproduction rates across the three windows were 0.692, 0.696, and 0.693.

The honest section first, because it is the one pre-registration exists to force.

Q2 asked whether worldwide routing costs reproducibility relative to US-bounded routing, tested by two one-sided tests (TOST) at a 2-percentage-point margin. The observed differences were +0.29 points on Opus 5, +0.32 on Sonnet 5, −0.25 on Haiku — tiny, direction inconsistent, and each 90% confidence interval sits within ±2.6 points. But the frozen test computes its standard error from the pooled proportions, and pooled variance across a task ladder whose rates span 0.01 to 1.00 is roughly twice the stratified sampling error of the pooled estimate. The test we committed to is therefore conservative by construction, and it returned p ≈ 0.10 against the margin on all three models: **not equivalent at δ = 2pp, by our own registered criterion.**

A stratified estimator — the one our own power calculation used, a mismatch we did not catch at the freeze — bounds the routing effect within about ±1.2 points and would conclude equivalence comfortably. We report that as a labeled sensitivity analysis and nothing more, because swapping estimators after seeing data is the exact maneuver pre-registration exists to prevent. The confirmatory record shows a bounded null (±2.6 points at 90% confidence) that missed its registered margin; the methodological lesson — *register the estimator your power analysis assumed* — is recorded for the successor study. Both facts are more useful published than repaired.

Q4 needs less discussion: across a US business-hours peak window, a US evening window, and a US overnight window, pooled reproduction rates were statistically indistinguishable. Whatever drives the divergence measured in this paper, it was present at 3am Sunday and at Monday midday in equal measure. An exploratory effort sweep (five effort levels, adaptive thinking, on the ladder's most and least stable tasks) likewise moved nothing — with the honest caveat that it omitted the one task where configuration demonstrably matters, an omission the successor study inherits as a question.

---

## 8. Building Under Measured Nondeterminism

> **Answer:** The numbers imply a short design doctrine. Route structured outputs through disabled thinking where the effort budget allows it, or canonicalize serialization at every boundary so byte-instability cannot propagate. Validate semantics, not bytes — but log bytes, hashes, and returned model IDs for the audit trail. Treat model tier as a content-risk decision, not just a cost decision. And for judgment-bearing extractions, run the call N times: disagreement across runs is the platform telling you the judgment is close, which is a route-to-human signal, not noise to suppress.

Each recommendation traces to a measured row:

- **Constrained tasks can be trusted as deterministic in practice.** Zero flips in 5,988 calls is not a guarantee, but it is a measured floor strong enough to build on for extraction and classification against clear inputs.
- **Structured outputs: disable thinking or canonicalize.** Thinking-off took Sonnet 5 to 97.5% byte-identity; a parse-and-reserialize boundary takes either mode to 100% semantic identity. Do one deliberately rather than neither accidentally. Where thinking must stay on, treat any byte-level comparison, cache key, or hash gate on raw model output as a latent intermittent bug — this study measured its firing rate.
- **Never parse model text bare.** The fence variants arrived despite explicit instructions, at single-digit percentages, on one model family and not the other. That is exactly the profile of a bug that passes every smoke test and fails weekly in production. Strip fences, validate against schema, and fail loudly on mismatch.
- **Tier selection is a substance decision.** The 7% field-value flip rate on the budget tier versus zero semantic flips on the frontier tier reframes "which model can this workload afford" as "which workload can afford a nondeterministic answer."
- **Prose is one-shot.** Store what was generated; never expect to regenerate it. If two runs of a report generator must agree, the agreement has to be engineered — computed values injected into templates — because the platform will not provide it.
- **Instrument for the audit you might face.** Request-body hash, response hash, returned model ID, and request ID on every call cost nothing to record. On models whose returned IDs are undated, that record is also the only artifact that lets anyone reconstruct *what* served a disputed output.

For regulated deployments the compliance boundary is unchanged by any of this — AWS lists Bedrock as HIPAA-eligible with specific model-family exclusions — but reproducibility is the operational property auditors and opposing experts will eventually ask about, and "we measured it, here is the number, here is the mitigation" is a categorically better answer than a vendor's assurance.

---

## 9. Limitations

> **Answer:** The version-drift control is blind on the frontier models because Bedrock returns undated IDs for them; the positive control is cross-model because no model that rejects sampling parameters can host one; each task rung is a single fixture; the study spans one platform, one day, and one account; and the primary endpoint is deliberately stricter than semantic equality. Each limit is disclosed because a careful reader would find it anyway, and the successor study inherits the list as requirements.

Stated plainly:

1. **Undetectable point-version rolls.** `claude-opus-5` and `claude-sonnet-5` return exactly those strings on every call. If the underlying weights rolled mid-study, no response field would show it. Mitigations — compressed windows, the dated Haiku anchor, recorded AWS request IDs — reduce and bound the exposure; they do not eliminate it.
2. **The positive control validates the instrument, not the counterfactual.** Haiku at temperature 0.7 proves the pipeline detects divergence when divergence exists. It cannot prove Opus 5 *would have* diverged under sampling, because no such configuration exists to test.
3. **One fixture per rung.** The ladder's shape — perfect, intermediate, zero — is measured; how far each rung's number generalizes across prompts of the same shape is not. The Haiku field-flip in particular rode on a deliberately ambiguous fixture boundary.
4. **One platform, one snapshot.** These are Bedrock numbers from a single 19-hour confirmatory span. Whether the thinking-mode effect is a property of the models or of this serving stack is precisely the question a replication on a second platform answers, and that pre-registration is the successor study.
5. **Byte-identity is conservative.** It counts a space after a colon as divergence. We consider that the correct primary endpoint — it is the standard a hash, a cache, or a courtroom applies — and §6 reports the semantic reading alongside it.
6. **Q2's estimator mismatch**, reported in full in §7, is a self-inflicted design flaw preserved rather than patched, and the strongest argument this paper contains for registering analysis code and not merely analysis intentions.

---

## 10. Reproducing This

> **Answer:** Everything required to check or repeat this study is public: the pre-registration and its freeze tag, the stdlib-only harness and analysis code, the per-call raw records including request hashes and returned model IDs, and the committed reports. Total compute cost to reproduce the confirmatory grid at list prices is under $80. A reader who distrusts any number in this paper is invited to spend an afternoon and the price of a dinner recomputing it.

```bash
git clone https://github.com/iSimplifyMe/determinism-harness
git verify-tag prereg-v1 2>/dev/null; git log -1 prereg-v1   # the freeze, before the data
python3 -m unittest discover -s tests -t .                   # 74 tests, stdlib only
python3 -m harness.verify_profiles                            # your account's routing forms
python3 -m harness.smoke                                      # instrument checks incl. expected rejections
python3 -m harness.runner --mode pilot --window pilot         # ~$4
python3 -m analysis.analyze runs/*.jsonl --out reports
```

The committed `runs/` directory contains every confirmatory record: 13,100 calls with request SHA-256, response text and hash, returned model ID, token usage including per-call thinking-token counts, latency, attempt count, and AWS request ID. The analysis that produced every table in this paper is `analysis/analyze.py` at the tagged commit, and it refuses to emit a report if any cell's request hashes disagree — the same fail-loud posture argued for in [*Layer 4*](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering), applied to our own instrument.

The claim this paper makes for its method is modest and specific: not that these numbers are permanent — they describe one platform in one week — but that they are *checkable*, which published evaluation numbers in this field almost never are. Reproducibility of the measurement is the floor under any statement about reproducibility of the system.

---

## References and further reading

- [`iSimplifyMe/determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — pre-registration (tag `prereg-v1`), harness, analysis, and complete raw data; MIT
- [*The Blind Panel*](https://isimplifyme.com/whitepapers/the-blind-panel) — the statistical machinery (Wilson intervals, exact tests) and the freeze-before-data argument, applied to evaluation panels
- [*The Finite Chain*](https://isimplifyme.com/whitepapers/the-finite-chain) — verification as deterministic code rather than a second model
- [*Layer 4: Reliability Engineering for Regulated AI*](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) — the guardrail and fail-loud architecture this study's harness applies to itself
- He, H. et al., *Defeating Nondeterminism in LLM Inference* (Thinking Machines Lab, 2025) — the batch-invariance analysis of why identical requests diverge under load
- Goldberg, D., *What Every Computer Scientist Should Know About Floating-Point Arithmetic* (ACM Computing Surveys, 1991) — the non-associativity that starts the causal chain
- [AWS HIPAA Eligible Services Reference](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/) — Bedrock's eligibility scope and model-family exclusions
