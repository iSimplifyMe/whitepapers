> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-same-model-three-doors](https://isimplifyme.com/whitepapers/the-same-model-three-doors)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-07-29 · License: [CC BY 4.0](LICENSE)

# The Same Model, Three Doors: Attributing Inference Reproducibility to the Serving Path

A pre-registered attribution study — 13,950 byte-identical requests sending the same Claude models through three serving planes at once: AWS Bedrock, Claude Platform on AWS, and Anthropic's first-party API. The reproducibility cost this study's predecessor measured turns out to live mostly in the door, not the model.

---

## Abstract

> **Answer:** The thinking-mode reproducibility cost follows the serving door, not the operator, and not (mostly) the model. Sending the same frozen task ladder through three doors simultaneously: on the first-party API the effect is a null for Opus 5 (+0.3 points, CI −2.1 to +2.8) and −13.3 points for Sonnet 5; through both AWS-fronted doors it is −16.3 to −43.8 points; and the two AWS doors — one operated by AWS, one operated by Anthropic — are statistically equivalent to each other at a 2-point margin. That answers the question [*The Same Request Twice*](https://isimplifyme.com/whitepapers/the-same-request-twice) had to leave open when it measured adaptive thinking costing ~39 points on structured JSON on one platform: the cost belongs mostly to the serving path.

The predecessor paper closed on a limitation it could not escape: "whether the thinking-mode effect is a property of the models or of this serving stack is precisely the question a replication on a second platform answers." This is that replication, pre-registered as its own study (tag `prereg-v2`, frozen and pushed 25 seconds before the first confirmatory call), with the estimator lesson from the first study's own inconclusive result registered as the primary test statistic this time.

The design is the same discipline applied wider: identical prompts, byte-canonical requests, deterministic measurement, raw records public. What is new is the factor under test — the *serving plane* — and one instrument upgrade it forced: because SDK clients own their own wire serialization, the harness now records the SHA-256 of the bytes actually sent on every call, captured at the HTTP layer, so the negative control is verified rather than assumed on every plane.

The reference implementation, both pre-registrations, and complete raw data are at [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — stdlib-only measurement, MIT; `prereg-v1` froze the first study, `prereg-v2` froze this one.

---

## 1. The Question the First Paper Could Not Answer

> **Answer:** A single-platform measurement cannot distinguish a model property from a serving-stack property — the two are confounded by construction. The current Claude models are reachable through genuinely distinct serving paths: Amazon Bedrock (AWS-operated), Claude Platform on AWS (Anthropic-operated, behind an AWS front door, SigV4-authenticated), and the first-party Anthropic API. Same weights, same request semantics, three infrastructures. That is the natural experiment this study runs.

The first study's headline was mechanically precise but causally ambiguous. Adaptive thinking turned a settled serialization preference into a near-fair coin on structured JSON — measured on Bedrock, at essentially zero thinking tokens, so the *mode* rather than any reasoning was doing the destabilizing. But nothing in that data could say whether the coin lived in the model or in the serving stack underneath it.

The three planes make attribution possible because they differ in exactly the dimension of interest. If the thinking-mode effect is model-intrinsic, it should appear at comparable magnitude through every door. If it is serving-stack behavior, it should attenuate or vanish on planes that do not share the stack. And the planes themselves offer a second, sharper contrast: Claude Platform on AWS is *Anthropic-operated* but sits behind the same AWS front door as Bedrock — so if the effect tracks the operator, those two should differ; if it tracks the path, they should match.

One point of honesty about what "same request" means across planes: the request bodies differ by construction. Bedrock's InvokeModel carries an `anthropic_version` field and no `model` field; the Messages-API planes carry `model` and no `anthropic_version`. The harness enforces — with a unit test — that these are the *only* structural differences. Cross-plane identity is semantic; byte-identity claims are always within-cell, within-plane, where they are verified per call.

---

## 2. Frozen First, Again — This Time With the Right Estimator

> **Answer:** The pre-registration was frozen, tagged `prereg-v2`, and pushed to the public repository at 19:52:24 UTC; the first confirmatory call went out at 19:52:49 UTC, 25 seconds later. The registered primary test for plane equivalence is the stratified estimator the first study's power analysis assumed but its frozen test did not use — the exact mismatch that left the first study's Q2 inconclusive, corrected here in advance rather than after seeing data. All validity gates held, including the new wire-byte negative control: not one of the study's cells sent varying bytes.

Its predecessor ended with a self-inflicted methodological finding: its equivalence test froze a pooled-variance standard error roughly twice the true stratified sampling error, and duly returned *inconclusive* at a margin the data plainly satisfied. The lesson — register the estimator your power analysis assumed — was recorded there as an obligation on the successor. This study discharges it. The stratified estimator takes the equal-weight mean of per-stratum differences with per-stratum binomial variances, a stratum being a matched model-task-thinking-window cell; it was implemented, unit-tested, and committed *before any confirmatory data existed*, with the pooled test demoted to a labeled sensitivity. The test suite includes a regression test that reproduces the first study's failure mode as an executable fact: on strata near 0 and 1, the pooled standard error exceeds twice the stratified one and the two tests return opposite verdicts at the same margin.

A pilot (n=20 per cell, 1,200 calls, $5.98) ran before the freeze and is disclosed in the pre-registration as exploratory — its point estimates confirmed feasibility and left every registered parameter unchanged: n=100 per cell, equivalence margin 2 percentage points, two load windows.

The gates, as registered and as they held:

- **Positive control, per plane.** Haiku 4.5 at `temperature 0.7` — the one grid model that still accepts sampling parameters — must show divergence on each plane independently. It fired 100 distinct outputs in 100 calls on all three.
- **Negative control, upgraded to the wire.** The first study hashed the canonical bytes it constructed and sent. On SDK planes the client library owns serialization, so this study captures the actual outbound bytes with an HTTP-layer request hook and records their hash on every call. Every cell on every plane was byte-uniform across its repeats — the SDK's determinism is now a measured property, not an assumption.
- **Version drift.** No returned model ID changed mid-study. The Claude 5 models return undated IDs on all three planes — the drift-blindness is ecosystem-wide, not a Bedrock quirk — so the dated Haiku build (`claude-haiku-4-5-20251001`) rides along on every plane as the one place a silent roll would be visible.
- **No cache contamination.** Zero prompt-cache reads or writes anywhere in 13,950 calls.
- **Completeness.** Every scheduled call produced a record. 85 of 13,950 (0.6%) failed terminally and were excluded and counted — all of them on one plane, in one window, discussed in §7.

---

## 3. Three Doors, One Request

> **Answer:** The confirmatory grid: three planes, the frozen four-task ladder, adaptive versus disabled thinking on Opus 5 and Sonnet 5, dated Haiku 4.5 as anchor, 100 repeats per cell, two UTC load windows — 12,000 grid calls, plus 300 positive-control calls, a 1,200-call streamed-delivery arm, and a 450-call input-length arm. Bedrock is pinned to its `us.` inference profile; the study ran from one account, one AWS region, one workspace, in a 22-hour span. Total confirmatory cost at list prices: $115.20.

| Factor | Levels |
|---|---|
| Plane | Bedrock (`us.anthropic.claude-*`, InvokeModel) · Claude Platform on AWS (bare IDs, SigV4) · Anthropic API (bare IDs, API key) |
| Model | `claude-opus-5` · `claude-sonnet-5` · `claude-haiku-4-5-20251001` (anchor) |
| Task | extraction · classification · structured JSON · open generation |
| Thinking | adaptive vs disabled (Claude 5 family; effort pinned `medium`) |
| Window | 15:00–19:00 UTC (peak) · 07:00–10:00 UTC (overnight) |

Two windows rather than the first study's three, because time-of-day was measured flat there (0.692 / 0.696 / 0.693) and the freeze records that justification. The first study's constraints carried over and replicated on the new planes during instrument checks: `temperature` on Opus 5 is rejected on all three doors, as is disabled thinking above effort `high` — the design's expected-rejection smoke cases passed identically everywhere, which is itself a small parity finding.

The task ladder's outer rungs behaved exactly as before, on every plane: extraction and classification reproduced byte-identically in every valid call, and open generation never reproduced once — modal share 0.01 (the fraction of a cell's valid calls matching its most frequent response; this study's primary endpoint throughout) in all 36 of its cells, 12 per plane, grid and streamed alike. Whatever differs between these serving paths, it does not touch the rungs where determinism is already settled, in either direction. The action, as before, is structured JSON.

---

## 4. The Effect Lives at the Door

> **Answer:** On structured JSON (n=200 per arm per plane), the adaptive-minus-disabled reproducibility effect was: Opus 5 — first-party **+0.3 points** (CI95 −2.1 to +2.8, a null), Bedrock **−22.8** (−28.4 to −17.2), Claude Platform on AWS **−16.3** (−21.1 to −11.5). Sonnet 5 — first-party **−13.3** (−17.2 to −9.5), Bedrock **−43.8** (−49.9 to −37.8), Claude Platform on AWS **−39.7** (−45.7 to −33.6). The difference-of-differences between the first-party door and each AWS door excludes zero for both models. For Opus 5 the effect is entirely a serving-path phenomenon; for Sonnet 5 a smaller intrinsic component survives on the first-party API and the AWS path roughly triples it.

The per-plane rates, pooled across both windows:

| Structured JSON, byte-identical modal share | thinking on | thinking off | effect |
|---|---|---|---|
| Opus 5 — first-party API | 0.983 | 0.980 | **+0.003** |
| Opus 5 — Claude Platform on AWS | 0.817 | 0.980 | −0.163 |
| Opus 5 — Bedrock | 0.737 | 0.965 | −0.228 |
| Sonnet 5 — first-party API | 0.867 | 1.000 | −0.133 |
| Sonnet 5 — Claude Platform on AWS | 0.573 | 0.970 | −0.397 |
| Sonnet 5 — Bedrock | 0.537 | 0.975 | −0.438 |

Read the disabled column first: with thinking off, all three doors agree — 0.965 to 1.000 everywhere, no plane story at all. The doors diverge only when adaptive thinking is on, and then they diverge by model in a consistent direction: the first-party API holds 0.983 (Opus) and 0.867 (Sonnet) where the AWS doors fall to 0.537–0.817.

The pre-registered attribution endpoint makes the comparison formal. Opus 5's difference-of-differences: first-party versus Bedrock +23.2 points (CI95 +17.1 to +29.3); first-party versus Claude Platform on AWS +16.7 (+11.3 to +22.0); Bedrock versus Claude Platform on AWS −6.5 (−13.9 to +0.9, spanning zero). Sonnet 5: +30.5 (+23.3 to +37.7), +26.3 (+19.1 to +33.5), and −4.2 (−12.7 to +4.4, spanning zero). Differences-of-differences are computed from unrounded per-cell rates, so they can differ from the rounded table's implied values by a tenth of a point.

Paper one reported, with appropriate hedging, that a thinking mode "changed the sampling landscape enough to move one marginal formatting token from a settled preference to a coin flip." This study's addition is *where* that landscape change happens. Same weights, same request, same thinking mode: whether the coin gets flipped depends on which infrastructure serves the call.

---

## 5. Two Doors, One Behavior

> **Answer:** Bedrock and Claude Platform on AWS behave identically here, despite different operators. AWS runs Bedrock's inference; Anthropic runs Claude Platform on AWS — yet the thinking-effect difference-of-differences between them spans zero for both models, and on the grid-pooled per-plane difference the registered stratified two-one-sided test (TOST) certifies equivalence at the 2-point margin (Opus 5 p=0.046, Sonnet 5 p=0.007, pooled p<0.0001). The first-party API, by contrast, is confirmably *more* reproducible than both AWS doors: pooled across the full grid it runs +2.8 to +2.9 points, with 90% intervals sitting entirely above the 2-point margin. The variable is the front door, not the operator behind it.

This is the study's most useful surprise, and the equivalence direction is as informative as the difference direction. Going in, the natural hypothesis was operator-shaped: Anthropic-operated planes (first-party and Claude Platform on AWS) on one side, AWS-operated Bedrock on the other. The data reject that split decisively. Claude Platform on AWS behaves like Bedrock, down to certified equivalence, while differing sharply from the first-party API that shares its operator.

What Bedrock and Claude Platform on AWS share is the AWS ingress and serving path — `*.api.aws` endpoints, SigV4 authentication, AWS's network and whatever request handling sits behind it. What the first-party API does not share is exactly that. This study measures behavior from outside and cannot see mechanism: batch composition, hardware pools, kernel selection, and request routing are all invisible from the caller's side, and we decline to speculate past the data. "The AWS-fronted serving path" is a location, not an explanation. But it is a *measured* location, and it narrows the search space for anyone with inside visibility to a fraction of what it was.

The stratified estimator earned its registration here. Its 90% intervals on the pooled plane differences are roughly ±0.6 points wide; the first study's pooled-variance test, applied to the same data as a sensitivity, produces intervals too wide to certify anything at 2 points — the same conservatism that made the first study's routing question inconclusive. Registered correctly, the same margin resolves cleanly in both directions: equivalence certified where the data support it (the two AWS doors), difference confirmed where they demand it (first-party versus both).

One asterisk belongs to the anchor model. Haiku 4.5 is plane-idiosyncratic: it measured *most* reproducible on Bedrock — about 4.5 to 5 points above both other planes, differences that failed the equivalence test rather than going untested. The budget tier does not follow the frontier tier's plane pattern, in either direction.

---

## 6. The Fence Is the Coin

> **Answer:** Parsing every structured-JSON response (3,587 across the confirmatory grid and streamed arm; zero parse failures): every byte variant from Opus 5 and Sonnet 5, on every plane, parsed to the identical object — frontier-tier semantic reproducibility was again 100%. The byte-level plane effect is concentrated in one formatting decision: Sonnet 5 with thinking on wrapped its JSON in a markdown fence — which the prompt explicitly forbids — in **~85% of calls through the AWS doors versus ~9–17% first-party**. Haiku 4.5's field-value flips, by contrast, appeared on all three planes at 3–10%: formatting risk is plane-dependent; content risk is model-intrinsic.

The first study established that frontier-tier byte divergence was cosmetic — serialization, not substance. The attribution study sharpens that into a mechanism-shaped observation: the divergence is not just cosmetic, it is *localized*, and its probability is a function of the serving path.

For Opus 5, the byte variants remain the compact-versus-spaced serialization pair from the first study — every response parsed strictly as JSON on every plane; no fences at all. Its plane effect is the whitespace coin's bias moving with the door.

For Sonnet 5, the dominant variant is the forbidden fence, and the fence rate *is* the plane effect in miniature. Thinking on, adaptive, same request: Bedrock fenced 85–87% of responses per window, Claude Platform on AWS 81–86%, the first-party API 9–17%. Thinking off, fence rates fall to 0–5% everywhere and the planes converge — which is the §4 table restated at the level of a single token decision. An instruction to a language model is a probability shift, not a constraint; this dataset now shows the *serving infrastructure* shifting that probability by seventy points on identical requests.

The streamed arm (§7) reproduces the same fence rates within a few points — 86/84/14 across the three doors — so delivery mode is not the mechanism either.

Haiku is the symmetric control. Its substantive flip — the same ambiguous field extracted two defensible ways, the first study's least comfortable finding — appeared in **every one of its six cells on every plane**, at 3.0–10.3% minority share. No door makes the budget tier's judgment-boundary instability go away, and no door created it. Content risk travels with the model; formatting risk lives in the path. A procurement decision that treats "which model tier" and "which serving door" as one question is conflating two independent risk axes this data cleanly separates.

---

## 7. What Did Not Matter — and One Thing That Failed Differently

> **Answer:** Streaming changed nothing: the streamed arm reproduced each plane's pattern within window-level variation, so SSE delivery is ruled out as the mechanism. Input length changed nothing: padding extraction to ~1k, ~10k, and ~50k tokens produced modal share 1.000 in all 18 cells on every plane. But the planes did fail differently as infrastructure: all 85 excluded calls in the study were 529-overload rejections from Claude Platform on AWS, concentrated in four Haiku cells during the US-overnight window — every terminal failure among the study's 13,950 calls sits on that one plane; Bedrock and the first-party API had none.

The two exploratory arms exist because "we didn't test that" is the first objection any measured claim meets.

**Streamed delivery** (1,200 calls: both 5-family models, structured JSON and open generation, adaptive thinking, all planes): open generation stayed at 0.01 everywhere, and structured JSON tracked each plane's non-streamed rate — 0.98 / 0.81 / 0.60 across first-party, Claude Platform on AWS, and Bedrock for Opus 5; 0.86 / 0.55 / 0.49 for Sonnet 5. The streamed arm ran in its own window, so cell-level comparisons carry that confound and are labeled exploratory; the plane *ordering* reproduced exactly.

**Input length** (450 calls: char-exact deterministic padding at three lengths, extraction task): a perfect null. 25 of 25 byte-identical in every cell, both models, all planes, out to ~50,000 padded input tokens. Long context, at least on this task shape, is not a reproducibility hazard on any door.

**The 529s** are an operational finding rather than a validity problem, and they are plane-specific in a way worth recording. During the 07:00–10:00 UTC window, Claude Platform on AWS returned `overloaded_error` terminally on 85 calls — 0.6% of the study, 100% of its failures — concentrated on Haiku cells (worst-hit cell kept 73 of 100 calls; all cells retained analyzable n). The same window on Bedrock and the first-party API completed 100% of calls, as did the peak window everywhere. What a door does under overnight load is a different property than its reproducibility, but a deployment choosing a door cares about both, and this study happened to measure both.

Time-of-day, for completeness, was flat again where it could be compared, replicating the first study's Q4 null.

---

## 8. Choosing a Door

> **Answer:** The serving plane is now a measured determinism variable — for byte-stable structured output with thinking on, the first-party API measured 15–30 points better than either AWS door on the frontier models. But the cheaper lever still dominates: disabling thinking reaches 0.97–1.00 on *every* plane, and canonicalizing at parse boundaries reaches 100% semantic identity on the frontier tier regardless of door. Choose the door with data; engineer as if the door might change.

Concretely, in the order a system designer would apply them:

- **The first study's doctrine survives attribution unchanged.** Disable thinking for structured outputs where the effort budget permits, or canonicalize serialization at every boundary; never parse model text bare (the fence variants are now measured at up to 87% on some paths); validate semantics, log bytes and hashes and returned model IDs for the audit trail. Nothing plane-specific weakens any of that, and the mitigations are cheaper than a migration.
- **If byte-stability under thinking matters, the door is now a design input.** A pipeline that must run adaptive thinking *and* compare raw outputs — caches keyed on response bytes, hash-gated dedupe, diff-based change detection — measured materially more stable through the first-party door in this study's window. That is a measured fact with a dated scope, not a permanent ranking; the reproduction rate for re-measuring it is one afternoon and about $120.
- **Do not read API parity as behavioral identity.** Claude Platform on AWS serves the same Messages API as the first-party platform, same-day features, same request shapes — and behaved, on this study's endpoint, like Bedrock instead. Surface compatibility and serving behavior are separate claims; only one of them is in the documentation.
- **Tier selection remains a substance decision on every door.** Haiku's 3–10% field-value flip rate moved with no plane. If an extracted value feeds something that must be defended later, the budget tier carries that risk through whichever door you choose — and the flips still concentrate exactly where the judgment is closest.
- **Capacity and reproducibility are different SLOs.** The door that measured most Bedrock-like in behavior was also the only one that shed load overnight. Measure both before committing traffic that runs at 3 a.m.

For a regulated deployment, the compliance boundary differs per door (Bedrock's HIPAA eligibility and model exclusions are AWS's list; Anthropic's platforms carry their own terms) — this study speaks only to the operational layer. But "which infrastructure serves our inference, and what did it measure for reproducibility" is now a question with a public, checkable answer format, and an auditor who asks it deserves better than a parity page.

---

## 9. Limitations

> **Answer:** This study attributes the effect to a location it cannot see inside: "the AWS-fronted serving path" is measured from the caller's side of the wall, with batching, hardware, and routing all invisible. It spans one account, one region, one workspace, a single 22-hour confirmatory span, one fixture per task rung. The version-drift control remains blind on the undated frontier models — now verified to be equally blind on all three doors. Sonnet 5's residual first-party effect means "serving path" is most of the story, not all of it. Each limit is disclosed because a careful reader would find it anyway.

1. **Attribution is to a path, not a mechanism.** The study design can say *where* the effect lives (it follows the AWS front door and not the operator) but not *why* — batch composition, kernel selection, quantization, hardware pools, and admission control are all unobservable from outside. Anyone with inside visibility could localize further; this data tells them where to look.
2. **One of everything else.** One AWS account, one region (us-east-1), one Claude Platform workspace, one API organization, a single confirmatory span per window. Whether these numbers hold across regions, accounts, or months has to be re-measured — and the harness prices that re-measurement at about $120.
3. **The intrinsic component is model-dependent.** Opus 5's first-party null makes the pure-serving-path story clean; Sonnet 5's −13.3-point first-party effect means the model contributes too. "The door explains most of it" is the precise claim; "all of it" is not.
4. **Undated IDs, all doors.** `claude-opus-5` and `claude-sonnet-5` return undated identifiers on every plane, so a silent weight roll during the study would be invisible everywhere except the dated Haiku anchor, which held constant. Compressed windows bound the exposure; they do not eliminate it.
5. **The streamed arm is window-confounded** (its own control window versus the grid's two), so its per-cell numbers are exploratory; only the reproduced plane ordering is leaned on.
6. **Byte-identity remains deliberately conservative** as the primary endpoint, and §6 again reports the semantic reading alongside it. The fence-rate observation inherits the fixture caveat: one structured-JSON schema, one prompt, however production-shaped.

---

## 10. Reproducing This

> **Answer:** Both studies live in one public repository: pre-registrations with their freeze tags (`prereg-v1`, `prereg-v2`), the stdlib-only harness and analysis code including the stratified estimator and the semantic addendum, per-call raw records with request and wire hashes, and every committed report. The confirmatory grid here cost $115.20 at list prices; the pilot cost $5.98. Checking our arithmetic requires no API access at all — the analysis re-runs on the committed records in seconds.

```bash
git clone https://github.com/iSimplifyMe/determinism-harness
git log -1 prereg-v2                                          # the freeze, before the data
python3 -m unittest discover -s tests -t .                    # stdlib-only suite
python3 -m analysis.analyze runs/low-study2-full-*.jsonl \
    runs/peak-study2-full-*.jsonl runs/control-study2-*.jsonl \
    --out reports --delta 0.02                                 # every table in this paper
python3 -m analysis.semantic_sj runs/low-study2-full-*.jsonl \
    runs/peak-study2-full-*.jsonl \
    runs/control-study2-q3-streaming-*.jsonl --out reports     # section 6
```

Re-running the calls themselves requires credentials for the three planes — an AWS account with Bedrock access, a Claude Platform on AWS workspace, and a first-party API key — and the runner's per-plane smoke tests will verify each door's expected behavior, including the expected rejections, before any money is spent.

The claim format is the same as the predecessor's, because the format is the point: not that these numbers are permanent — they describe three doors during one 22-hour span — but that they are checkable, that the hypotheses predate the data by verifiable ordering, and that the one methodological debt the first study recorded was paid here in the registered primary, not in a footnote.

---

## References and further reading

- [`iSimplifyMe/determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — both pre-registrations (tags `prereg-v1`, `prereg-v2`), harness, analysis, semantic addendum, and complete raw data; MIT
- [*The Same Request Twice*](https://isimplifyme.com/whitepapers/the-same-request-twice) — the predecessor study: the task ladder, the thinking-mode headline, and the estimator lesson this study inherits
- [*The Blind Panel*](https://isimplifyme.com/whitepapers/the-blind-panel) — the statistical machinery and the freeze-before-data argument
- [*The Finite Chain*](https://isimplifyme.com/whitepapers/the-finite-chain) — verification as deterministic code rather than a second model
- He, H. et al., *Defeating Nondeterminism in LLM Inference* (Thinking Machines Lab, 2025) — batch-invariance analysis of why identical requests diverge under load
- [Claude Platform on AWS documentation](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws) — the plane's own account of its architecture and parity scope
- [AWS HIPAA Eligible Services Reference](https://aws.amazon.com/compliance/hipaa-eligible-services-reference/) — the per-door compliance boundary this study does not address
