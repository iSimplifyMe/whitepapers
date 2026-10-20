> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-same-model-five-doors](https://isimplifyme.com/whitepapers/the-same-model-five-doors)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-08-23 · License: [CC BY 4.0](LICENSE)

# The Same Model, Five Doors: OpenAI's Flagship Through Two Vendors' Serving Stacks

A pre-registered replication across a vendor boundary — 5,850 confirmatory calls sending the same OpenAI flagship through five doors in one measured span: OpenAI's first-party API, Amazon's OpenAI-compatible door, Amazon's translated door under two routing modes, and the agent harness subscribers actually get. Every door emits the same two byte-variants of the same JSON. Which variant you mostly receive depends on the door — and on a reasoning-effort knob whose effect turns out to be door-dependent too.

---

## Abstract

> **Answer:** The serving door changes the model's output distribution, and this time the doors span two vendors. On the frozen structured-JSON task at pinned effort `none`, the three raw HTTP doors — OpenAI first-party, Bedrock's OpenAI-compatible endpoint, and Bedrock's translated Converse endpoint — are statistically indistinguishable (modal-variant share 0.609–0.630), while OpenAI's own codex harness sits at 0.880, confirming the registered direction at p<0.0001. The sharper finding was not registered as a direction at all: pinning reasoning effort `high` moves the two Bedrock doors from a near-coin to a three- and four-to-one preference — and flips *which* byte-variant they prefer — while OpenAI's first-party door does not move and the harness door gets less stable. No pair of HTTP doors passes the registered 2-point equivalence test.

The predecessor study, [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors), attributed a reproducibility cost to the AWS-fronted serving path — but everything it measured ran on one vendor's weights, reached through infrastructure that vendor or its partner operated end to end. Since July 2026, GPT-5.6 has been a first-class Bedrock citizen, which creates something the series has not had before: a natural experiment where *neither company controls the whole path*. Same OpenAI weights, served by OpenAI on one door and by Amazon on three others, with OpenAI's own subscription harness as a fifth arm.

The discipline is unchanged: pre-registration frozen and pushed (tag `prereg-v4`) about five hours before the first confirmatory call, byte-canonical requests with the SHA-256 of the actual outbound bytes recorded per call, committed pre-data estimators, raw records public. What is new is the boundary being crossed — and a deviations ledger, because this registration shipped with one internal inconsistency that we disclose and resolve in the open rather than edit out of the frozen file.

The reference implementation, all four pre-registrations, and complete raw data are at [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — stdlib-only measurement, MIT; `prereg-v4` froze this study.

---

## 1. A Question That Crosses a Vendor Boundary

> **Answer:** Studies one and two asked whether the serving stack changes model behavior *within* one vendor's ecosystem. This study asks it across a vendor boundary: `gpt-5.6-sol` — OpenAI's 1M-context flagship — is reachable through OpenAI's first-party Responses API, through Bedrock's in-region OpenAI-compatible endpoint, through Bedrock's translated Converse endpoint under both geographic and global routing, and through the codex CLI billed to a ChatGPT subscription. Five doors, two vendors, one model. If door effects are real, they should survive a change of weights, vendor, and protocol; if they are a Claude-serving quirk, this is where that hypothesis dies.

*Three Doors* ended on a measured location: the thinking-mode reproducibility cost followed the AWS front door, not the operator behind it. That result carried an unavoidable scope limit — every door served Claude weights. A skeptic could still read [the whole series](https://isimplifyme.com/whitepapers) as one vendor's serving story. The clean test is to hold the experiment design fixed and swap everything else: different weights, different protocol family, different second vendor relationship. Amazon now serves OpenAI's flagship in exactly the configuration that test needs — up to and including OpenAI's most restricted model, GPT-5.6-Cyber, which sits on Bedrock behind OpenAI's "Trusted Access for Cyber" gate. That gate is why Cyber is a hook and not a subject here: we could not test it, so nothing in this paper depends on it. Vanilla Sol has no gate, and Sol is the subject.

The fifth door needs its own justification, because it is not an API. `codex exec` is OpenAI's agent harness, billed to a consumer ChatGPT subscription — the way a growing share of real usage actually reaches the model. [*The Ceiling Is Not the Model*](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model) argued that harness configuration can fabricate results before the model gets a vote. The codex door instantiates that thesis as a measured arm: it injects roughly 13,900 tokens of standing scaffold per call, defaults reasoning effort to `none` silently, and offers no wire-level control. It is registered as a labeled arm — "what subscribers actually get" — and excluded from every equivalence claim by construction.

One vocabulary note: throughout, *modal-variant share* means the fraction of a cell's valid calls whose response bytes match that cell's most frequent response — the series' primary endpoint.

---

## 2. Frozen First — and a Deviations Ledger

> **Answer:** The pre-registration was frozen, tagged `prereg-v4`, and pushed to the public repository at 18:07 UTC on August 17; the first confirmatory call went out at 23:17 UTC the same day. Every registered validity gate held: the positive control fired on all five doors (open generation at effort `none` produced 100% distinct outputs — 750 calls, 750 unique responses across the doors), every cell was wire-byte-uniform, and all 5,850 scheduled calls produced a clean record — zero exclusions, with exactly three calls anywhere needing a second attempt. The ledger carries one pre-data deviation, disclosed here rather than edited into the frozen file.

The deviation first, because the series' credibility rests on how these are handled. The frozen registration's prose for Q2 names "four HTTP doors" for the equivalence test, but the registered grid gives the global-routing door Q4 cells only — no full-grid cells exist for it. The committed pre-data estimator likewise scopes Q2 to the three full-grid doors. Prose and grid disagreed inside one frozen document. The grid and the committed code are the operative registration; the four-door sentence was unexecutable as written. Q2 therefore runs on three doors, the inconsistency was found and recorded before any HTTP-door confirmatory call, and the frozen file stays frozen. A second entry in the ledger is a note rather than a deviation: both HTTP windows ran on a Sunday — the series' first weekend windows. The registration pins UTC bands, not weekdays; running both windows on the same day holds day-type constant within the study, but a Sunday peak should not be equated with a weekday peak when comparing across studies.

Discovery also forced one correction that shaped the registration itself. Early single probes suggested the first-party door's *default* reasoning effort behaved like `high`. At n=15 per door, that read collapsed: the default-effort reasoning-token distributions coincide across all the API doors (medians 172–191 on the discovery probe prompt — the default's spend is task-dependent, as Q5 later confirmed), with 4–5× run-to-run spread on byte-identical requests — the single samples had been tail draws from an adaptive, run-variable process. The registered consequence: effort is *pinned* (`none` and `high`) in every confirmatory arm, and "default" appears only as a labeled exploratory question. Single-sample reads of adaptive systems mislead; this study nearly registered one.

The remaining gates, as registered and as they held:

- **Positive control, per door.** No door accepts sampling parameters on Sol — `temperature` and `top_p` are rejected on every API door, `seed` is unrecognized, and the harness exposes no parameter surface at all — so study two's sampled-divergence control does not exist here. The registered replacement: open generation at effort `none` must diverge within every door, and a ≥99%-identical open-generation cell would have flagged instrument failure and gated that door's null results. Measured: 100% distinct on every door — codex 100/100, mantle 200/200, first-party 200/200, runtime-us 225/225, runtime-global 25/25.
- **Negative control at the wire.** The harness records the SHA-256 of the bytes actually sent, captured at the HTTP layer (a botocore before-send hook on the Converse door, hook-firing smoke-checked). More than one wire hash inside a cell fails the cell. No cell failed.
- **Receipts on the harness door.** `codex exec --json` prints no effort banner, so each batch ran a registered plain-mode probe whose stderr banner had to state the pinned effort and model before the batch's measured calls ran with identical arguments.
- **Version drift.** No door serves a dated snapshot of Sol — the dated-anchor mitigation the Claude studies leaned on is structurally unavailable on OpenAI weights, on either vendor. Compressed windows, recorded response IDs and served-model fields, and cross-window consistency are the fallback; a silent mid-study weight roll remains a registered limitation. The served-model field held constant on the doors that return one.
- **Completeness.** 5,850 of 5,850 scheduled calls produced valid records. The three retried calls were all on the Converse door in the peak window; the first-party 5xx flakiness discovery had braced for (4 of 15 discovery calls one evening) never appeared — 1,650 of 1,650 first-attempt.

---

## 3. Five Doors, One Request

> **Answer:** The confirmatory grid: three full-grid HTTP doors × four tasks × pinned effort {`none`, `high`} × two UTC windows × 100 repeats = 4,800 calls; the codex arm at 4 tasks × 2 efforts × 100 = 800, batched across subscription rate windows; a 100-call routing arm (Q4) and a 150-call adaptive-default arm (Q5) complete the 5,850. HTTP windows ran 41 and 43 minutes on August 23; the codex arm ran as twenty 40-call batches over four days. Total API spend: about $32 at list prices. The codex arm billed $0 marginal to a ChatGPT Plus subscription — while pushing 11.1 million input tokens through it.

| Door | Path | Protocol | Auth | What you should know |
|---|---|---|---|---|
| OpenAI first-party | `api.openai.com/v1/responses` | Responses | API key | `store:true` is the *default* — responses persist server-side unless you turn it off; disclosed here, not neutralized |
| Bedrock mantle | `bedrock-mantle.us-east-1.api.aws/openai/v1/responses` | Responses (OpenAI-compatible) | Bearer (IAM service-specific credential) | request body byte-identical to first-party except the model alias — a one-field difference, unit-tested |
| Bedrock runtime-us | `us.openai.gpt-5.6-sol` | Converse (translated) | SigV4 | effort pin rides `additionalModelRequestFields`; reasoning tokens invisible — billed inside aggregate `outputTokens` |
| Bedrock runtime-global | `global.openai.gpt-5.6-sol` | Converse (translated) | SigV4 | routing sub-axis, Q4 cells only |
| codex-sub | `codex exec` 0.147.0, ChatGPT Plus | agent harness | ChatGPT OAuth | ~13.9K-token standing scaffold per call; effort defaults to `none` silently; no wire control; labeled arm |

"Same request" has to be earned across doors, and this grid earns it two ways. Between the two Responses doors it is literal: the first-party and mantle request bodies differ in exactly one field — the model alias, because mantle rejects the bare model name — and the aggregate billing meters agree: both doors metered **211,400 input tokens exactly** across their 1,650 confirmatory calls each. Between protocol families it is semantic, as in the Claude studies: the Converse door carries the same task text with the effort pin expressed in its own parameter shape, and the within-cell byte-uniformity of what was actually sent is verified per call on every door.

The task ladder is the frozen four-rung set from [study one](https://isimplifyme.com/whitepapers/the-same-request-twice) — extraction, classification, structured JSON, open generation — with one honest asterisk: classification entered this study unpiloted on OpenAI weights, and it turned out degenerate, reproducing byte-identically in all 1,400 of its calls on every door. It joined extraction at the ladder's floor rather than adding a middle rung — an uninformative cell, disclosed as such. The floor itself is a finding: extraction and classification reproduce byte-identically at n=100–200 per cell through every door, both vendors, both protocols, harness included. Whatever the doors do to this model, they do not touch the settled rungs. The action, for the third study running, is structured JSON.

Effort semantics came from the API itself: the accepted set is {`none`, `low`, `medium`, `high`, `xhigh`, `max`} — `minimal` is gone in 5.6 — and the flat `reasoning_effort` parameter is rejected on every Responses-derived door; the nested form is required. Bedrock strips OpenAI's helpful migration guidance from model-layer errors down to a bare "Unknown parameter," which matters to anyone debugging a cross-door port and is the kind of difference no parity page mentions.

---

## 4. Two Bytes, Five Doors

> **Answer:** Across all 1,525 structured-JSON confirmatory calls — five doors, both pinned efforts, plus the default arm — the model emitted exactly two byte-variants of the same JSON object, and never a third: `"unit_price_usd":349.5` versus `349.50`, one character, semantically equal. At effort `none`, the registered primary cell, the three HTTP doors are statistically indistinguishable — modal-variant share 0.630 (first-party, CI95 0.561–0.694), 0.620 (mantle, 0.551–0.684), 0.609 (runtime-us, 0.544–0.670) — while the codex harness sits at 0.880 (0.802–0.930). The registered contrast, first-party versus codex, confirms at −0.250 (CI95 −0.342 to −0.158, p<0.0001). And no HTTP-door pair passes the registered 2-point equivalence test.

Discovery had found this knife edge at n=20 per door with a suggestive p=0.019 on the registered pair. At n=100–225 per door, the pattern is no longer suggestive. The two raw-API doors and the translated door cluster so tightly at effort `none` that their pairwise differences (1.0–2.1 points) are not distinguishable from noise (p=0.69–0.92) — same near-coin, three infrastructures, two vendors. The harness door is 25–27 points away from all of them at p<0.0001 on every contrast (the registered pair exactly: Fisher p=3.3×10⁻⁶). One registered direction, confirmed; the other codex gaps reported as measured.

So the doors agree at `none` — but the registered equivalence test says certifying that agreement is a different matter, and the reason turns out to be the next section's finding. The test (TOST at a 2-point margin — certification requires the 90% interval to sit inside ±2 points — stratified across all sixteen task-effort-window strata): the two Bedrock doors miss certification narrowly — difference 0.9 points, CI90 −0.4 to +2.3, failing only because the upper bound cannot be excluded at this sample size; an underpowered near-miss, not evidence of a gap. The vendor-boundary pairs fail for a real reason: first-party versus mantle lands at −2.4 points, *outside the margin entirely* — and that is an average over sixteen strata, twelve of which measured degenerate on every door (byte-identical at the ladder's floor, fully divergent on open generation), leaving them no room to contribute a difference. The gap concentrates in the structured-JSON strata where effort is pinned `high`.

For readers of the earlier studies, note the echo with a twist. In *Three Doors*, the doors agreed with thinking *off* and diverged when it was on. Here the doors agree at effort `none` — and diverge when effort is pinned `high`. The pattern crossed the vendor boundary. What did not cross is the direction, and that is the finding the series did not see coming.

---

## 5. The Effort Knob Turns Differently Behind Each Door

> **Answer:** Pinning reasoning effort `high` instead of `none` — same task, same windows, same doors — *raised* byte-stability on both Bedrock doors (mantle +19.5 points, CI95 +10.9 to +28.1; runtime-us +15.1, +6.4 to +23.8), did nothing measurable on OpenAI's first-party door (−2.0, −11.5 to +7.5), and *lowered* it on the codex harness (−11.0, −21.4 to −0.6). The cross-door difference-of-differences excludes zero for first-party versus each Bedrock door. And behind Amazon's doors the knob does not just tilt the distribution — it flips which variant is modal: at `high`, mantle and runtime-us prefer `349.50` at roughly four- and three-to-one, while first-party and codex stay with `349.5`.

The per-door table, structured JSON, pooled across both windows:

| Modal-variant share | effort `none` | effort `high` | high − none (CI95) | modal variant at `high` |
|---|---|---|---|---|
| OpenAI first-party | 0.630 | 0.610 | −0.020 (−0.115 to +0.075) | `349.5` |
| Bedrock mantle | 0.620 | 0.815 | **+0.195** (+0.109 to +0.281) | `349.50` |
| Bedrock runtime-us | 0.609 | 0.760 | **+0.151** (+0.064 to +0.238) | `349.50` |
| codex-sub | 0.880 | 0.770 | **−0.110** (−0.214 to −0.006) | `349.5` |

[Study two's headline](https://isimplifyme.com/whitepapers/the-same-model-three-doors) was that a reasoning mode's reproducibility *cost* followed the serving door. This study set out to test whether that generalizes across a vendor boundary, and the answer is yes with an inversion: on OpenAI weights the reasoning knob's effect follows the door too — but on these doors it is the *Bedrock* path where effort stabilizes the output, the vendor's own API where it does nothing, and the vendor's own harness where it destabilizes. No direction was registered for Q3, and either outcome was publishable; this is the measured one.

Two disciplined readings of the flip. First, at the level the registration defined — modal share of byte-identical output — the effect is unambiguous and its confidence intervals are wide of zero on three of four doors. Second, the variant-identity observation (which byte-form wins at `high`) is a descriptive tally over the committed records, not a registered endpoint; we report it because it is checkable and because it sharpens the mechanism question, not because the registration blessed it. Both variants parse to the identical object on every door — the byte/semantic gap the series keeps finding is intact: a system validating semantics sees five agreeing doors; a system comparing bytes sees a knife edge that a config knob flips differently depending on whose infrastructure serves the call.

What this study cannot see is *why*. Batching, kernel selection, sampling-server implementation, and whatever each vendor's serving stack does with the effort parameter are all invisible from the caller's side. "The Bedrock doors respond to effort and the first-party door does not" is a measured location, not an explanation — but it is a narrow one, and the two Bedrock doors moving *together* (their difference-of-differences is +4.4 points with a CI spanning zero — consistent with joint movement, though a null is not proof of it) while both differ from the first-party door by 17–22 points localizes it further: whatever does this sits behind Amazon's front door, on both of Amazon's protocols, and not behind OpenAI's.

---

## 6. The Harness Door

> **Answer:** The codex arm measured what a scripting subscriber actually gets, and it is a different product from the API on every axis this study can measure: ~13,900 input tokens of standing scaffold per call at scale (11.1 million across the 800-call arm), versus ~128 input tokens per call for the identical schedule on the HTTP doors — roughly 108× — with reasoning effort silently defaulted to `none`, the most byte-stable structured output of any door at that default (0.880), and the only door where pinning effort `high` made output *less* stable. Priced at the API's $5.50-per-million input rate, the scaffold-dominated input alone would have cost about $61 — nearly double this study's entire three-door HTTP bill.

[*The Ceiling Is Not the Model*](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model) argued that the harness is part of the system under measurement, whether the measurer admits it or not. The codex door makes that concrete three ways.

- **The silent default is a behavioral choice.** `codex exec` runs Sol at reasoning effort `none` unless overridden with a config flag most subscribers will never find. §5's table says what that choice does: the harness's stability profile sits on the opposite side of the effort knob from the Bedrock doors. A team that scripts through codex and later "upgrades" to the API inherits a different output distribution twice over — once for the scaffold, once for the effort semantics.
- **The overhead is charged against the subscriber's own limits.** The ~13.9K-token scaffold rides every call, spending the subscription's rate windows on tokens the subscriber never wrote. The 800-call arm had to be batched across four days to respect those windows — itself a measured property of the door.
- **Folklore lost to measurement again.** A widely-cited community issue reported the exec path clamping context around 258K tokens. Direct measurement pushed 621,804 input tokens through one exec call and retrieved a needle; the real bound is a 1,048,576-*character* protocol cap in the JSON-RPC layer.

The harness door's numbers are labeled, never pooled with the API doors, and excluded from equivalence claims by registration. It is not a worse API; it is a fifth product with its own defaults, its own overhead, and — this study's addition — its own response to the one control knob the model exposes.

---

## 7. What Did Not Cross, and What Did Not Matter

> **Answer:** Study two's most vivid finding — a forbidden markdown fence fired in ~85% of AWS-door calls versus 9–17% first-party on Claude weights — did not reproduce on OpenAI weights: fence rate 0 for 20 in discovery on every door, so in everything measured here that formatting failure was model-family-specific, not door-intrinsic. This section's other two results, each taken up below: geographic-versus-global routing on Bedrock was an exploratory null, and the adaptive default's reasoning spend is run-variable on identical requests — visible on two doors, invisible on the third.

The fence result kills the tempting generalization that "AWS doors cause formatting failures" — study two located a formatting risk in a path; this study shows the *risk* was the model family's and only its *probability* belonged to the path. Cross-family, the durable claim is narrower and more useful: doors modulate whatever byte-level instability the model already has; they do not conjure the same failure everywhere.

The routing null is the OpenAI-weights analog of study one's finding on Claude: at n=25 per cell, `us.` versus `global.` inference profiles on the Converse door showed no measurable difference (stratified +2.7 points, SE 5.6, per-task CIs straddling zero) — a bound, not a proof of equality, and registered as exploratory.

The default-burn arm (Q5) closes the loop on the §2 correction story. Pinned effort was the right registration call: left to its default, the model's reasoning spend on byte-identical open-generation requests ranged 43–287 tokens on the first-party door (median 73) and 44–149 on mantle (median 75) — nondeterministic spend on deterministic input. On structured JSON the adaptive default spent *zero* reasoning tokens in all 50 calls on both Responses doors — the default works out to "don't think about JSON," which makes §5's finding sharper: at effort `none`, where there is no reasoning spend to vary, the structured-JSON knife edge lives entirely in serving behavior. And on the Converse door none of this is visible at all: `outputTokens` arrives as one aggregate number. An auditor reconstructing spend or behavior from Bedrock's translated door is working with strictly less information than the same auditor on the OpenAI-compatible door next to it — same vendor, same model, different instrument panel.

---

## 8. Choosing a Door

> **Answer:** "GPT-5.6 on Bedrock" and "GPT-5.6 from OpenAI" are the same weights behind measurably different serving behavior — indistinguishable on the primary cell with effort off, 15–21 points apart with effort pinned high, differently instrumented, identically metered — the only cost difference *measured here* is overhead — and differently controlled. The knob you validate on one door does not mean the same thing behind another. Procurement should treat door choice as a config decision with behavioral consequences, and treat the subscription harness as a fifth product rather than a cheaper API.

Concretely, in the order a system designer would apply them:

- **The series doctrine survives its second vendor unchanged.** Validate semantics, never raw bytes, at every boundary — both byte-variants here parse identically, and a semantic validator sees five agreeing doors. Canonicalize serialization where bytes must match. Log request bytes, wire hashes, response IDs, and served-model fields; on OpenAI weights there is no dated snapshot on any door, so your logs are the only drift instrument you will ever have.
- **If you pin effort, re-validate per door.** The same `high` pin that stabilized output behind Bedrock's doors did nothing on the first-party API and destabilized the harness. An effort setting tuned on one door silently means something else behind the next one. This is the study's most transferable operational fact.
- **Byte-comparing pipelines should know the knife edge is real on OpenAI weights too.** Hash-keyed caches, diff-based change detection, and dedupe layers sit on a two-variant coin whose bias moves with door and effort. At effort `none`, expect roughly 60/40 behavior on every HTTP door; do not tune thresholds on the harness's 88/12 and expect them to transfer. And if a pipeline must both pin `high` and compare bytes, the Bedrock doors measured most stable in this span, while the first-party door was the one whose behavior did not move with the knob — dated observations from one span, not rankings.
- **The compatible door is compatible at the wire, not in the fine print.** Mantle accepts byte-identical bodies (one field aside) and meters identical input tokens — and still strips the guidance text from upstream error messages, rejects the bare model name, and requires its own credential type. Budget for the port; do not assume it.
- **Know what your door lets you see.** First-party persists responses by default (`store:true`) — a data-governance fact, not a behavior fact. Converse hides reasoning spend. The harness hides nearly everything and adds ~13.9K tokens you did not write to every call. What you can audit is part of what you are buying.
- **Subscription scripting is a real door with real limits.** It is the cheapest per marginal call and the most expensive per token of visibility; its rate windows made an 800-call study take four days. If your team scripts through codex today, you are already running these defaults in production — you just have not measured them yet.

For regulated deployments, the compliance boundary differs per door and per vendor, and nothing here substitutes for that analysis. What this study adds is that the *operational* boundary differs too — and §10 prices checking it at pocket change.

---

## 9. Limitations

> **Answer:** This study measures five doors from outside during one span: one AWS account, one first-party organization, one region pinned per door, one ChatGPT Plus workspace, one task fixture per rung — and "the same model" is itself the vendors' representation, since weight identity across vendors is not verifiable from the caller's side. The HTTP windows ran on a single Sunday — the series' first weekend windows, held to the registered UTC bands but not comparable to a weekday peak across studies. No door serves a dated model snapshot, so a silent mid-study weight roll would be visible only through the indirect checks that held. The equivalence near-miss between the two Bedrock doors is underpowered, not adjudicated. Classification joined the ladder floor unpiloted and contributed nothing. And attribution stops at the door: nothing here says *why* Amazon's path responds to the effort knob and OpenAI's does not.

1. **A location, not a mechanism.** "Behind the AWS front door, on both protocols" is where the effort response lives; batching, kernels, quantization, and admission control are invisible from the caller's side. Anyone with inside visibility could take the search from there.
2. **One of everything.** One account per vendor, one region (us-east-1; us-east-2 unexercised), one subscription, one confirmatory span per arm. The harness prices a re-measurement at about $32 plus an afternoon.
3. **Weekend windows.** Both HTTP windows ran Sunday, August 23 — within-study day-type is constant, but cross-study comparisons to weekday peaks carry that asterisk, disclosed in the ledger.
4. **Version-drift blindness is total on these weights.** The Claude studies had one dated anchor model; OpenAI's doors offer none. Response IDs, served-model fields, and window compression are mitigation, not proof.
5. **"Same weights" is a representation, not a measurement.** Nothing observable from outside distinguishes identical weights behind different serving behavior from near-identical weights — quantization and serving-stack transforms are named among the invisibles above precisely because they are invisible. The title's premise rests on both vendors representing this as GPT-5.6-Sol; the study measures doors and takes the vendors at their word about what stands behind them.
6. **The TOST near-miss is a power statement.** Mantle-versus-runtime at 0.9 points with an upper bound of 2.3 fails certification at 2.0; doubling n would likely resolve it either way. We report it as unresolved rather than rounding it to "equivalent."
7. **The fixture caveat, as always.** One structured-JSON schema, one prompt per rung, however production-shaped. The two-variant knife edge is this fixture's; the *pattern* — doors and knobs moving byte-bias while semantics hold — is what the series keeps replicating across fixtures, models, and now vendors.
8. **The harness arm is one CLI version** (codex 0.147.0) under one subscription tier's rate windows; harness defaults are software and can change faster than serving stacks.

---

## 10. Reproducing This

> **Answer:** Everything is in one public repository: the frozen registration (tag `prereg-v4`, pushed five hours before the first confirmatory call), the runner, the committed pre-data estimators, per-call raw records with wire hashes for all 5,850 confirmatory calls plus the disclosed discovery corpus, and the analysis artifact this paper's every number comes from. Total spend: about $32 in API fees; the codex arm rode a ChatGPT Plus subscription. Checking our arithmetic requires no credentials and no spend — the analysis re-runs from committed records in seconds.

```bash
git clone https://github.com/iSimplifyMe/determinism-harness
git log -1 prereg-v4                            # the freeze, before the data
python3 -m unittest discover -s tests -t .      # stdlib-only suite (349 tests)
python3 -m analysis.analyze_study4 \
    runs/low-study4-full-*.jsonl \
    runs/peak-study4-full-*.jsonl \
    runs/control-study4-codex-b*.jsonl \
    runs/control-study4-q4q5-*.jsonl            # every registered number in this paper
```

Re-running the calls themselves needs a first-party API key, a Bedrock API key for the mantle door, SigV4 credentials for the Converse doors, and a ChatGPT subscription with the codex CLI for the harness arm; the runner's per-door smoke checks verify each door's expected behavior — including the expected parameter rejections — before any money is spent.

The claim format is the series': not that these numbers are permanent — they describe five doors during one measured span — but that they are checkable, that the hypotheses [verifiably predate the data](https://isimplifyme.com/whitepapers/the-blind-panel), that the one registered direction confirmed and the surprises are labeled as surprises, and that the deviation the registration shipped with is in the paper instead of quietly edited out of it.

---

## References and further reading

- [`iSimplifyMe/determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — all pre-registrations (tags `prereg-v1` through `prereg-v4`), harness, analysis, and complete raw data; MIT
- [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) — the predecessor: serving-path attribution on Claude weights, and the AWS-door equivalence this study's design inherits
- [*The Same Request Twice*](https://isimplifyme.com/whitepapers/the-same-request-twice) — the original task ladder and the byte-versus-semantic endpoint discipline
- [*The Ceiling Is Not the Model*](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model) — the harness thesis the codex door instantiates
- [*The Blind Panel*](https://isimplifyme.com/whitepapers/the-blind-panel) — the freeze-before-data argument and the statistical machinery
- He, H. et al., *Defeating Nondeterminism in LLM Inference* (Thinking Machines Lab, 2025) — batch-invariance analysis of why identical requests diverge under load
- [OpenAI models on Amazon Bedrock](https://aws.amazon.com/bedrock/openai/) — the vendor's own account of the doors this study measured from outside
