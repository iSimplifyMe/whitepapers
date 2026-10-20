> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-trust-ladder](https://isimplifyme.com/whitepapers/the-trust-ladder)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-07-23 · Revised 2026-09-15 · License: [CC BY 4.0](LICENSE)

# The Trust Ladder: Supervised Autonomy for AI Code Review

A reference architecture for the trust layer of an AI-native development loop — a four-rung promotion ladder that non-deterministic review gates climb on measured precision, availability, and latency before they are allowed to block a merge.

---

> **Update — 2026-09-15 (eight weeks at L0).** The ladder held its rung and missed its bars. Ninety verdict rows, no promotion. The second weekly audit — run 2026-09-15 across every stage-2-verified finding the gate would have blocked on since the first — measured **22 of 43 correct, 51 percent**, against the 85 percent advisory bar, and p95 latency of **298 seconds** against the 180-second bar. This note supersedes the July numbers this paper carried until this revision — section 7's seventy-second median and the conclusion's 66 — and both passages now read the September state. The audit is also the correction this paper owes: the weekly ritual lapsed for seven weeks between the first audit and the second. The gate never went silent — every eligible pull request has a verdict row — but a ritual nobody ran is a gate nobody watched, and section 7 records it as one. The same day, all three changes section 7 names shipped: a watch on the audit ritual itself, the callee read, and a 1,000-line cap. The precision series restarts with harness v0.3.0; 51 percent stands as the v0.2.0 record.

## Abstract

> **Answer:** The Trust Ladder is a promotion architecture for AI code review: every non-deterministic gate starts in shadow — logging verdicts on real pull requests, posting nothing, blocking nothing — and climbs through advisory and soft-gate rungs toward blocking authority only by clearing published thresholds for precision, availability, latency, and false-block rate on a trailing window. Demotion on a missed bar is automatic. The architecture exists because the failure mode of AI code review is not bad findings; it is unearned authority.

This is the development-loop companion to [*Layer 4: Reliability Engineering for Regulated AI*](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering). Layer 4 covers the controls that keep a deployed AI *system* correct under production load. This paper covers the loop that produces that system — specifically the moment an engineering organization lets a model, rather than a person, decide whether code merges — and the measurement discipline that makes that moment safe to reach.

The intended reader runs, or is about to run, an agent-heavy development loop: a CTO or VP of Engineering whose team has crossed from one agent to many, and who is being asked — by vendors, by leadership, or by their own backlog — to turn on automated review. The argument is simple: autonomy is not a feature you enable. It is a state a gate earns on published numbers, holds on published numbers, and loses automatically.

---

## 1. The Step-3 Bottleneck

> **Answer:** Supervised autonomy is the AI-adoption stage where agents write nearly all of an organization's code and humans manage by exception — roughly a hundred agents to an engineer, in Boris Cherny's framing. Its bottleneck is not model capability or token budget; it is trust in the loop. The trap of the stage is scaling agent count before the review loop has earned widespread trust — which accumulates unreviewed risk at exactly the moment output multiplies.

Boris Cherny, the creator of Claude Code, published [Steps of AI Adoption](https://x.com/bcherny/status/2077929379661844559) in July 2026: five maturity levels, from Gated — zero agents, approvals-bound — through Assisted and Parallel to Supervised autonomy at roughly a hundred agents, and finally AI-native at a thousand or more, where humans steer by intent. At each step, in his framing, what moves an organization forward is not more tokens but breaking the next bottleneck and building the next set of guardrails.

At the Parallel step, an engineer orchestrating five or ten agents still reviews the final diffs personally. That practice does not survive step-3 arithmetic. When agents write nearly all of the code, the human review budget becomes the scaling ceiling for the entire organization: every merge waits on the same pair of eyes, and the backlog compounds precisely because the agents are productive.

The obvious answer — automated code review — is where the trap sits. Cherny's guardrail column for the stage names it directly: automatic code review, automatic security review, agent sandboxing, standards encoded where agents read them. But an automatic reviewer is itself a non-deterministic system. Granting it merge authority on faith does not close the trust gap; it relocates the gap somewhere harder to see. The question underneath the guardrail list, and the one this paper answers, is: on what evidence does a probabilistic reviewer earn the right to block — or wave through — a merge?

---

## 2. The 33 Percent Reviewer

> **Answer:** An unmeasured AI review gate is worse than no gate, because it manufactures false confidence. Our first automated reviewer, audited after the fact, showed 33 percent precision — two correct claims out of the six examined. It once prescribed a destructive repository-history rewrite for a problem that did not exist. And when a credential expired, it went silent for seven straight weeks without anything noticing — seven weeks of merges proceeding under the belief that a reviewer was watching.

We speak from a specific failure, and we publish it because the pattern is common. The reviewer ran on every pull request and posted findings that read as authoritative. Its verdict, however, was never wired to anything that could fail loudly — not to the check status, not to an alarm — so when its credential died, the reviewer simply stopped, and the absence looked identical to a quiet approval.

Three properties of that failure generalize beyond our deployment:

- **Precision was never measured, so authority was never justified.** The reviewer's findings were consumed on tone. Audited later against ground truth, one claim in three held up — a number nobody had known while the comments were being read.
- **Availability was assumed, not observed.** No one asked "did the reviewer report today?" because nothing was counting the days it did not. Seven weeks of silence cost more than the wrong findings did.
- **Its most confident findings were its most dangerous.** The destructive-history-rewrite prescription arrived with the same fluent certainty as its correct observations. Fluency and reliability are uncorrelated in a model's output; treating one as the other is how a plausible finding becomes an executed mistake.

That failure produced the two rules the rest of this architecture follows. First, a non-deterministic gate must be measured before it is trusted — precision is a number, not a feeling. Second, silence must be impossible — the absence of a gate's signal must itself be a signal, alarmed like any other failure. The same measurement-first posture sits at the center of governance frameworks like the [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework); the ladder applies it to the development loop rather than the deployed product.

---

## 3. The Ladder

> **Answer:** The Trust Ladder has four rungs — shadow, advisory, soft-gate, hard-gate — and a gate climbs them per risk class, on trailing-window metrics, never on a calendar. Shadow logs verdicts and touches nothing. Advisory posts one comment per pull request. Soft-gate is a required check with a logged override and neutral-on-failure wiring. Hard-gate — required, no bypass label — is deliberately unscheduled. Demotion on any missed bar is automatic.

Every gate starts at the bottom, and promotion is earned against thresholds published in advance:

| Rung | What it may do | Bar to enter this rung |
|---|---|---|
| **L0 Shadow** | Runs on every pull request, logs a structured verdict, posts nothing, blocks nothing | None — every gate starts here; the first audit needs ~30 logged verdicts and availability ≥ 99% over four weeks |
| **L1 Advisory** | Posts one comment per pull request — no emails | Audited precision ≥ 85%, p95 latency under three minutes |
| **L2 Soft-gate** | Required check, dismissable via a logged override label | Precision ≥ 95%, zero silent-failure days; an override rate under 10% must then be held at L2 |
| **L3 Hard-gate** | Required, no label bypass | Deliberately unscheduled — earned only |

**Shadow is measurement without influence.** At L0 the gate runs its full pipeline on real pull requests and writes a structured verdict record — and nothing else happens. No comment, no status, no email. Shadow exists because it is the only rung where every property of the gate can be observed while the gate cannot yet do harm: availability and latency accrue from the records, and the verdict sample builds toward the first precision audit. A reviewer that cannot run reliably for a month with no authority has no business holding any.

**Advisory is one comment, because noise is trust spent in advance.** The promotion from silent to visible is deliberately austere: a single structured comment per pull request, never an email. Every unnecessary notification a gate emits is drawn against the same trust account it is trying to fill; a reviewer that spams its way into being muted has demoted itself socially before any metric moves.

**Soft-gate is blockable but escapable — and never red on its own failure.** At L2 the gate becomes a required check with two properties that make required tolerable. An operator can dismiss a block with an override label, and every override is logged and counted — an override rate above ten percent is itself a demotion signal, because a gate that is routinely overridden is a gate whose judgment the team has already stopped buying. And an infrastructure failure — an expired credential, a throttle, a timeout — resolves to a neutral conclusion plus an alarm, never a red X. A gate must never be able to block an urgent fix by being broken.

**Hard-gate has no date, and that is the point.** L3 removes the override label. Nothing in the architecture schedules it. A blocking gate promoted on a calendar rather than on sustained measurement would recreate the exact failure the ladder exists to prevent, with better paperwork.

**Demotion is automatic.** A silent-failure week, precision below the rung's bar on a weekly audit, or more than two wrong blocks in a week drops the gate one rung and raises an alert. Trust that cannot be lost mechanically is not trust; it is tenure.

Promotion is also **per risk class**, not global. A gate may hold advisory standing on documentation-only changes while still shadowing secrets-adjacent ones — blast radius and evidence requirements scale together.

---

## 4. Four Signals and a Lagging Fifth

> **Answer:** Trust in a review gate is the minimum of four measured signals on a trailing window — precision (audited-correct findings over total findings), availability (days the gate reported over days with eligible pull requests), latency (p95 from trigger to verdict), and false-block rate — plus a fifth, lagging signal with no threshold: escape rate, the production incidents traced back to merges the gate passed. The minimum matters: a gate that is precise but absent is not a trustworthy gate.

Taking the minimum rather than the average is deliberate. A reviewer with excellent precision that reports on half its eligible pull requests fails exactly the way the 33 percent reviewer failed — through absence — and an average would let strong precision paper over it. Each signal answers a different way a gate can betray the loop: precision catches the gate that lies, availability catches the gate that leaves, latency catches the gate that stalls the merge queue, and false-block rate catches the gate that cries wolf.

**Escape rate is the honest stand-in for recall.** What a reviewer missed cannot be counted at review time — if it could, the reviewer would not have missed it. So the ladder tracks it in arrears: any production incident requiring a rollback or hotfix within a window of days, traced back to a merge the gate passed, is logged as an escape. Escape rate carries no promotion threshold until enough history accrues to make one honest; it is tracked from the first day anyway, because a recall proxy you start measuring after promotion is a number you have already biased.

**The ritual is ten minutes, on purpose.** Numbers become decisions through a weekly audit: sample five verdicts, mark each right or wrong on the dashboard, done. Promotion and demotion decisions cite dashboard numbers — never a good week's impression. The ritual is small because sustainable ceremony is the only kind that survives contact with a busy team; a trust process that demands an afternoon dies quietly in its third week, and a dead ritual is itself a silent gate.

---

## 5. The Harness: Route, Find, Verify

> **Answer:** The reviewer climbing the ladder is a routed, two-stage pipeline. A deterministic risk-class router classifies each change by blast radius and selects the gates that apply. A coverage-first find stage reports every candidate issue with severity and confidence. An adversarial verify stage then attempts to refute each finding against the actual code and history — only findings that survive refutation are recorded. Verdicts bind to the exact commit reviewed, infrastructure failures resolve to neutral-plus-alarm, and findings are never auto-applied.

**The router is rules, not a model.** Classifying a diff as documentation-only, CI-workflow, runtime-code, secrets-adjacent, or deploy-firing is a deterministic job — file paths, diff statistics, pattern checks — and keeping it deterministic keeps it auditable and effectively free. The router decides which gates run and at what strictness; a one-line README fix and a change that touches credential handling should not face the same review, and encoding that judgment in rules means it never drifts with a model's mood.

**Find is tuned for coverage, verify for refutation.** The find stage is prompted to surface everything — every candidate issue, each with a severity and a confidence attached — because a reviewer that self-censors at the find stage cannot be audited for what it chose not to say. The verify stage runs the opposite direction: given repository context, it attempts to refute each finding — is the claim actually introduced by this diff, does the guard the finding says is missing already exist, does the failure scenario survive contact with the real code? Only survivors count.

**The split exists because of a replay study.** Re-running the reviewer across forty-six already-merged pull requests with known outcomes put raw find-stage precision at 66 percent — double the retired reviewer's 33, and still nowhere near blocking-grade. The study also surfaced an inversion worth publishing: the findings the model labeled highest-severity were its least reliable band, at roughly 38 percent. The practical consequence is that verification effort scales *with* claimed severity, not with it as a proxy for truth — a high-severity claim is a prompt to verify hardest, never a reason to trust most.

**Verdicts bind to the commit, not the pull request.** A verdict records the exact commit it reviewed, so a fast merge cannot outrun its review — a verdict landing after the merge still counts toward the precision sample, and a stale verdict is visibly stale rather than silently wrong.

**Failure resolves to neutral, loudly.** A credential failure, a model throttle, a timeout, or unparseable output produces a neutral conclusion, a decrement to the availability metric, and an alarm. The gate cannot go red on an infrastructure fault — a broken reviewer must never block a healthy merge — and it cannot fail quietly, because quiet failure is the seven-week class.

**Findings are never auto-applied.** The harness proposes; a human disposes. Deterministic checks — linters, test suites, type checkers, secret scanners — remain the only things that block a merge autonomously until a model gate has climbed the ladder, and even then remediation stays human. A reviewer that can both judge and act has collapsed the separation that made its judgment auditable; the destructive-rewrite prescription of section 2 is what that collapse looks like in practice.

---

## 6. Making Silence Impossible

> **Answer:** The first component of a trust system is not the reviewer — it is the watch that notices when any gate stops reporting. A fleet-wide availability alarm treats absence of signal as a first-class failure: any required check that has stopped reporting on an active repository raises an alert as soon as its reporting window lapses, and shadow-rung gates are watched through their own verdict records. A gate whose death is detectable only by archaeology has no availability number worth trusting.

The seven-week outage of section 2 was not a precision failure; the reviewer's opinions were no worse that month than any other. It was an availability failure that nothing was positioned to see, because every monitor in the system watched for bad signals and none watched for missing ones. Absence reads as health precisely when no one has made absence a signal.

The remedy is structural. Across the fleet, an availability watch alarms on any required check that has stopped reporting on a repository with active pull-request traffic — a class of failure that otherwise surfaces only when someone wonders, weeks late, why a check last ran in a different month. Shadow gates, which by design produce no visible check, are watched from the other side: their verdict records are the heartbeat, and a gap in the records on an active repository is the alarm condition.

Sequencing matters here. The availability watch shipped before the reviewer it watches, because the order encodes the priority: a trust system that cannot detect its own death will eventually report an availability number that is fiction, and every metric downstream of a fictional availability number — precision samples, latency percentiles, promotion decisions — inherits the fiction. Silence is the failure mode that falsifies all the others.

---

## 7. Publishing the Method Before the Outcome

> **Answer:** The ladder is live: the availability watch runs fleet-wide and the review harness runs at L0 shadow on two production repositories. Published 2026-07-23 with the first organic verdicts days old and a seventy-second median; revised 2026-09-15 at eight weeks: ninety verdict rows, a verdict row for every eligible pull request, median latency 116 seconds, p95 298 seconds against the 180-second bar, audited precision 51 percent against the 85 percent bar — no promotion; harness v0.3.0 shipped the same day and the precision series restarts with it, 51 percent standing as the v0.2.0 record. Publishing in that state is deliberate: the architecture is on record before its numbers can flatter it, and the numbers are on record when they do not.

At publication, 2026-07-23, the honest state was early: shadow verdicts were accumulating toward the thirty the first audit series needed; the availability clock was inside its first four-week window; the first automatic needs-changes verdict had already landed in the log, unprompted, on an organic pull request. Nothing had been promoted, because nothing had yet earned promotion — the bars were published before the system could clear them, in that order, on purpose.

That ordering is the discipline. A trust system unveiled only after its metrics turned flattering is a marketing asset wearing instrumentation; the measurement cannot be audited by anyone who could not watch it accrue. This paper will be revised as rungs are climbed — the publication date and modification date are part of the record — and if a gate demotes, that revision will say so, because a ladder that only reports ascents is the 33 percent reviewer with a better dashboard.

*Revision, 2026-09-15 — eight weeks at L0.* The sample the first audit was waiting for arrived and the bars did not move: ninety verdict rows across the two repositories — on eighty-six pull requests, since a re-review writes its own row — a verdict row for every eligible pull request in the window, median latency 116 seconds, p95 298 seconds against the 180-second bar. The second weekly audit labeled every stage-2-verified finding on every needs-changes verdict since the first audit — the exact set an advisory gate would have surfaced and a soft gate would have blocked on — against independent ground truth: the diff at the reviewed commit, the source of every callee a finding named, vendor documentation, production logs, and where a regular expression was the claim, execution. This was a census rather than the ritual's five sampled verdicts — stricter, chosen because seven weeks of sample were owed.

**22 of 43 held, 51 percent**, against the 85 percent bar; the first audit, 2026-07-25, had measured 4 of 11 on the same series under an earlier prompt revision, so the pooled record stands at 26 of 54. Read by verdict rather than by finding, 5 of the 17 needs-changes verdicts carried no correct finding at all — the wrong-block rate a soft gate would have shown, 29 percent. Escape rate was not consulted for this audit: it lags by design, and its July record — six incidents, all past deterministic gates — predates every verdict here. Nothing promoted.

The 21 that failed sort into classes the harness can act on. The largest, five findings, was cross-file blindness: the verifier confirmed that a helper "can throw" or that a call "has no timeout" without reading the helper, which swallows every error, or the neighboring file, which carries the timeout. Six were preferences dressed as defects, and five were environments not shown to exist. Two were trade-offs the pull request itself acknowledged, verified anyway; one was a future-edit hypothetical; one named a real gap but the wrong consequence. The last was the sample's only high-severity finding, and it was wrong — it asserted that DynamoDB rejects a `REMOVE` clause placed before `SET`, which the service documentation contradicts in one sentence. A sample of one, consistent with the replay study's 38 percent band.

The 22 that held were not small. Three rounds of findings on one pull request were each fixed by the author in the next commit; a timeout finding the gate raised on 2026-07-29 materialized in production on 2026-08-16, exactly as described; an error-exposure finding was fixed by the next day's pull request. The gate finds real things. It would also block on things that are not there, and that ratio is the number that decides the rung.

The revision this paper committed to is also a confession. The first audit ran on 2026-07-25; the second ran on 2026-09-15. Seven weekly slots passed with no labels written. The verdict record accrued on its own, because the availability watch and the shadow-rung verdict writer need no human; the ritual did not, because it does. By the rules in section 3 a lapsed ritual promotes nothing and, at the bottom rung, demotes nothing; the seven weeks count toward availability, which the gate reported every day, and toward nothing else, since precision is measured on the labeled sample rather than on a calendar. But by the words in section 4, a dead ritual is itself a silent gate, and the paper that wrote those words does not get to exempt its own. 

What changes — three commitments, none shipped as this revision was written. The verify stage reads the source of every symbol a finding names before it rules, since 5 of the 21 misses — nearly a quarter — would have been rejected by one file read. A diff over a size published with the change resolves to a neutral verdict that counts as an availability miss, never a pass, so the latency tail is capped rather than hoped away. And the weekly audit gets a mechanical watch of its own, so that the next lapse raises an alarm instead of a paragraph — and once that watch is live, an unlabeled week is a silent-failure week under section 3's rule and demotes like one. Each lands here as a dated line, the way this one did.

*Shipped, 2026-09-15 — the watch.* The weekly audit now has a mechanical watch. The availability watch of section 6 gains a leg that reads the audit ledger every day, takes the newest label date across every row, whatever order they were written in, and fails the run with an alert once that date is more than ten days old: seven for the cadence, three of grace. A missing or unreadable ledger fails the same way, because a watch that cannot read its subject would report the ritual alive by omission. From this line on, an unlabeled week, past its three days of grace, is a silent-failure week under section 3.

*Shipped, 2026-09-15 — the callee read.* Harness v0.3.0, merged and pinned on both shadow repositories the same day. Before the verify stage rules on a finding, it receives the source of every symbol the finding names, within a per-finding size budget, resolved at the reviewed commit through the flagged file's imports and aliases and the repository's top-level definitions, each labeled with its path and line. What cannot be resolved is named as unresolved, so the verifier may not assume the callee's behavior in either direction, and a finding that the callee's source defeats is its own rejection class. On the two cases named above, the resolver reads the helper that swallows every error and follows the local alias to the neighboring file that carries the timeout. The precision series restarts with the version, since a changed verifier is a different gate; 51 percent stands as the v0.2.0 record.

*Shipped, 2026-09-15 — the cap.* The published size is 1,000 changed lines. A pull request over it receives a neutral verdict without a model call, and the verdict record counts it as an availability miss, never as coverage. The number comes from the ninety pull requests the verdict record holds as this line is written, four of them arrived the morning of the audit: seven are over it. Every verdict the model returned on those seven, six of six, ran past 180 seconds; the seventh had produced a neutral error and no latency at all. Under the cap those seven are availability misses — counted per pull request, the way the instrument counts it, about 92 percent against the ladder's 99 percent availability bar. That is the trade: a miss the record can see is better than a verdict that ran past the bar every time one was tried.

It also holds the number the cap does not fix. Below it, p95 is still 244 seconds, and most of that is the find stage writing out its findings on mid-size diffs. The find stage is the larger of the two in twelve of the fourteen rows under the cap and over 180 seconds, and five of them run past 180 on the find stage alone; the verify stage's serial calls account for the rest of each row. With verification free, the p95 would still be 184 seconds. The find stage's output time is the next change.

---

## 8. The Operating Doctrine

> **Answer:** The ladder is one instance of a general doctrine: model judgment proposes, deterministic checks block, and nothing non-deterministic gains authority it has not measured its way into. The same promotion pattern governs every probabilistic gate in the organization — a content-quality gate is climbing the same ladder in shadow today — and a short list of decisions never delegates to any gate at any rung: production deploys, money movement, external communications, secrets custody, and destructive operations.

The runtime half of this doctrine is the [validator-gated architecture](https://isimplifyme.com/blog/determinism-gap-validator-architecture) described across this series: deterministic validators wrapped around probabilistic components, covered as deployed-system reliability in [Layer 4](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering). The development-loop half is the ladder — the recognition that the loop *producing* the system contains probabilistic components too, and that they deserve the same treatment: bounded authority, published thresholds, automatic revocation.

The pattern is portable beyond code review. Any non-deterministic gate — a content-quality scorer, a security triager, a classification step in an operations pipeline — can run in shadow against real traffic, accrue a verdict record, face a weekly audit, and climb the same four rungs.

Some decisions never climb. The five human-only classes named above stay human at every rung — not because a gate could never be accurate enough, but because these are the decisions where a wrong block is an inconvenience and a wrong pass is an incident, and that asymmetry does not reduce to a precision percentage.

For a team standing this up, the order of operations is the architecture: first alarm on gate absence, so silence is impossible before anything has authority. Then shadow the reviewer and let the verdict record accrue. Then audit weekly, ten minutes, against bars published in advance. Promotion after that is arithmetic, not a meeting. For engineering teams that want a second set of eyes on their own gate inventory, [the validator gap audit](https://isimplifyme.com/services/validator-gap-audit) is where to start.

---

## Companion Papers

This paper is part of the iSimplifyMe reference-architecture series on LLM-native systems:

- [**Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock**](https://isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock) *— shipped.* Model isolation and compliance in a HIPAA workflow.
- [**Layer 3: Data + Retrieval**](https://isimplifyme.com/whitepapers/layer-3-data-retrieval) *— shipped.* Pipelines, permissioned retrieval, hybrid search, context engineering, memory, and feedback loops.
- [**Keeping AI Spend Flat: Caching and Model Routing**](https://isimplifyme.com/whitepapers/ai-spend-caching-and-model-routing) *— shipped.* The cost-and-latency half of Layer 4.
- [**Layer 4: Reliability Engineering for Regulated AI**](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) *— shipped.* The runtime-reliability companion to this paper: guardrails, atomic integrity, investigate-only audit, circuit breakers, retries, and quality gates.
- [**Layer 5: Multi-Tenant Business Integration**](https://isimplifyme.com/whitepapers/layer-5-business-integration) *— shipped.* Single-table multi-tenancy, domain routing, the unified lead pipeline, permissioned dashboards, and synchronized billing.
- ***The Trust Ladder: Supervised Autonomy for AI Code Review*** *— this paper.* The development-loop trust layer.

The companion [Labs dossier](https://isimplifyme.com/labs/trust-ladder) is the short-form canonical reference for the ladder itself.

---

## Conclusion

Supervised autonomy arrives when agents write nearly all of the code and a human manages by exception — and the bottleneck of that stage, as Cherny names it, is trust in the loop. The trap is granting authority ahead of evidence. The Trust Ladder is the refusal to do that, made mechanical: four rungs, four measured signals and a lagging fifth, a routed find-then-verify harness that fails neutral and loudly, an availability watch that makes silence impossible, and demotion that is automatic rather than negotiated. The first reviewer failed at 33 percent precision with seven silent weeks; its replacement stands in shadow at a measured 51 percent verified precision after eight weeks, against an 85 percent bar it has not cleared. Autonomy is not a feature you enable. It is a state a gate earns — and can lose.

---

## Notices

**Not professional advice.** This paper is an engineering reference for informational purposes. Development-process and governance decisions require review against your organization's own risk, compliance, and audit requirements.

**Implementation details vary.** The architecture here is a reference pattern drawn from production systems; specific thresholds, window lengths, audit cadences, and risk classes are tuned per organization and evolve over time. Operational specifics describe representative configurations, not guarantees.

**Measurements are point-in-time.** Precision, latency, and availability figures cited here are frozen observations from the systems described, at the time of writing; live numbers change as trailing windows accrue, and revisions to this paper will reflect material changes including demotions.

**Trademarks.** Claude and Claude Code are trademarks of Anthropic, PBC. AWS and Amazon Bedrock are trademarks of Amazon.com, Inc. or its affiliates. References, including to Boris Cherny's published framework, are descriptive and do not imply endorsement.

---

**About the author.** Joseph W. Elstner is the founder and principal architect of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific. iSimplifyMe is bootstrapped, deploys production AI on AWS Bedrock, and runs a multi-tenant orchestration platform across healthcare, legal, financial, and editorial verticals.

**Contact.** ai@isimplifyme.com — for engineering teams standing up trust systems around their own AI gates, we offer a validator-gap review at no cost.

**Cite this paper.** Elstner, J. (2026). *The Trust Ladder: Supervised Autonomy for AI Code Review.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/the-trust-ladder
