> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-same-object-different-bytes](https://isimplifyme.com/whitepapers/the-same-object-different-bytes)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-07-30 · License: [CC BY 4.0](LICENSE)

# The Same Object, Different Bytes: What a Deterministic Validator Catches — and What It Cannot See

Three pre-registered studies measured where LLM inference reproduces and where it does not. This companion asks the engineering question underneath all of them: if every one of those 33,444 confirmatory outputs passes through a deterministic output validator, how much of the measured instability survives? The answer splits cleanly — and the split is the risk map.

---

## Abstract

> **Answer:** A deterministic validator — parse, schema-check, canonicalize, fail closed — was applied to all 33,444 valid confirmatory outputs from the three reproducibility studies, with zero new model calls. On structured JSON under deterministic configurations, it recovers almost everything: byte-level reproducibility of 0.765 (study 1), 0.816 (study 2), and 0.991 (study 3) becomes post-validator semantic reproducibility of 0.986, 0.990, and 0.999. Every frontier-tier structured-JSON cell in all three studies — including every streamed arm — closes to exactly 1.0. What survives is the residue no output gate can see: a budget-tier model returning a different field value in 4–9.6% of calls per serving plane, a frontier model flipping a classification label on 5 of 600 thinking-enabled calls, and one greedy flip in 1,400 local structured-JSON calls — the same field, forked the same two ways, as the budget-tier model on the API. Outright rejection almost never fired: 5 outputs in 33,444, zero JSON parse failures anywhere. Under temperature-0.7 sampling in the local study the picture inverts: 20% of structured-JSON calls and 7% of classification calls return a semantically different answer, and no validator recovers it.

This is a reanalysis, not a new experiment. [*The Same Request Twice*](https://isimplifyme.com/whitepapers/the-same-request-twice) measured reproducibility through one serving door with no knobs; [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) attributed the instability to the serving path; [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) measured the ceiling under full control. All three kept concluding, in their mitigation sections, that a parse-then-canonicalize gate recovers most of what the substrate takes away. This paper stops asserting that and measures it — with a validator whose exact rules are stated below and whose code is public alongside the data it ran on.

The reanalysis is descriptive and was not pre-registered; it invents no estimators and tests no hypotheses. It counts, over committed public records, what a stated deterministic gate would have done.

---

## 1. Two Kinds of Reproducibility

> **Answer:** Byte-level reproducibility asks whether repeated identical requests return identical bytes; semantic reproducibility asks whether they return the same object after a deterministic normalization. The gap between the two is exactly the portion of LLM output instability that ordinary engineering absorbs for free — and the portion of instability *outside* that gap is the residual risk that no output-side control can address.

The three studies measured bytes, deliberately: byte identity is the only definition of "same answer" that requires no interpretation layer and no judgment call. But nearly every production consumer of model output already runs some interpretation layer — a JSON parse, a schema check, an enum lookup. A serialization flip that a parser erases is a different category of production risk than a field value that changes between calls. The studies' byte-level numbers bound the problem; they do not, by themselves, say how much of the problem reaches an application.

The dataset for answering that already existed. Across the three confirmatory runs there are 33,563 records — 13,100 through AWS Bedrock, 13,950 across three serving planes, 6,513 against open weights on owned hardware — of which 33,444 are valid completed calls (13 recorded warm-ups and 106 transport-level errors are excluded, the same exclusions the studies themselves registered). Every record carries the full response text. Re-running a deterministic function over committed public data costs nothing and changes nothing upstream: the studies' registered answers stand untouched.

---

## 2. The Validator, Stated Exactly

> **Answer:** For structured JSON: parse the raw text as JSON — or, failing that, parse the interior of a markdown code fence, the one deviation the studies showed serving paths inject — then require exactly the instructed six keys with the instructed types, and canonicalize (sorted keys, fixed separators). For classification: accept only an exact member of the four instructed labels after whitespace strip. For extraction: accept only a string matching the instructed output shape (`PO-#####-AA`). Anything else is rejected, fail-closed. Open-ended generation gets no validator at all — no deterministic acceptance rule exists for free prose, which is itself a finding-shaped fact.

Three properties of this gate matter for reading the numbers.

It is deterministic code — no model judges model output anywhere in the loop, consistent with every measurement decision in the series. The module and its tests are committed in the public harness repository next to the records it consumes.

It is fail-closed. An output that does not parse, does not match the schema, or does not belong to the label set is *rejected*, the way a production gate protecting a downstream system would reject it — counted here as caught, with the availability cost reported honestly.

And it is format-only where format is all a validator can know. The extraction rule accepts any string shaped like a purchase-order number, including a wrong one. A validator that has the correct answer is an answer key, not a gate. This structural blindness turns out not to be exercised anywhere in the 33,444 records — extraction was byte-perfect in every cell of every study, up to 50,000-token inputs and even under sampling — but the blindness is real, and section 9 returns to it.

Every output lands in exactly one of four bins: **consistent** (accepted, same canonical object as the cell's semantic majority), **neutralized** (accepted with different bytes but the same object — the validator's recovery), **rejected** (caught by the gate), or **invisible divergence** (accepted, well-formed, and a *different object* — the class no output gate can see).

---

## 3. What the Gate Recovers

> **Answer:** On deterministic configurations, pooled across cells: study 1's structured JSON goes from 0.765 byte-reproducible to 0.986 post-validator; study 2's from 0.816 to 0.990; study 3's local greedy baseline from 0.991 to 0.999. Every structured-JSON cell on the frontier tier — Opus 5 and Sonnet 5, every plane, every thinking arm, streamed and unstreamed, in both API studies — closes to exactly 1.0. The serving-path effect that was the second study's headline is, from an application's point of view, fully recoverable formatting.

| Structured JSON, deterministic configs | n | byte modal share | post-validator | invisible |
|---|---|---|---|---|
| Study 1 — Bedrock, 3 models pooled | 2,995 | 0.765 | 0.986 | 0.014 |
| Study 2 — 3 planes, 3 models pooled | 3,587 | 0.816 | 0.990 | 0.010 |
| Study 3 — local open weights, greedy | 1,400 | 0.991 | 0.999 | 0.001 |

The recoveries concentrate exactly where the studies located the instability. Sonnet 5 with adaptive thinking through Bedrock — 0.560 byte-reproducible, the fence firing on 172 of 200 calls — validates to 1.0. The same model's *streamed* Bedrock arm, the least byte-stable frontier cell in the entire dataset at 0.490, validates to 1.0: a 51-point recovery in a single cell. Opus 5's adaptive serialization coin-flip — compact versus spaced JSON, near-fair through AWS doors — validates to 1.0 everywhere, because both faces of the coin parse to the identical object. The forbidden markdown fence, whose firing rate moves seventy points with the serving door, is eaten whole by one regex.

The third study's contribution is the control-ceiling version of the same statement: the scale-gated reasoning-mode effect — gpt-oss-120b's 89:11 two-variant split at high effort, the study's headline — is *fully cosmetic*. Both variants parse to the same object; the cell validates to 1.0. The coin the API studies measured and the coin full control reproduced at scale are, in every frontier-scale case across all three studies, the same benign kind: serialization, not meaning.

---

## 4. What the Gate Cannot See

> **Answer:** The residue is small and it is not random — it concentrates by model tier, and it is semantic. Haiku 4.5 returned a different *field value* on structured JSON in 4.0–9.6% of calls depending on plane: pooled across study 2's three planes, 553 parsed responses said the item name was "Corvid CS-220 badge printer" and 34 said "Item Corvid CS-220 badge printer" — two well-formed, schema-valid objects that disagree about the answer. Opus 5 with adaptive thinking flipped a classification label — BILLING to TECHNICAL, both valid members of the instructed set — on 1 of 200 calls first-party, 1 of 200 through Bedrock, and 3 of 200 through Claude Platform on AWS. No deterministic gate can flag any of that, by definition: the outputs are exactly what a valid answer looks like.

The local baseline supplies the most instructive single record in the reanalysis. Study 3's one greedy structured-JSON flip — 1 call in 1,400, on gpt-oss-20b under full control with a fixed seed — is *the same fork as Haiku's*: the name field, "Item Corvid CS-220 badge printer" versus "Corvid CS-220 badge printer", the identical two readings of the identical phrase boundary in the fixture. A budget-tier API model at temperature defaults and a 20-billion-parameter open-weight model under greedy decoding fell into the same semantic fork, because the fork lives in the *input*: the fixture's inventory note reads "Item Corvid CS-220 badge printer, SKU …", and whether "Item" is a label or part of the name is genuinely ambiguous. Where the input is ambiguous, near-tied continuations exist, and near-ties are exactly where every study in this series found instability expressing itself. The instability chooses the ambiguity; the validator cannot referee it, because both readings are defensible.

Open-ended generation is the same limit at full scale. The API studies measured open generation at effectively zero byte-reproducibility — all ~100 responses distinct in every cell — and the local study measured its only cross-machine divergence there. None of that is recoverable by an output gate, because no deterministic acceptance rule exists for free prose. The task ladder the studies froze turns out to double as a validator-coverage ladder: extraction and classification are fully format-gateable, structured JSON is gateable down to its field values, and open generation is not gateable at all.

---

## 5. What Almost Never Happens

> **Answer:** The fail-closed gate rejected 5 outputs of 33,444 — 0.015% — and zero of them were JSON parse failures. Across every structured-JSON cell in three studies, every response parsed: strictly, or after stripping the fence. The five rejections were all Opus 5 classification calls with thinking *disabled*: two leaked a literal `<thinking>` tag into the output, two prefixed the label with a stray `<br>`, and one answered BILLING and then appended an apology — in Russian. A fail-closed gate on these workloads costs essentially nothing in availability and exists for exactly these five outputs: the long tail is strange, rare, and real.

Two operational readings follow. First, the availability objection to fail-closed validation — "rejects will eat my throughput" — finds no support anywhere in this dataset: on deterministically-configured frontier and local models, the reject path fired at one in roughly 6,700 calls. A retry-on-reject policy would have retried five times in three studies.

Second, and less obviously: because rejects and fences are this rare and this patterned, their *rates* are sensitive drift signals. The fence rate tracked the serving door at seventy points of separation in study 2; a reject appearing in a workload that has produced none for a month is information. A validator is usually deployed as a gate. Its byproduct telemetry — reject rate, fence rate, serialization-variant distribution — is a free reproducibility canary on production traffic, no separate probe required. That observation feeds the monitoring design the series' roadmap has pointed at since study 2; it costs a production system nothing to start recording those three rates today.

---

## 6. Sampling Moves Divergence Into Meaning

> **Answer:** Under temperature 0.7, the validator stops rescuing. Study 3's sampled structured-JSON cells: 0.565 byte-reproducible pooled, and only 0.803 post-validator — 19.7% of all sampled calls returned a well-formed, schema-valid object that disagreed with the majority object. The sampled classification cells: 92.8% byte-reproducible, every deviation a different *valid label*, all invisible. On the least stable sampled cell (gpt-oss-20b, CUDA), 44 calls in 100 produced a different valid object. Sampling does not add formatting noise for a validator to clean; it adds meaning-level variation that passes every deterministic check.

The contrast with the deterministic arms could not be sharper, and it inverts the intuition the formatting results might tempt. On greedy and thinking-mode configurations, byte divergence was overwhelmingly cosmetic — the same object wearing different bytes. Under sampling, the same models on the same fixtures produce *different objects* wearing perfectly valid bytes. The positive-control arms in all three studies were built on exactly this property (a sampler that cannot vary is a broken instrument), and the reanalysis quantifies its application-facing face: temperature is not a formatting knob. Teams that run sampled configurations for good reasons — diversity, creativity, exploration — should budget for the consequence: no output-side validation recovers determinism for them. The recovery story in section 3 belongs to deterministic configurations only.

---

## 7. The Ladder, Read as Risk

> **Answer:** Assembled, the reanalysis gives the series' practical risk map. Frontier-tier instability under deterministic configs is a *formatting* risk: large at the byte level, moved dramatically by the serving path, and almost entirely recoverable by a parse-and-canonicalize gate — post-validator 1.000 in every frontier structured-JSON cell measured. Budget-tier instability is a *content* risk: smaller at the byte level but 4–9.6% semantic, unrecoverable by any gate. Sampled configurations are a content risk by design. And ambiguous inputs are where semantic risk lives across every tier and substrate — the same fixture phrase forked a budget API model and an open-weight model under full control into the same two answers.

| Where the instability lives | Measured size | Validator recovers? | Residual class |
|---|---|---|---|
| Frontier tier, serving-path serialization | up to 51 points/cell | yes — to 1.000 in every measured cell | none measured |
| Frontier tier, thinking-mode label flips | ~0.5–1.5% of calls | no | semantic, rare |
| Budget tier, field-value flips | 4–9.6% of calls | no | semantic, systematic |
| Large-model reasoning coin (local, 89:11) | 11 points | yes — cosmetic | none |
| Sampled configs (temp 0.7) | 20–44% of calls | no | semantic, by design |
| Open-ended generation | ~total | no validator exists | unbounded |

One row of that table deserves its own sentence: the reasoning-mode instability that all three studies measured — the API coin-flip, the scale-gated local replication — never once produced a semantic divergence. Every measured face of that coin, on every substrate, parsed to the same object. The mechanism the series has been chasing across three papers is real, physical, and — for a consumer with a parser — benign in every instance observed so far. The instability that actually changes answers came from somewhere else entirely: model tier, sampling, and ambiguity in the input itself.

---

## 8. What To Do With This

> **Answer:** Parse and canonicalize before comparing, storing, or hashing any structured model output — on this evidence it converts most measured byte-instability into determinism at zero model cost. Run the gate fail-closed; the measured availability price was 5 rejects in 33,444 calls. Record the gate's byproduct rates (reject, fence, variant distribution) as drift telemetry. Do not expect any of this to help sampled configurations. And treat unvalidatable output surfaces — free prose above all — as a different risk class requiring process controls rather than output controls.

Concretely, in the order a team would implement it:

1. **Canonicalize at the boundary.** Sorted keys, fixed separators, one serializer. Every downstream equality check, cache key, and content hash should see canonical bytes, never raw model bytes. This single step is worth 17–22 points of pooled reproducibility on the API studies' structured JSON.
2. **Validate schema, not just syntax.** Zero parse failures in 33,444 calls means a bare `JSON.parse` gate would have caught *nothing*; every semantic flip in the dataset parsed cleanly. The schema check is what turns a parser into a gate — and even the schema check catches only format. Field-value risk needs redundancy (ask twice, compare canonically) or tier selection.
3. **Alarm on gate telemetry.** Reject rate and fence rate are near-zero and stable within a serving door respectively in this dataset; movement in either is a substrate change announcing itself on production traffic, for free.
4. **Confine sampling to surfaces that tolerate semantic variance.** A sampled structured-output pipeline inherits a measured ~20% object-level disagreement rate that no gate reduces.
5. **Disambiguate inputs and templates.** The one ambiguous input phrase in these studies is the one place semantic flips appeared on every substrate. Ambiguity in, near-ties out — the cheapest semantic-risk reduction available is editing the input.

This reanalysis is the evidence base behind treating deterministic validation as infrastructure: the [reliability-engineering layer](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) has argued the architecture; these are its measured rates. The residue the validator cannot reach — bounded-autonomy process controls for unvalidatable surfaces — is the province of [*The Finite Chain*](https://isimplifyme.com/whitepapers/the-finite-chain).

---

## 9. Limitations

> **Answer:** This is an unregistered, descriptive reanalysis of three registered studies: it states one reasonable validator and counts its behavior; it tests no hypotheses and reports no confidence intervals. Its validator is one of several defensible designs — a stricter one (no fence tolerance) would reject more and neutralize less; a looser one (any parseable JSON) would neutralize more and check less. The extraction rule is structurally blind to well-formed wrong values, a blindness this dataset never exercised (extraction was byte-perfect everywhere) but that a harder extraction task would. Semantic identity is measured against each cell's own majority object, not against ground truth — a cell that is reliably wrong the same way every time counts as reproducible, which is the correct reading of *reproducibility* and says nothing about *accuracy*. And each task family rides on one fixture: the field-flip finding, in particular, is one ambiguous phrase observed forking three models, not a survey of ambiguity.

The reanalysis also inherits every scope limit of its inputs: three Claude models on three serving planes in fixed windows, five open-weight models on one engine version and two machines, English-language fixtures, and the studies' own exclusion rules. Nothing here extends the underlying claims — it re-buckets them through a gate a production system would actually run.

---

## 10. Reproducing This

> **Answer:** Zero model calls are required. The validator module, its tests, the run inputs, and the emitted report are committed in the public harness repository; the reanalysis re-runs from the committed raw records with one command.

The module is `analysis/validator_catchability.py` in [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness); the emitted report (`reports/validator-catchability-20260730T210644Z.json` and `.md`) contains every per-cell number in this paper. The command in the module's docstring reproduces it from `runs/` verbatim — the same public records the three studies committed, so every number here is checkable against data that predates this paper.

The validator's exact rules are the ones in section 2, implemented in ~200 lines of standard-library Python with no model in the loop: the same measurement stance as every paper in this series. Anyone who prefers a different validator can state it and run it over the same records; any interesting disagreement would be a finding.

---

## References and further reading

- [*The Same Request Twice: Inference Reproducibility on Models With No Sampling Controls*](https://isimplifyme.com/whitepapers/the-same-request-twice) — study 1: the byte-level measurement this paper re-buckets.
- [*The Same Model, Three Doors: Attributing Inference Reproducibility to the Serving Path*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) — study 2: the serving-path attribution; its semantic addendum is this paper's nearest ancestor.
- [*The Same Weights, Two Machines: The Control Ceiling for LLM Reproducibility*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) — study 3: the control ceiling; source of the local records reanalyzed here.
- [*Layer 4: Reliability Engineering*](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) — the architectural argument for validation as infrastructure.
- [*The Finite Chain: Bounding Multi-Agent Autonomy by Construction*](https://isimplifyme.com/whitepapers/the-finite-chain) — process controls for the surfaces no validator covers.
- Raw data, registrations, validator module, and report: [`github.com/iSimplifyMe/determinism-harness`](https://github.com/iSimplifyMe/determinism-harness).
