> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-tokenizer-is-a-fingerprint](https://isimplifyme.com/whitepapers/the-tokenizer-is-a-fingerprint)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-08-22 · Revised 2026-08-31 · License: [CC BY 4.0](LICENSE)

# The Tokenizer Is a Fingerprint: Identifying the Lab Behind a Stealth Model from Token Counts Alone

A stealth frontier model appeared on a public aggregator with its operator undisclosed. Using only the token counts the API must return to bill you — 126 calls, $0 — we identified the vocabulary it runs, the lab that vocabulary belongs to, and the generation it implies, and registered the attribution before anyone claimed the model.

> **Update — 2026-08-23 (the morning after publication).** Community reports named GLM-5.3 as a tokenizer match, so we added Zhipu's released GLM-5 vocabulary to the candidate set — a release newer than our original set's GLM-4.5. Result: **95 of 95 measured counts exact, mean absolute error 0.00.** Ox Alpha runs Zhipu's current released vocabulary on every probe, and the original battery's eleven GLM-4.5 misses were GLM-5's shipped improvements, not an unreleased extension. The vocabulary match upgrades from lineage to identity; the sections below carry dated corrections where the original interpretation is superseded — including the abstract's "more efficiently than any released GLM," which was true of the 2026-08-22 candidate set and is not true of GLM-5. Full follow-up receipt: `results/glm5_followup.json` in the companion repo.

> **Update II — 2026-08-23, later the same day (Battery II: the runtime fingerprint).** A second battery compared three stacks — Ox Alpha, Zhipu's own served GLM-5.2, and a generic local implementation of the same vocabulary — on chat-template structure and 29 divergence-prone inputs. The template grammar is identical across Ox and Zhipu's endpoint, down to the same rejection of malformed input; 25 of 29 divergence probes agree three ways — the four exceptions are exactly the probes embedding a real special literal — and every divergence found anywhere localizes to a single layer, special-token input policy, where this battery also corrects §5's original interpretation. Receipts: `results/battery2.json`.

---

## Abstract

> **Answer:** An anonymous model's tokenizer is identifiable from outside, without logprobs or insider access, by differential token counting: the hidden chat template adds a constant overhead, so `prompt_tokens(BASE + probe) − prompt_tokens(BASE)` is the model's own count for the probe string. Over 95 probes against fourteen candidate vocabularies, the stealth model Ox Alpha matched GLM's vocabulary on 84 of 95 counts exactly; the best non-GLM candidate managed 46.

Every one of the eleven misses tokenizes more efficiently than any released GLM — the signature of a next-generation vocabulary. We register the attribution as a prediction: high confidence that Ox Alpha is a next-generation GLM model from Zhipu AI, unconfirmed by any party as of 2026-08-22.

On August 20, 2026, a stealth model called Ox Alpha appeared on OpenRouter and opencode: one-million-token context, image and video input, free for a week, operator undisclosed. The API's own responses carry `"provider": "Stealth"`. Within 48 hours, press speculation attributed the model to Zhipu AI on the strength of informal "fingerprints." We tested the attribution ourselves, using nothing but the `usage.prompt_tokens` field of the public chat-completions API.

Tokenizer identity establishes lineage; the operator is a separate, softer inference, since vocabularies can be licensed or forked. The durable contribution is the method — the verdict is its first worked example. Any team routing work to an anonymous endpoint can run this battery in an afternoon for approximately zero dollars.

---

## 1. Why Does Provenance Matter?

> **Answer:** Stealth model previews ask users to send prompts to an endpoint whose operator, jurisdiction, and retention practices are all undisclosed — a "zero data retention" claim with no counterparty to audit. For a firm weighing client work or regulated data against that endpoint, "who is actually serving this?" is a due-diligence question, and it is answerable from the API's own billing metadata.

Stealth previews are now a standard go-to-market motion: an unlabeled model appears on an aggregator, harvests real-world feedback at scale, and is claimed weeks later. During the window, the terms of the trade are one-sided. An individual developer kicking tires may accept them. A firm with obligations cannot evaluate them, because there is nobody to evaluate.

The operational rule this paper suggests: before routing regulated or proprietary work to any unclaimed endpoint, run the battery. A confident match converts an unauditable retention promise into a named counterparty with a jurisdiction, a track record, and published terms. A mismatch means the endpoint is not what the rumor mill says it is — which is exactly when you want to know.

---

## 2. The Differential Trick

> **Answer:** Every chat-completions request reports `usage.prompt_tokens` — the size of the fully templated prompt under the provider's real tokenizer. The template overhead is unknown but constant, so `count(probe) = prompt_tokens(BASE + probe) − prompt_tokens(BASE)` recovers the tokenizer's count for the probe string exactly. Computing the same differential locally for each candidate vocabulary on the identical concatenation cancels boundary-merge effects too, so every candidate faces the same test. Each probe costs one API call at `max_tokens: 1`.

Ox Alpha's template overhead is ≈90 tokens, 64 of them served from cache. The subtraction cancels it exactly.

Probe strings are chosen to *separate* vocabularies: different BPE merge tables disagree most on non-Latin scripts, code punctuation, digit grouping, emoji ZWJ sequences, and rare Unicode. Two vocabularies that agree on English prose diverge within a handful of Chinese, Thai, or regex probes. The full set spans two dozen natural languages, fifteen code probes across thirteen formats, structured data, and a battery of Unicode edge cases — 95 strings in all.

The entire attribution ran in 126 calls and $0 (free preview window); at paid rates it would cost cents. The template baseline was measured before the original battery and re-verified unchanged before the expanded battery. A drifting template would invalidate the merge, so the harness refuses to combine runs across a baseline change.

One fair objection deserves a pre-emptive answer: is `usage.prompt_tokens` the upstream provider's own metering, or an aggregator-side estimate? The data settles it. The measured counts match one specific vocabulary at 84/95 exactness while every other candidate misses by whole tokens on most probes — no generic estimator produces that pattern — and the responses carry upstream cache granularity (`cached_tokens: 64`) that an aggregator-side approximation would not report. The battery is also non-adversarial by construction: it counts the provider's own billed tokens on the requester's own prompts.

---

## 3. Fourteen Candidates, Thirteen Distinct

> **Answer:** The candidate set covered every lab the speculation named plus outgroups: GLM-4.5 and GLM-4 (Zhipu), Qwen3 and Qwen2.5 (Alibaba), Xiaomi MiMo, DeepSeek-V3.1, Kimi K2 (Moonshot), MiniMax-M1, Yi-1.5 (01.AI), Hunyuan (Tencent), ERNIE-4.5 (Baidu), Seed-OSS (ByteDance), Llama-3.1 (Meta), and GPT-OSS (OpenAI). One candidate eliminated itself before any API call: Xiaomi MiMo's tokenizer file is byte-identical to Qwen2.5's, so the "maybe it's Xiaomi" theory was never a separate hypothesis at the vocabulary level.

Candidate vocabularies were pulled directly from their public model repositories; Kimi K2 ships a tiktoken-format table rather than a `tokenizer.json`, so it was reconstructed with its published pre-tokenization pattern rather than approximated. MiMo's byte-identity to Qwen2.5 (same MD5) is why fourteen candidates are thirteen distinct hypotheses — and it means the Xiaomi theory lives or dies with Qwen, which loses decisively below.

---

## 4. The Ranking

> **Answer:** GLM's vocabulary predicted 84 of 95 measured token counts exactly at a mean absolute error of 1.08; the best non-GLM candidate, Llama-3.1, managed 46 of 95 at 3.83. GLM is exact on every English, Chinese, Japanese, Korean, Arabic, Russian, and European-language probe, all fifteen code probes, and every structured-data probe. The eleven misses concentrate in Devanagari, Bengali, Thai, emoji, and combining marks — and on those probes the nearest competitor is a different vocabulary almost every time, each catastrophic outside its pocket, so no alternative lineage explains the misses.

95 probe strings; differential counts measured against `stealth/ox-alpha` on 2026-08-22, `pt_base` re-verified stable (92 tokens) before the expanded run.

| Candidate vocabulary | Exact matches | MAE (tokens) |
| --- | --- | --- |
| **GLM-4.5** (Zhipu) | **84/95** | **1.08** |
| **GLM-4** (Zhipu, 2024) | **84/95** | **1.08** |
| Llama-3.1 (Meta) | 46/95 | 3.83 |
| Hunyuan (Tencent) | 33/95 | 5.25 |
| GPT-OSS (OpenAI) | 32/95 | 5.45 |
| MiMo (Xiaomi) | 26/95 | 5.14 |
| Qwen2.5 (Alibaba) | 26/95 | 5.14 |
| Qwen3 (Alibaba) | 26/95 | 5.14 |
| Kimi K2 (Moonshot) | 23/95 | 4.48 |
| DeepSeek-V3.1 | 19/95 | 5.28 |
| MiniMax-M1 | 16/95 | 4.93 |
| Seed-OSS (ByteDance) | 14/95 | 6.44 |
| ERNIE-4.5 (Baidu) | 13/95 | 7.82 |
| Yi-1.5 (01.AI) | 6/95 | 19.46 |

The gap is not close. GLM predicts the measured count *exactly* on every English, Chinese (five registers, simplified and traditional), Japanese, Korean, Arabic, Russian, and European-language probe; on all fifteen code probes across thirteen formats (Python, TypeScript, Rust, Go, Java, C, HTML, CSS, YAML, TOML, Dockerfile, shell, SQL); and on every structured-data, number-format, and identifier probe. Note also that MiMo, Qwen2.5, and Qwen3 score identically: MiMo's vocabulary file is byte-identical to Qwen2.5's, and the probe set confirms the collapse empirically.

The eleven GLM misses concentrate entirely in one region — Hindi (+8, +7), Thai (+11, +14), Bengali (+21), emoji ZWJ chains (+14 twice), combining marks (+10), and near-misses of 1–2 tokens on IPA and mathematical Unicode — where "+n" means GLM-4.5 predicts n *more* tokens than Ox Alpha actually used.

On those eleven probes the nearest competitor is a different vocabulary almost every time: the Qwen family comes closest on Devanagari and Bengali, Kimi on Thai, DeepSeek on one emoji probe, GPT-OSS and MiniMax on the IPA and math-symbol near-misses, with the ZWJ and combining-mark probes falling to the same scattered pattern. Each of those candidates loses catastrophically everywhere outside its pocket, so no alternative lineage explains the misses. And all eleven deviations point the same direction: better compression than any GLM release we tested.

> **Update — 2026-08-23.** The candidate set above was assembled with GLM-4.5 as Zhipu's newest release; it was not. Against the released GLM-5 vocabulary (`zai-org/GLM-5.2`, byte-identical to `zai-org/GLM-5` — MD5 pair in `results/glm5_followup.json`), Ox Alpha scores **95/95 exact at MAE 0.00** — the eleven "misses" were GLM-5's shipped improvements to exactly these scripts. The scattered-competitor analysis above still holds for the original candidate set; the stronger, simpler statement is that the measured vocabulary is Zhipu's released vocabulary, exactly.

---

## 5. The Special-Token Battery, and What Escaping Tells You

> **Answer:** Twenty-five special-token literals sent as user content all failed to collapse to single tokens: the endpoint escapes special tokens in user input, which is itself a fingerprint — that is the sanitization GLM's own serving stack performs. Modeled jointly (GLM's vocabulary with its added tokens forced to plain-text segmentation), the prediction matches 23 of 25 measured counts exactly. The two deviations are `<think>` and `</think>`, exactly the tokens a lab promotes to true specials when reasoning graduates to a first-class serving mode.

The battery sent `[gMASK]`, `<sop>`, `<|user|>`, `<|im_start|>`, `<|begin▁of▁sentence|>`, harmony tags, Llama headers, and the rest of the candidate labs' special inventories as ordinary user content. None collapsed to a single token.

When the escaping is modeled correctly — GLM's vocabulary with its added tokens forced to plain-text segmentation — the prediction matches **23 of 25** measured counts exactly, including GLM's own `[gMASK]`→3, `<sop>`→3, `<eop>`→4, `<|observation|>`→5, `<|endoftext|>`→7. The two deviations are `<think>` and `</think>` (measured 3; GLM-4.5's vocabulary holds each as one ordinary token). The parsimonious reading: in Ox Alpha's generation of the vocabulary, the thinking delimiters were promoted to true special tokens, and the endpoint escapes them like the others. Ox Alpha is observably a reasoning model — it emits a reasoning stream before every reply — which makes the promotion expected rather than anomalous.

> **Correction — 2026-08-23 (Battery II).** This section originally read the escaping as "the sanitization GLM's own serving stack performs." Measured directly, it is not: Zhipu's public GLM-5.2 endpoint performs no special-token sanitization at all — every special we measured against it — five, including `<|endoftext|>` — passes through raw as its single token. The escaping is Ox-specific hardening. The fingerprint logic inverts but strengthens: Ox segments specials exactly as GLM's vocabulary would under an escaping policy the released Zhipu endpoint lacks — an input-hardening layer in front of a GLM-5-family runtime, of the kind a lab adds before opening a model to a massive anonymous free preview.

---

## 6. Which GLM Generation Is It?

> **Answer:** GLM-4 and GLM-4.5 share their text vocabulary and tie at 84/95, so the text differential alone cannot name the generation. Two independent signals resolve it, both pointing forward: every miss is an efficiency gain concentrated in Devanagari, Bengali, Thai, and emoji — a targeted vocabulary refresh, since Tamil matched released GLM exactly — and the `<think>`/`</think>` delimiters were promoted from ordinary tokens to escapable specials. A fork does not improve the tokenizer; a next generation does.

Two observations carry the generational argument:

1. **The misses are all efficiency gains.** Hindi (measured 62 vs 70 predicted), Thai (58 vs 69), Bengali, emoji ZWJ chains (30 vs 44), combining marks — every deviation from released GLM is a *better* compression of scripts a lab extends when refreshing a vocabulary for a global audience. The extension is also selective: Tamil matched released GLM exactly while Devanagari, Bengali, and Thai improved, which reads like a targeted vocabulary refresh rather than a wholesale replacement.
2. **The `<think>`/`</think>` promotion.** Released GLM-4.5 carries the thinking delimiters as ordinary vocabulary entries; Ox Alpha's endpoint treats them as escapable specials, like `[gMASK]` and `<sop>` — the upgrade you make when reasoning becomes a first-class serving mode.

Whoever built Ox Alpha's tokenizer started from GLM's merge core and extended it. That is not what borrowing looks like; it is what shipping the next version looks like.

> **Update — 2026-08-23.** Signal 1 is superseded: the "extension" is the released GLM-5 vocabulary itself (95/95 exact), so the vocabulary no longer separates generations beyond GLM-5. Signal 2 survives and sharpens: released GLM-5.2's `tokenizer_config` declares 18 special tokens — including the `<|begin_of_video|>`/`<|end_of_video|>` delimiters that match Ox Alpha's video modality — but does **not** declare `<think>`/`</think>` special, and Ox Alpha's endpoint escapes them anyway. The observation that survives every released candidate: Ox Alpha uses the GLM-5 vocabulary exactly and treats the reasoning delimiters as specials where the released GLM-5.2 configuration does not. The inference — stated separately because the data cannot rank Zhipu's internal variants — is an internal or later GLM-5-family reasoning runtime rather than the publicly released 5.2 configuration.

> **Update II — 2026-08-23.** Battery II sharpened the runtime observation. Ox and Zhipu's served GLM-5.2 share identical chat-template grammar — the same per-role token constants (+5 system, +11 assistant turn, +6 second user, +1 empty system) — and identical rejection of empty user content. 25 of 29 divergence probes agree three ways, including the corrupted-special probes (case variants, fragments), which Ox does not escape; the four divergences are exactly the probes embedding a real special literal — the hardening is exact-match surgical. Every divergence found in either battery localizes to that one layer: Ox escapes special tokens in user content — all of GLM's declared specials plus `<think>`/`</think>` — where the released endpoint escapes none. The inference is unchanged in kind and stronger in evidence: a GLM-5-family runtime behind a hardened, reasoning-aware input layer not present in the released GLM-5.2 configuration.

---

## 7. Behavioral Corroboration

> **Answer:** Behavioral probes agree with the vocabulary without being decisive alone: the model self-identifies only as "ox-alpha, developed by an undisclosed organization" in English and Chinese alike; its claimed knowledge cutoff wobbles between "October 2024" and "early 2025" across two probes while its actual knowledge stops at December 2024 releases; it writes native-quality literary Chinese; and its serving metadata — text, image, and video input at a 1,048,576-token context — matches the modality speculation in press coverage.

Five behavioral probes point the same direction as the vocabulary:

- **Self-identification** (English and Chinese, consistent): "I am **ox-alpha**, an LLM developed by an undisclosed organization."
- **Claimed cutoff:** "October 2024" to the direct probe — and "early 2025" in a neighboring answer about recent model releases. The self-report wobbles across two questions; the tokenizer gives one answer every time. Either way, the model knows Gemini 2.0 Flash, o1, DeepSeek-V3, and Llama 3.3 (all December 2024) and nothing later, consistent with a base trained on data through late 2024.
- **Chinese fluency** at native literary register, including correct classical allusion — expected of a Chinese lab's flagship, unremarkable for its class, but consistent.
- **Serving metadata:** text, image, and **video** input; 1,048,576-token context; 131,072-token max completion (models-endpoint record captured in the companion repo as `results/model_metadata.json`), matching the modality speculation in press coverage.
- **Introspection:** asked to print its native tool-call token syntax, the model declined and correctly explained that it cannot introspect its own token stream — a well-trained answer that leaks nothing.

None of these is decisive alone; all of them agree with the vocabulary.

---

## 8. What This Method Cannot Tell You

> **Answer:** Tokenizer identity is strong evidence of lineage, not proof of operator: a third party could serve a licensed or fine-tuned derivative, measurements are point-in-time, and escaping means vocabulary membership is only answerable jointly with an escaping model. The method can also be defeated — an operator could corrupt its `usage` reporting — but `usage` is the billing meter, so defeating the fingerprint costs billing transparency, the one credibility an anonymous provider cannot easily abandon.

Five limits bound the claim:

1. **Vocabulary ≠ operator.** A tokenizer identifies lineage. A third party could serve a licensed or fine-tuned GLM derivative. The capacity figure circulating in the launch coverage — 100 trillion tokens per day, from the opencode announcement — and the extended vocabulary both argue for the lab itself, but that inference is softer than the tokenizer match. *Follow-up note, 2026-08-23: with the vocabulary now known to be released, it no longer argues for the lab itself — a public vocabulary is easier, not harder, for a third party to serve. The operator inference rests on the runtime signal, the capacity figure, and behavior.*
2. **Point-in-time.** A stealth operator can change template, tokenizer, or backend mid-window. Every raw response is pinned in the companion repo with timestamps.
3. **Escaping can mask vocabulary membership.** Because the endpoint sanitizes special tokens, "is this literal in the vocab as a special?" is only answerable jointly with an escaping model, as above.
4. **The method can be defeated — at a price.** An operator determined to hide could pad, coarsen, or randomize `usage` reporting so that token counts no longer reflect the real tokenizer. But `usage` is the billing meter: breaking its fidelity means charging customers on numbers that no longer mean anything, and honest metering is the one credibility an anonymous provider cannot easily abandon. Defeating the fingerprint costs billing transparency.
5. **The prediction can be wrong.** If Ox Alpha is claimed by someone other than Zhipu, this paper's method survives and its worked example becomes a cautionary tale about confidence calibration. We are comfortable with that trade, and we commit to updating this page with the outcome either way.

---

## 9. Sidebar: What Else Fell Out of the Battery

> **Answer:** Temperature 0 does not buy determinism on this endpoint — open prose produced fifteen unique outputs in fifteen identical greedy runs, the batched-MoE serving signature studied in our determinism series. And on a 12-task auto-graded coding eval, Ox Alpha passed 11/12 against 10/12 for Zhipu's current public flagship GLM-5.2, whose both misses were reasoning-budget blowouts at the identical cap — the stealth model fits its reasoning where its alleged ancestor overruns.

Two byproducts of the battery are worth recording:

- **Temperature 0 does not buy determinism on this endpoint.** Fifteen identical greedy runs per prompt: the fixed-answer prompt was stable (1 unique output in 15), a code prompt was not (4 unique), open prose produced fifteen unique outputs in fifteen runs, and the reasoning streams almost never repeat (7–15 unique per prompt). This is the batched-MoE serving signature examined in [*The Same State, The Same Bytes*](https://isimplifyme.com/whitepapers/the-same-state-the-same-bytes) and [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines); it joins the dataset of that series rather than this paper.
- **Capability, briefly.** On a 12-task auto-graded coding eval (self-validating grader; JS/Python/SQL), Ox Alpha passed 11/12 with one genuine correctness bug; Zhipu's current public flagship GLM-5.2 passed 10/12, with both misses being reasoning-budget blowouts at the identical 16,000-token cap. (A first baseline attempt against the rate-limited `:free` endpoint was discarded for retry exhaustion — 9 of 12 calls never completed; the paid endpoint is what we report, and both runs are in the companion repo.) The stealth model fits its reasoning where its alleged ancestor overruns — consistent with a next-generation sibling, and with the benchmark claims in the preview's press coverage.

---

## 10. Reproducing This Study

> **Answer:** The companion repository carries every probe string, every raw API response, the candidate fetch script, the ranking code, and the coding-eval grader with its self-test. One environment variable and a short sequence of commands, mirrored in the README, regenerate every number in this paper. The free-window cost was $0; at paid rates the battery costs cents.

Measurements were taken 2026-08-22 against `stealth/ox-alpha` via OpenRouter. As throughout this series, the claim format is the point: not "trust our lab," but "here are the bytes; run it."

---

## References and further reading

- [*The Same State, The Same Bytes*](https://isimplifyme.com/whitepapers/the-same-state-the-same-bytes) — the batched-serving nondeterminism this paper's sidebar observed from outside, studied under full control.
- [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) — the control ceiling for reproducibility on owned hardware.
- [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) — the serving path as part of the system under test; this paper treats the tokenizer the same way.
- [*The Ceiling Is Not the Model*](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model) — harness configuration deciding benchmark results; the coding-eval sidebar here applied its triage discipline.
- opencode announcement of the Ox Alpha preview window (August 20, 2026) — the free-week terms and the capacity figure.
- OpenRouter model listing for `stealth/ox-alpha` — context length, modalities, and provider designation captured in the companion repo as `results/model_metadata.json`.
- [`tokenizer-fingerprint`](https://github.com/iSimplifyMe/tokenizer-fingerprint) — probes, harness, candidate fetch script, and raw records for every call in this study.

---

*Registered prediction, stated 2026-08-22, before any official attribution: Ox Alpha is a next-generation GLM model trained by Zhipu AI (Z.ai). Confidence: high. This page will be updated with the outcome when the model is claimed.*

*Update, 2026-08-23: the candidate set was extended with Zhipu's released GLM-5 vocabulary — 95/95 exact, MAE 0.00. The prediction is unchanged and the attribution is strengthened: Ox Alpha carries Zhipu's current released vocabulary exactly, served by a reasoning-aware runtime not represented by the released GLM-5.2 configuration.*

*Update II, 2026-08-23: a second battery (`results/battery2.json`) found Ox Alpha and Zhipu's served GLM-5.2 identical in chat-template grammar and edge-case tokenization, with every observed divergence localized to Ox's special-token input hardening — and corrected §5's reading of the escaping accordingly.*

**Outcome, 2026-08-31: the registered prediction was right.** On 2026-08-26 Zhipu publicly identified Ox Alpha as **GLM-5.3-Flash** — the first natively multimodal model of the GLM-5 series — and released the model weights. The attribution this page registered on 2026-08-22, before any official claim, named the right lab and the right reading of the signals: a next-generation GLM model. The registered call, the two dated follow-ups above, and this note complete the loop this page committed to on publication day.*
