> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-routing-table](https://isimplifyme.com/whitepapers/the-routing-table)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-08-31 · License: [CC BY 4.0](LICENSE)

# The Routing Table: Where Each Enterprise AI Workload Actually Belongs

A synthesis of eight measured papers into one operating doctrine. Enterprises are being pitched four contradictory answers to "where should our AI run" — local appliances, frontier APIs, cloud model gardens, agent platforms — and each pitch is true somewhere, which is exactly why none of them is an answer. This paper replaces the question "which side do we pick" with a routing table: for each class of workload, where it belongs, which controls it needs, and the measured, public number behind every row.

---

## Abstract

> **Answer:** The doctrine in one sentence: buy capability for judgment, own hardware for replay, validate everything between, and disambiguate inputs before any of it. Judgment-bearing work routes to the most capable model tier you can instrument, because semantic-precision risk tracked capability tier, not deployment location, in every substrate we measured. Byte-exact replay exists only on hardware you control, under state-pinning — no API door offered it. Between those poles, reliability comes from validators and canonicalization (measured recovery to 1.000 on every frontier structured-output cell), not from model choice. And the one failure that survived a change of vendor, tier, and hosting — a flipped field value — was caused by an ambiguous input, which no routing decision fixes.

This is a synthesis, not a new study. It registers no hypotheses and reports no new calls; every number in it is already published in [the series](https://isimplifyme.com/whitepapers) — eight papers spanning four pre-registered studies, a registered manipulation confirmation, a zero-new-calls reanalysis, and two applied measurements: north of forty thousand recorded model calls across two vendors' frontier weights, seven API serving doors plus one agent harness, and five local open-weight models on two machines, with raw records public in [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness). What the series never did is say what an enterprise should *do* about all of it. That is this paper's whole job: the findings, rearranged in the order a system designer would apply them, with the doctrine stated plainly enough to disagree with.

One scope statement before any advice: the studies measured specific models over specific spans — the Claude 5 family and Haiku 4.5, GPT-5.6, and five open-weight models under Ollama. Carrying a finding to a model or door we did not measure is inference, and the text labels it as such wherever it happens.

---

## 1. Every Vendor Is Right, Which Is the Problem

> **Answer:** An enterprise evaluating AI in 2026 hears four pitches that cannot all be the answer: run open-weight models on hardware you control, because control is safety; use the frontier API, because capability is all that matters; use your cloud's model garden, because it is the same model inside your compliance boundary; adopt an agent platform, because raw models are yesterday's interface. Our measurements support a version of each claim — and falsify each one as a universal. They are not competing answers to one question. They are correct answers to four different questions, and the buyer's real task is to ask which question each workload is posing.

The pitches deserve to be stated fairly, because each one survived contact with our data in a narrower form.

- **"Go local — control, privacy, determinism."** True, measured: on owned hardware we drove greedy decoding to byte-identical output in [37 of 41 cells](https://isimplifyme.com/whitepapers/the-same-weights-two-machines), and with [state-pinning](https://isimplifyme.com/whitepapers/the-same-state-the-same-bytes) achieved something no API door offered — byte-reproducible long-form generation. False as stated: the same local substrate reproduced [the identical field-value flip](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) that the budget-tier API model showed, on the same ambiguous input. Location purchased replay. It did not purchase precision.
- **"Use the frontier API — capability is everything."** True, measured: across more than five thousand deterministic structured-JSON calls on Claude weights — and all 1,525 of GPT-5.6's besides — the frontier models returned zero field-value flips, while the budget tier flipped an extracted field value on up to roughly one call in ten, depending on the serving plane (4–9.6%). False as stated: the frontier models' *bytes* were the least stable structured output we measured — a reasoning mode cost up to [39.5 points of byte-reproducibility](https://isimplifyme.com/whitepapers/the-same-request-twice) — and which bytes you receive [depends on the serving door](https://isimplifyme.com/whitepapers/the-same-model-three-doors). Capability bought semantic precision. It did not buy stability, and it did not answer where to run.
- **"Your cloud's model garden — same model, your boundary."** True, measured: the cloud doors carried the same weights — one of them metering [input tokens identical to first-party for the identical schedule](https://isimplifyme.com/whitepapers/the-same-model-five-doors) — and satisfied residency requirements no first-party API matches. False as stated: "same model" is not "same behavior." On Claude weights the thinking-mode reproducibility cost followed the AWS front door (first-party effect +0.003 — a null — versus −0.228 behind Bedrock, same model, same requests); on OpenAI weights the one behavioral knob the model exposes *meant something different* behind each door. The garden is the same species, different weather.
- **"Adopt an agent platform."** True, measured: the harness we instrumented was the most byte-stable structured-output door at its default settings. False as stated: it injected roughly 13,900 tokens of standing scaffold per call — about 108× the API doors' input for the identical schedule — silently defaulted the reasoning knob, exposed no wire-level control, and was [the only door the knob destabilized](https://isimplifyme.com/whitepapers/the-same-model-five-doors). A harness is not a cheaper API; it is a fifth product with its own defaults, and most teams running one have never measured what those defaults do.

A buyer who hears all four pitches and feels unable to choose is not confused. The pitches genuinely conflict at the level they are made — "where should our AI live" — because that level is wrong. Workloads have homes. Companies do not.

---

## 2. The Evidence Base, and What This Paper Is Not

> **Answer:** Everything routed below stands on published, third-party-checkable measurement: four pre-registered studies whose hypotheses were frozen and pushed publicly before data collection, companion runs whose plans were committed pre-data, a reanalysis that added zero new calls, and two applied papers on harness configuration and model provenance. The corpus spans two vendors' frontier weights through seven API serving doors and one agent harness, five open-weight models on two machines and two silicon families, and more than forty thousand recorded calls — every raw record public. This paper adds zero new calls. It is the series' doctrine chapter: opinionated where the data is strong, labeled as inference where it is not, and wrong in checkable ways if it is wrong.

Three structural facts about the evidence matter for everything that follows.

First, the studies used a fixed **task ladder** — extraction, classification, structured JSON, open generation — precisely because reliability is not one property. The ladder's floor held almost everywhere we looked: simple extraction reproduced byte-identically at n=100–200 per cell on every model, both vendors, every door, both machines, harness included — and classification matched it everywhere except five frontier label flips in 600 thinking-enabled calls and five fail-closed rejects in 33,444 records, the accounting [the series' own reanalysis](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) keeps. The ladder's top never stabilized through any API door: open generation produced fully distinct outputs in every positive control on every substrate. All the action — and therefore all the routing difficulty — lives on the middle rung, structured output, where a model must commit judgment to a schema.

Second, the studies separated **bytes from meaning** at every step, and the two diverge constantly. The most vivid single number: across all 33,444 validated outputs of the first three studies, the number of JSON parse failures was zero — every structured-JSON response parsed, strictly or after stripping a fence — while byte-level agreement in the worst structured cells fell below 0.60. Almost everything that looks like nondeterminism at the byte level was serialization cosmetics; almost everything that was *semantic* was rare, tier-concentrated, and input-triggered. An enterprise that measures its AI at the wrong layer will buy the wrong fix.

Third, the corpus keeps finding that **the system is bigger than the model**: the serving door moved behavior that the model's own front door did not; [the harness moved results before the model got a vote](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model); a 3,072-token output cap scored a 122-billion-parameter model at 68% of its true capability and inverted a leaderboard; [token accounting identified the lab behind a stealth model](https://isimplifyme.com/whitepapers/the-tokenizer-is-a-fingerprint) whose operator was undisclosed. Procurement that evaluates "the model" is evaluating a minority share of the deployed system.

What this paper is not: it is not compliance advice — regulatory boundaries differ per door, vendor, and jurisdiction, and nothing here substitutes for that analysis. It is not a leaderboard — no vendor is recommended over another anywhere in it. And it is not permanent — several findings are dated observations from measured spans, and the discipline that produced them matters more than any row of the table they support.

---

## 3. The First Row Has No Model in It

> **Answer:** The most important routing decision is the one that removes the model: any computation whose answer is checkable by rule — arithmetic that reaches a customer, eligibility against written criteria, totals, dates, thresholds, reconciliations — routes to deterministic code, full stop. This is not caution; it is dominance. Code is byte-reproducible for free, auditable by reading, and testable exhaustively, and no measured property of any model on any door approaches that. The series spent forty thousand calls establishing how close controlled inference can get to determinism. The answer — impressively close, on owned hardware, under discipline — is still an argument for code wherever code can do the job.

The corollary handles scale: when a workload is high-volume but only partly judgment-bearing — score ten thousand records, find the hundred worth a person's time — the deterministic layer goes *first*. Filters, joins, arithmetic screens, and rules run across the full population at effectively zero marginal cost and perfect reproducibility; the model sees only the short list the rules produce. This inverts the default architecture most teams reach for, where the model reads everything and the rules clean up afterward. Routing the sweep to code and the judgment to the model is cheaper and — because the expensive nondeterministic component now touches two orders of magnitude fewer items — dramatically easier to monitor.

Two tests identify first-row work reliably. The *regeneration test*: if running it twice must produce the identical answer for the answer to be correct at all, it is code. The *explanation test*: if a customer, auditor, or regulator asking "why this number" must receive a derivation rather than a rationale, it is code. Work that fails both tests — where the output is a judgment a reasonable expert could defend in more than one form — is what the rest of the table is for.

The anti-pattern this row exists to stop has a recognizable shape: a model inside a loop that computes something a spreadsheet could, justified by "it's usually right and it was faster to ship." Every finding in this series about validation, doors, and tiers is a description of the price of that sentence.

---

## 4. Capability Tier Is a Precision Control, Not a Budget Line

> **Answer:** Where a workload requires judgment — extracting contested values from messy documents, scoring against criteria that interact, resolving what a paragraph actually asserts — the measured rule is: semantic-precision risk tracks model capability tier, not deployment location. On a deliberately ambiguous input, the budget-tier model returned a different extracted field value in 4–9.6% of calls per serving plane; the frontier-tier models returned zero field-value flips across more than five thousand deterministic structured-JSON calls on Claude weights, and all 1,525 of GPT-5.6's besides; and a 20-billion-parameter open-weight model under full greedy control on owned hardware fell into the *same* fork as the budget tier — one flipped call in 1,400. You cannot host your way out of a capability problem.

The finding is worth stating carefully, because it cuts against both cost intuition and control intuition at once. The [validator-catchability reanalysis](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) split all residual divergence in 33,444 records into what validation can catch and what it cannot. What it cannot catch — the divergence that survives schema validation and canonicalization because every variant is *valid* — was concentrated almost entirely in the budget tier, and it was semantic: different field values, different labels, a different reading of the same sentence. The frontier tier's instability was the opposite profile: noisy at the byte level, unanimous at the level of meaning — every one of its thousands of structured-JSON variants parsed to the identical object.

For a system designer this converts directly into placement:

- **Judgment-bearing extraction and scoring pin to the frontier tier.** At enterprise document volumes, model cost is usually a rounding error against the cost of one wrong field value that enters a workflow — a misread party name, a flipped comparable, a wrong dosage unit. Downgrading the tier to save single-digit dollars per thousand documents trades a measured content risk for noise savings. Where the measured flip rate would be intolerable from a person, it is intolerable from the budget tier.
- **The settled rungs tolerate any tier that passes your fixtures.** Simple classification and simple extraction reproduced byte-identically on every substrate we measured — budget tier and 20B local models included — with the reanalysis's full ledger against them totaling five frontier label flips and five rejects across 33,444 records. High-volume routing, tagging, and triage can chase price freely — *provided* the fixtures proving "simple" are yours and not the vendor's, and provided a validator stands behind them.
- **A local open-weight model is not a precision purchase.** It can be the right purchase — §7 gives the three triggers — but on the one axis this section is about, the local substrate reproduced the budget tier's failure, not the frontier tier's measured zero on that field. Teams told "on-prem is the safe choice" should ask: safe against what? Against data leaving the building, yes. Against a wrong answer entering it, no.

One inference boundary, stated plainly: "zero flips" describes the models and fixtures measured, at n in the hundreds per cell — it is a strong measured contrast, not a guarantee that frontier models cannot flip a field. The operational reading is comparative, and the comparative reading was unambiguous everywhere we looked.

---

## 5. The Door Is a Configuration Decision With Behavioral Consequences

> **Answer:** The same weights behind different serving doors are not operationally the same product, on either vendor we measured. On Claude weights, enabling the thinking mode cost Opus 5 a bounded null through the vendor's first-party door (+0.003, while Sonnet 5 kept a −13.3-point intrinsic share) and cost 16–44 points through the AWS-fronted doors — with a forbidden markdown fence appearing in ~85% of AWS-door calls versus 9–17% first-party. On OpenAI weights, pinning reasoning effort `high` *raised* byte-stability behind both Bedrock doors by 15–20 points and flipped which byte-variant was modal, did nothing measurable on OpenAI's own API, and destabilized the codex harness. The knob you validated on one door does not mean the same thing behind the next one.

The enterprise consequences are concrete and mostly cheap to act on.

- **Choose the door on data governance, capacity, and auditability — then re-validate behavior on the door you chose.** The doors differ in what they let you see as much as in what they emit: one first-party door persists responses server-side by default (`store:true`); the translated Bedrock door reports reasoning spend only inside an aggregate token count; the harness door hides nearly everything. What you can audit is part of what you are buying.
- **Pin every behavioral knob per workload, per door, in code.** Thinking modes and effort settings are reproducibility settings, not just cost settings, and their effects inverted across doors within one vendor's lineup. A defaults change upstream — the harness defaulted effort silently — is a production behavior change you did not review. Pin, and alarm on the returned configuration where the door reports one.
- **Log the response's served-model identifier next to your pinned request identifier on every call.** Some doors return undated model identifiers, which makes your own logs the only version-drift instrument you will ever have. This costs one column.
- **Plan for capacity behavior as door behavior.** In one study every one of the 85 excluded calls was the same door returning 529s, all in the overnight window — the other doors recorded zero. Retry budgets and schedule placement are per-door properties; measure them where you run.
- **Treat harness-mediated usage as its own product.** If teams script through a subscription CLI today, they are already in production on its scaffold, its silent defaults, and its rate windows. Migrating "up" to the API later inherits a different output distribution twice over — once for the scaffold, once for the knob semantics. Measure before you migrate, in either direction.

What the series does *not* license is a ranking. The AWS-fronted doors were the less byte-stable path for Claude weights with thinking enabled, and the *more* stable path for OpenAI weights with effort pinned high, within weeks of each other. The durable finding is not "door X is better"; it is that door effects are real, model-family-specific, and cheap to measure — the five-door replication cost about thirty dollars in API fees and re-runs in an afternoon, using [a public harness](https://github.com/iSimplifyMe/determinism-harness) — and that any vendor claim of cross-door equivalence should arrive with receipts at the byte level or not at all.

---

## 6. Validation Is Where Reliability Actually Lives

> **Answer:** The highest-leverage reliability investment in the measured corpus was not model choice, door choice, or hosting — it was the validation boundary. Schema validation plus canonical re-serialization converted nearly all measured byte-instability into determinism at zero model cost: every frontier structured-JSON cell validated to 1.000, with a worst-case single-cell recovery of 51 points. The same reanalysis carries a warning: a bare `JSON.parse` caught *nothing* — zero JSON parse failures anywhere in 33,444 validated outputs — because modern failures are not malformed syntax; they are valid JSON containing a different answer. Validate meaning against schema, canonicalize before any hash or comparison, and never let raw bytes serve as a cache key or a diff — store them, alongside their canonical form, in the audit record.

The validation doctrine, in the order a pipeline encounters it:

- **Disambiguate inputs first.** The one failure that refused to stay in a single substrate — the budget tier at its API defaults and a 20-billion-parameter local model under full greedy control fell into the *same* fork on the same ambiguous phrase boundary, where a label could be read as part of a value. Near-ties in the input become coin-flips in the output, and no downstream control can un-flip them. A one-afternoon review of field definitions, extraction instructions, and fixture phrasing — tightening every place where two readings are defensible — is the cheapest precision improvement this corpus identifies. This is the doctrine's least glamorous clause and its best return.
- **Canonicalize at every boundary.** One sorted-keys, fixed-separator serializer, applied after schema validation — it is the validator's companion, never its substitute — and before anything is hashed or compared. On the measured data this single function erases the serialization coin-flips, the fence cosmetics, and the `349.5`-versus-`349.50` class of variant entirely. Byte-comparing pipelines — hash-keyed caches, dedupe layers, change detectors — otherwise sit on a knife edge whose bias moves with door and knob settings.
- **Validate against schema, and treat rejects as telemetry.** Rejects were rare in the measured corpus — five in 33,444 records — which is exactly what makes their *rate* a sensitive drift signal. The same holds for fence-stripping events and canonical-variant distribution: the cheapest substrate canary an enterprise can run is three counters on traffic it is already paying for. When a serving stack changes under you silently — and we measured them doing so — these rates move first.
- **Route disagreement to people.** For judgment-bearing fields, running the call twice and comparing *canonicalized, validated* outputs converts the series' central observation — divergence concentrates where inputs are ambiguous — into a queue: agreement proceeds, disagreement escalates to a human. Under sampling, divergence moved into meaning (roughly 20% object-level disagreement in sampled cells), which makes ask-twice a *detector* of exactly the items a person should see. The agreement rate doubles as a free calibration metric.
- **Know where no validator exists.** Free prose admits no deterministic validator — open generation produced fully distinct outputs in every positive control on every substrate, and no schema can adjudicate between two well-written paragraphs. Long-form output that matters routes to human review, or to §7's replay regime, or it does not ship. Any vendor promising validated prose is describing a review process or describing nothing.

Note what this section did not need: any particular model, door, or host. The validation layer is the portable part of the doctrine — it travels across every substrate we measured, it is almost free, and it is where "governed and monitored" stops being a slide and becomes a measured recovery rate.

---

## 7. Owned Hardware Buys Replay and Custody, Not Correctness

> **Answer:** Byte-exact reproducibility — the property auditors, replication, and forensic reconstruction actually require — existed in the measured corpus only on hardware the operator controls. On owned machines, greedy decoding was byte-identical in 37 of 41 cells, identical weights produced byte-identical short outputs across Apple and NVIDIA silicon, and the residual "nondeterminism" [decomposed into a mixture of server states](https://isimplifyme.com/whitepapers/the-same-state-the-same-bytes), each internally byte-deterministic, selected by instance history — a variable we then manipulated on schedule, five cycles out of five. State-pinning turned that mechanism into an engineering pattern: byte-reproducible long-form generation, the one thing no API door offered at any price. That is what local buys. It is also *all* it buys: the same substrate reproduced the budget tier's semantic fork, and the capability ceiling of deployable open-weight models — an inference from the wider market, not a series measurement — is not a fact that ownership moves.

The honest routing rule for local deployment has three triggers, and most enterprises currently being pitched appliances have none of them:

1. **Replay:** a workload where outputs must be *re-derived* byte-exactly — scientific replication, certain regulatory reconstructions, dispute forensics where "run it again and compare" is the procedure. This is rarer than it sounds; §8 shows that most audit requirements are satisfied by storage, not regeneration.
2. **Custody:** data that contractually or statutorily cannot transit any third party — not "prefer it didn't," but *cannot*. Note that a dedicated cloud account with the enterprise's own keys and boundary answers most residency-and-control requirements without owning GPUs; custody routes local only when the requirement is absolute.
3. **Volume:** sustained throughput at which owned-hardware economics genuinely cross over API pricing — a spreadsheet question, answerable per workload, whose columns are known (hardware amortization, power, the operations headcount the discipline below implies, and API list price at your measured token volumes) and that pitch decks systematically answer with adjectives instead.

Where a trigger is real, the measured discipline that makes local *work* is specific: greedy decoding; per-model pinned engine versions and weight digests; state-pinning for anything long-form; and the knowledge that per-state bytes were session-scoped — stable across four server sessions, then re-rolled in the fifth — so replay regimes must pin the serving session or store what §8 stores. Two more local findings earn their place in any deployment plan: concurrency did not perturb greedy outputs (a registered null — the mixture-of-experts-versus-dense contrast measured +0.005 with a confidence interval tight around zero, and both MoE models sat at exactly zero effect under four-way load), and thinking-mode instability under full control was scale-gated — a measured null on the other three models, real (−11 points, with an 89:11 two-variant split) on the 120-billion-parameter one. Determinism on owned hardware is achievable, is *not* automatic, and degrades with exactly the model scale that makes local capability respectable.

The pitch this section exists to disarm is "on-prem means you finally control the model." What the operator controls is the serving substrate; the measured behavior — the forks, the scale-gated instability, the capability ceiling — came with the weights.

---

## 8. Audit Needs Snapshots, Not Replay

> **Answer:** The most common argument for heroic determinism — "we need to reproduce what the system said for the auditors" — is answered by architecture, not by physics: generate once at write time, version the artifact, and serve the stored copy forever. "What did the system say on date X" becomes a storage lookup with a hash, not a regeneration gamble. Every stored AI artifact carries its request hash, the raw response with its hash, the canonicalized response hash, returned model identifier, pinned model identifier, knob settings, and timestamp — at which point byte-replay is unnecessary for audit on *any* substrate, including API doors where it is impossible. Replay-grade infrastructure (§7) is reserved for workloads whose procedures genuinely re-derive outputs, which most audit regimes do not.

Snapshot-at-write resolves what would otherwise be a hard conflict in the doctrine: §4 routes judgment to frontier API tiers, and API doors cannot promise byte-replay — sessions re-rolled bytes on identical requests even locally, some doors serve no dated model snapshot at all, and serving behavior moved measurably under our instruments across the series. If audit required regeneration, those facts would force precision-critical work onto owned hardware and away from the capability tier that §4 showed it needs. It does not, because regeneration is the wrong operation: the record of an AI decision is the artifact that entered the workflow, not the process's ability to produce it again on demand.

The policy has one more clause, and it converts model churn from a threat into information: when a regeneration *is* run — a model-version bump validation, a dispute, a what-would-it-say-now — it is a new versioned artifact compared against the old one, never an overwrite. Divergence between the two is not corruption; it is a measurement of drift between substrates, timestamped and kept. Teams that adopt this clause discover their golden-file suites and their audit trail are the same system: every model bump regenerates baselines as a *new* baseline with a diff report, the diff is reviewed like a code change, and "the model changed under us" stops being an incident and becomes a ritual.

For buyers evaluating vendors, this section compresses to one due-diligence question: "show me the stored artifact for a decision from last quarter, with its hashes and model identifiers." A vendor who answers with a demo of re-running the prompt has told you they do not have an audit trail. They have a slot machine with a logbook.

---

## 9. The Routing Table

> **Answer:** Routing by workload, in the order each rule fires: (1) rule-checkable computation → deterministic code, always; (2) high-volume screening → deterministic sweep first, model only on the short list; (3) judgment-bearing extraction and scoring → frontier tier, any well-instrumented door, behind schema validation, canonicalization, and ask-twice; (4) simple classification and extraction at scale → cheapest tier that passes *your* fixtures, behind the same validators; (5) long-form generation that matters → human review, because no deterministic validator exists for prose; (6) byte-exact replay obligations → owned hardware under state-pinning; (7) absolute custody constraints → owned or single-tenant infrastructure, chosen for custody, not correctness; (8) audit and recordkeeping → snapshot-at-write with hashes, on every route above.

The table, with the measured backing per row:

| # | Workload class | Route | Load-bearing measurement |
|---|---|---|---|
| 1 | Arithmetic, rules, totals, eligibility — anything rule-checkable that reaches a customer | Deterministic code; no model | Dominance, not measurement: nothing measured anywhere in the series approaches code's free byte-reproducibility |
| 2 | High-volume screening (rank 10,000, surface 100) | Code sweeps the population; model sees the short list | Ladder floor: simple rungs at 1.000 in all but ten of 33,444 records (ledger: row 4) — the expensive component adds nothing to the sweep |
| 3 | Judgment-bearing extraction, scoring, interpretation | Frontier tier, well-instrumented door, full §6 validation stack | Budget tier flipped field values in 4–9.6% of calls per plane; frontier tier: zero across >5,000 Claude-weights calls and all 1,525 GPT-5.6 calls; local 20B reproduced the budget-tier fork |
| 4 | Simple classification, tagging, triage at scale | Cheapest tier passing your golden fixtures, behind validators | Floor held at n=100–200 per cell on every substrate — the full ledger against it: five frontier label flips and five rejects in 33,444 records |
| 5 | Long-form drafts, letters, narratives that matter | Model drafts; a person owns it | Open generation: 100% distinct outputs in every positive control; free prose admits no deterministic validator |
| 6 | Byte-exact replay obligations (re-derivation procedures) | Owned hardware, greedy, state-pinned, pinned engine + digests | 37/41 greedy cells at 1.000; state flip manipulated 5/5; per-state bytes session-scoped — pin the session or snapshot |
| 7 | Absolute custody constraints | Owned or single-tenant boundary; §6 stack unchanged | A custody decision, not a precision one: the local substrate reproduced the semantic fork — location ≠ correctness |
| 8 | Audit, recordkeeping, "what did it say on date X" | Snapshot-at-write, versioned, hashed; regeneration = new artifact + diff | Serving stacks and per-session bytes changed under instruments mid-series; storage is the only stable record |

And the operating discipline that rides every row — none of it is a module, all of it is policy:

- **Pin** model identifiers and every behavioral knob per workload, per door, in code; re-validate the knobs when the door changes, because their effects did not transfer between doors in either vendor's measured lineup.
- **Log** request hash, wire hash where you can get one, raw and canonicalized response hashes, returned model identifier, and knob settings on every call. On undated-identifier doors this is your only drift instrument.
- **Watch three rates** — schema rejects, fence-strips, canonical-variant distribution — on production traffic. They separated serving substrates by up to seventy points in 33,444 records, and they cost three counters.
- **Ask twice** on judgment-bearing fields; escalate disagreement to a person; keep the agreement rate as a calibration curve.
- **Re-baseline goldens** at every model bump as a new versioned baseline with a reviewed diff — never edit a baseline in place.
- **Schedule around measured capacity**, not assumed capacity; one measured door concentrated all of its failures in one overnight window while its neighbors recorded zero.
- **Evaluate in your own configuration.** A [harness setting scored a model at 68% of its capability and inverted a ranking](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model); a deterministic failure loop on one serving stack vanished on a second stack serving identical weights. A benchmark verdict attaches to model *plus* stack, so a procurement bake-off run on the vendor's preferred harness measured the vendor's harness.
- **Verify provenance where it matters.** Serving substrates are representations until checked — [token accounting alone identified the lab behind an undisclosed model](https://isimplifyme.com/whitepapers/the-tokenizer-is-a-fingerprint) at a cost of zero dollars. The same class of check verifies that the door you bought serves the lineage you think it does.

Eight rows and eight habits. Nothing in them requires believing this series' interpretation of anything — every claim above traces to a public raw record, and the doctrine's response to disagreement is the same as its response to drift: measure again, in your configuration, and route on what you find.

---

## 10. Limitations

> **Answer:** A doctrine synthesized from measurements inherits their scope. The studies measured specific model families over specific spans — Claude 5-family and Haiku 4.5 through three doors, GPT-5.6 through five, five open-weight models under one engine on two machines — and door behavior is a property of a span, not a constant: two findings about the same class of door pointed in opposite directions weeks apart on different weights. The tier→precision rule rests on measured contrasts at n in the hundreds per cell on deliberately constructed fixtures; it is a strong comparative result, not a warranty. Every "route it here" in this paper is an argument from that evidence, stated so it can be checked, priced to re-verify — and expected to be re-verified rather than believed.

The specific inheritances a careful reader should carry:

1. **Fixture scope.** The task ladder used one production-shaped fixture per rung. The two-variant knife edges and the specific flip rates are those fixtures'; what replicated across fixtures, models, vendors, and machines is the *pattern* — floors that hold, prose that never converges, structured output that forks where inputs are ambiguous.
2. **Span scope.** Serving doors changed measurably under our instruments within the series. Any number attached to a door is a dated observation. The doctrine's door advice is therefore procedural — measure, pin, watch rates — rather than a standing verdict on any vendor.
3. **Model scope.** Unmeasured models inherit nothing. Where an enterprise's model is not one we measured, the transferable assets are the method and the harness, both public, not the numbers.
4. **The synthesis itself is not pre-registered.** The underlying studies froze hypotheses before data; this paper selects and arranges their results, and selection is judgment. The mitigation is that every selected number links to a frozen registration or a labeled descriptive in a public record — the arrangement is arguable, the ingredients are not.
5. **No compliance claims.** Door and hosting choices carry regulatory consequences this paper does not analyze. The routing table orders the *operational* considerations; counsel orders the rest.
6. **Economics are pointed at, not computed.** Crossover volumes for owned hardware and tier pricing move too fast to publish; the paper's claim is only that they are spreadsheet questions, and that adjectives are not answers.

---

## Reproducing This

> **Answer:** Zero calls and zero credentials are required to check this paper, because it adds no new data — reproducing it means re-deriving its citations. Every study number traces to a frozen registration or committed raw record in [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness), whose analysis re-runs from committed records in seconds; the two applied papers carry their own public companion repositories, linked from their pages. The arrangement — which findings become which routing rows — is this paper's only contribution, and disagreeing with it requires no API key: re-run the analyses, read the registrations, and check whether the rows follow.

---

## References and further reading

- [`iSimplifyMe/determinism-harness`](https://github.com/iSimplifyMe/determinism-harness) — all pre-registrations, the measurement harness, analysis code, and complete raw records behind every study number cited here; MIT. The two applied papers carry their own public companion repositories, linked from their pages
- [*The Same Request Twice*](https://isimplifyme.com/whitepapers/the-same-request-twice) — the task ladder, the byte-versus-semantic discipline, and the tier finding's first appearance
- [*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors) — serving-path attribution on Claude weights: the thinking-mode cost follows the AWS front door
- [*The Same Weights, Two Machines*](https://isimplifyme.com/whitepapers/the-same-weights-two-machines) — the control ceiling on owned hardware, cross-silicon byte-identity, and scale-gated thinking instability
- [*The Same Object, Different Bytes*](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) — what validators recover (everything cosmetic) and what they cannot see (rare, semantic, tier-concentrated)
- [*The Same State, The Same Bytes*](https://isimplifyme.com/whitepapers/the-same-state-the-same-bytes) — state mixture masquerading as nondeterminism, and the state-pinning pattern behind §7
- [*The Same Model, Five Doors*](https://isimplifyme.com/whitepapers/the-same-model-five-doors) — the vendor-boundary replication: door effects on OpenAI weights, and the harness as a fifth product
- [*The Ceiling Is Not the Model*](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model) — harness configuration fabricating benchmark results; the evaluation discipline behind §9's bake-off rule
- [*The Tokenizer Is a Fingerprint*](https://isimplifyme.com/whitepapers/the-tokenizer-is-a-fingerprint) — provenance verification from token counts; the substrate-verification habit
- He, H. et al., *Defeating Nondeterminism in LLM Inference* (Thinking Machines Lab, 2025) — the batch-invariance mechanism analysis complementing the black-box measurements here
