> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/layer-1-model-selection-and-routing](https://isimplifyme.com/whitepapers/layer-1-model-selection-and-routing)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-10-02 · Revised 2026-10-03 · License: [CC BY 4.0](LICENSE)

# Layer 1: The Model Is a Dependency With a Retirement Date

Frontier model selection and routing — why we don't train, why we route per task, and how the architecture stays model-version-independent. A reference architecture for the model layer of LLM-native systems on AWS Bedrock, with one fleet's move between two model generations as the worked case.

---

## Abstract

> **Answer:** Layer 1 is the model layer of an LLM-native systems architecture — the decision of which frontier model each task runs on, and the engineering that keeps that decision cheap to change. The model is a dependency with a retirement date: providers retire versions on their own schedule, and in Kim's 2026 study of 22,555 commits, an estimated 82% of migrations away from retired models were made after the shutdown date. The architecture that survives this treats the model as a configuration value behind a named role, routes each task to a model tier by a static table rather than a learned router, verifies every generation change on the system's own archived prompts before the flip, and watches the serving path from outside with a daily canary. It never trains a language model, because a trained model is the one dependency that cannot be swapped by configuration.

This is the first paper in the series by layer number and was written after Layers 3, 4 and 5, because the model layer is the thinnest layer by design. [Layer 3](https://isimplifyme.com/whitepapers/layer-3-data-retrieval) covers data and retrieval; [Layer 4](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) covers reliability and, with its [companion on caching and routing](https://isimplifyme.com/whitepapers/ai-spend-caching-and-model-routing), cost; [Layer 5](https://isimplifyme.com/whitepapers/layer-5-business-integration) covers business integration. Those layers are where most of the engineering is. This paper covers the one above them: how to choose a frontier model, how to route tasks across several, and — the part that gets skipped — how to make the choice survivable when the model goes away.

The intended reader is the person who will be asked, in a procurement review or a board meeting, "what happens when the model you built on is retired?" The answer should be a configuration change and a verification run, not a rewrite. This paper is how.

---

## 1. A Calendar the System Did Not Write

> **Answer:** A frontier model accessed through an API is a dependency the vendor can withdraw, and does, on notice periods that Kim's 2026 study puts at a year down to two weeks. The share of applications that migrate only after the shutdown tracks that notice: across the same study's 17,703 repositories, 13% on a one-year notice and 89% on notices of two to four months, with model identifiers hard-coded in 94% of the applications that migrated. The question for an architecture is not whether the model will change but what in the system has to change with it.

Every production system built on a commercial model API inherits a calendar it did not write. Providers retire model versions on their own schedule, and the notice is uneven. Hyungjin Lukas Kim's study of open-source applications, [*When the Model Retires*](https://arxiv.org/abs/2609.31288), puts the range from a year to two weeks. Across 22,555 commits in 17,703 repositories, an estimated 82% of migrations away from retired models were committed after the shutdown date — after the application had started failing — and the share tracks the notice almost exactly: 13% reactive on a one-year notice, 89% on notices of roughly two to four months. Each e-fold increase in notice — a notice period about 2.7 times longer — cut the odds of a post-shutdown migration by about three quarters. The same study found migration effort scaling with architecture: a median of 2 files and 6 added lines for a prompt-only application, 63 lines for an agent with tools, 155 for a retrieval system, and 693 lines across 14 files for a fine-tuned one.

Two of its findings matter more than the headline. First, a pre-existing abstraction layer (a gateway, a router library, a model map) was present in only 3% of migrations, and where it existed it "shrinks the fix but does not make it timely," in Kim's words: those migrations were the smallest non-trivial ones in the corpus, a median of 4 files and 20 lines, and 70% of them still landed after the shutdown. Second, the breakage was not only the obvious 404. Eight percent of genuine migrations were *silent* failures, where an exception handler had quietly converted the retired model's error into a fallback — cases that, in Kim's reading, hid outages for weeks; 7.5% of commits mention *parameter incompatibilities* — the replacement model rejecting a request parameter the old one accepted.

Our own fleet has moved its Sonnet-tier roles twice since July — from Sonnet 4.6 to Sonnet 5 beginning July 16, and from Sonnet 5 to Sonnet 5.5 between September 28 and October 1. Neither move was forced by a retirement, but each was a rehearsal for one, and the lesson was the same both times: the identifier is the easy part. What has to change with the model is the request shape around it, the ceilings sized for it, the prompts that quietly depended on it, and the evidence that the new one behaves. Everything below is about making those four things cheap.

---

## 2. Why We Do Not Train Language Models

> **Answer:** A fine-tuned model is the one dependency that cannot be swapped by configuration. When its base model is retired the adapter does not carry: portability across base-model releases decays toward zero with continued-pretraining distance, and the measured migration cost for fine-tuned applications is a median of 693 added lines against 6 for prompt-only. So the rule is to train only what cannot be routed. Every language task in the fleet runs on a routed frontier model; the only trained models are three narrow vision models for tasks no frontier model performs.

The case for fine-tuning a frontier model is usually made on quality or cost at a single point in time. The case against it is made over time, and time wins.

**The adapter is married to its base.** [UpgradeBench](https://arxiv.org/abs/2608.20918), a longitudinal benchmark across four consecutive releases of one open-weight family, found that directly copying a task adapter to a new base model works only while the new base is close to the old one: on checkpoints with known training lineage, retention fell from 0.88–0.99 after a short continued-pretraining run to zero after a long one. That is a measurement on open-weight lineages; applying it to fine-tunes behind a closed API, where the vendor retires the fine-tune outright, is an inference, and a conservative one. Every base-model release therefore forces a decision — retain, port, refresh, or retrain — and organizations keep paying for that decision on every release. Kim's migration data shows what the bill looks like when the base is retired outright: 14 files and 693 lines at the median, more than a hundred times the prompt-only cost.

**What we run instead.** The fleet runs on frontier models through AWS Bedrock inference profiles, routed per role (section 3), with no custom models on Bedrock and no language-model training job on record. The exception is deliberate and narrow. NexV, our dental platform, runs three task-specific vision models on SageMaker endpoints — tooth detection, smile inpainting, and tooth segmentation — because no frontier model we have tested performs those tasks to the standard the product needs, and because their inputs are radiographs and intra-oral photographs whose distribution does not shift when a language model is retired. Those models have their own lifecycle; they do not inherit the frontier vendor's calendar. The rule is not "never train." It is *train only what you cannot route*, and keep the trained surface small enough to count on one hand.

The healthcare reference architecture in this series ([*Private LLM Architecture for Mid-Market Healthcare*](https://isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock), section 2.3) describes the SageMaker side of that split. This paper is about the routed side, which is almost all of it.

---

## 3. Why We Route Per Task, and Why the Router Is a Table

> **Answer:** Routing per task means each named role in the system — chat, drafting, classification, narrative, triage — is bound to a model tier chosen for that task's precision requirement and volume, in a static table reviewed when the task changes. The alternative, a learned per-query router, recovers only a small share of the gain it is sold on: in a 2026 evaluation whose confidence intervals survive choosing the best router after the fact, the strongest deployable router captured 7.5–14.4% of the oracle opportunity, and in a four-router comparison three of the four emitted near-constant tier assignments. At fleet scale the deployable form of routing is the table.

Two companion papers make the case for routing itself, and this paper does not repeat them. [*Keeping AI Spend Flat*](https://isimplifyme.com/whitepapers/ai-spend-caching-and-model-routing) (sections 3 to 6) shows why matching the model to the task is the second cost lever after caching, and [*The Routing Table*](https://isimplifyme.com/whitepapers/the-routing-table) (sections 4, 5 and 9) shows, from more than forty thousand measured calls, that a model's capability tier is a *precision* control — the larger tier earns its price on the tasks where a wrong answer is expensive, not on volume. The same paper shows that the serving door is a configuration decision with behavioral consequences.

What this paper adds is the form the router should take. The research literature on routing is mostly about learned routers: [RouteLLM](https://arxiv.org/abs/2406.18665) trains a classifier on preference data to send each query to a strong or a weak model; [FrugalGPT](https://arxiv.org/abs/2305.05176) cascades models by confidence; [RouterBench](https://arxiv.org/abs/2403.12031) benchmarks the family. Two results from 2026 temper the promise. Shihab, Al Ahsan and Swaqeeb, in [*Opportunity Is Not Realizability*](https://arxiv.org/abs/2608.08265), separate what an oracle router could gain from what a deployable one does, and find the realizable share small: 7.5–14.4% of a certified 9.7–30.7 point gap. Kumar and Saminathan's [common-protocol evaluation of four open-source routers](https://arxiv.org/abs/2608.14641) found that three of them barely vary their tier assignment with the prompt at all, and that observed gains tracked the mix of tiers selected rather than any demonstrated per-task targeting.

The information a router needs — which tasks are precision-critical, which are volume, which need a tool only one door offers — is known at design time, per role, and it changes when the task changes, not per query. So the router is a table:

| Role | Tier | Why |
|---|---|---|
| Visitor-facing chat, lead drafting, weekly insights, prompt generation | Large | A wrong answer is visible or expensive; volume is bounded |
| Dashboard narratives, anomaly explanations, ticket execution, damage-check vision (a public photo-triage endpoint) | Small | Volume paths where the small tier's precision is sufficient |
| The base-knowledge citation probe | Small, pinned with its reason | The one citation path with real volume; a larger model has the identical blind spot at a higher price |
| Investigate-only operations agents | Large, thinking off | Tool loops need the precision, not the deliberation |

In the orchestration platform that is fifteen roles in one file, and the thirteen that run on Bedrock resolve to two model tiers. Several of the pins carry a written reason beside them in the source, because a pin without a reason is the first thing the next engineer "upgrades."

---

## 4. The Registry: Roles, Not Models

> **Answer:** The registry is one file in which every model-consuming path in the system is a named role, and each role resolves at access time in a fixed order: an explicit per-stage environment override, then the stage's application inference profile, then the code default. Call sites import a role, never an identifier. The deployed unit is the inference profile — same model, same US-bounded routing — which carries the role's tag into the bill. A model identifier change is an edit to this file and its contract test; what else has to move when the model behind the identifier changes is section 5.

The pattern is older than language models — it is dependency injection with a configuration precedence — but the specifics matter, because Kim's failure classes are specific ones.

**One file.** Before the registry existed, seventeen hard-coded model identifiers sat across thirteen files of the orchestration platform, flagged by an estate review in July. The registry replaced them with named roles. A call site asks for the lead-drafting role; it has no opinion about which model that is. Kim's remedy for the 94% hard-coding rate is exactly this — in his words, "an environment-variable or registry indirection would have made the majority of these migrations a one-line change" — and his data on why that is not sufficient on its own is the subject of section 7.

**Precedence.** Each role is an access-time getter that resolves in order: an explicit environment override for that role, then the stage's application inference profile from a compact map the deploy ships, then the code default. The override is the one-word rollback — set it to the previous model and the role follows with no code deploy, as a function-configuration update — and because the request shape of section 5 keys on the same override, the rolled-back role carries the previous generation's shape with it. The profile is the normal production path. The default is what runs locally and in tests, and it is pinned to the profile by a contract test so the two cannot drift.

**Identity is not the invoked identifier.** Inside a deployed function, the value a role resolves to is an application-inference-profile ARN — an opaque string that does not name a model. Any code that shapes a request *by model* — the thinking configuration, the output ceiling, the effort level — therefore reads a second accessor that returns the role's model identity (override or default, never the profile) and keys on that. Branching on the string shape of the invoked identifier is the bug this prevents; it would read every production role as "not the new generation."

**The deployed unit is the profile.** Every production Bedrock role invokes an application inference profile rather than a bare model identifier. The profile routes to the same model through the same US-bounded inference path — the cross-region profile bounded to us-east-1, us-east-2 and us-west-2 — and it carries tags (application, stage, role) that appear in the bill, so model spend is attributable per role without instrumenting the call sites. Residency is enforced at the same point: the registry's contract test rejects any Bedrock identifier that is not the US-bounded form, and pins the one Claude role that must stay first-party as the recorded exception (section 8).

What the registry buys is a small, inspectable blast radius. The question "what runs on model X?" is answered by reading one file. The question "what happens when model X is retired?" is answered by changing it — and then by sections 5 through 7, which are about everything the file does not contain.

---

## 5. Version Independence Is a Request-Shape Problem

> **Answer:** Changing a model identifier is one line. Staying independent of the model version is the work around that line: the request shape the new generation accepts, the output ceilings it needs, the latency profile it has, and the prompts it declines. In the fleet's move from one Sonnet generation to the next, the identifier was a single constant; the migration was nine pull requests across five repositories — eight for the move and one price-table correction the flip exposed — 62 files and 994 added lines, 436 of them in tests. Kim's taxonomy calls the first of these breakages parameter incompatibility; 7.5% of his migration commits mention one, and it would have been a production outage here.

Here is the worked case, dated so that it ages honestly. Between September 28 and October 1, 2026, every Sonnet-tier role in the fleet but three moved from Claude Sonnet 5 to Claude Sonnet 5.5 — on Bedrock for the orchestration platform, the content-strategy product, the public site's concierge and the dental platform's text roles; on the vendor's direct API for a consumer app's brief generator. The three that stayed are section 8. The identifier change was one constant in each registry. Four other things changed with it.

**The request shape changed by generation.** The new generation rejects the previous generation's thinking-off configuration — `thinking: {type: "disabled"}` — with a 400 error, and accepts a form, `{type: "between_tools"}`, that every other model in the fleet rejects. Two kinds of interactive path — the streaming chat loops and the operations agents' tool loop — pin thinking off, because they rebuild assistant turns without thinking blocks. A bare identifier flip would have returned a 400 on every chat turn and every investigation. The fix is structural: the thinking configuration is a function of the model *identity* (section 4), never a literal in a call site, so a role that moves generations carries its request shape with it. The new form was verified before the flip on all three invocation paths — single call, stream, and the conversation API — and through an application inference profile. This is the parameter-incompatibility class — the replacement rejecting what the original accepted — and it is why "just change the id" is a plan for an outage.

**Output length changed.** On the one long-form path measured, answers ran about 1.6 times longer. A strategy-report generator with a 4,096-token ceiling truncated 25 of 29 reports on the new generation, against one to five on the old; its ceiling went to 10,000. Wall time was flat, because the new generation also generates about 1.6 times faster.

**Latency changed.** With thinking left at its default, a dental assistant endpoint's first token moved from 1.6 seconds to 2.7 in the pre-flip probes; at a lower effort setting it came in at 0.9 — a setting the old generation accepted as well, so the change was safe to make before the flip.

**The decline surface changed.** One class of system prompt — one that asks the model to write out its reasoning after each tool result — is declined outright by the new generation, every time, with a refusal stop reason and no content. Casey, Roberts, Sim and Beaver saw the same shape from the other direction in [*When Your LLM Reaches End-of-Life*](https://arxiv.org/abs/2604.27082): switching a candidate model's reasoning mode on raised its style violations from 2.1% to 7.5%, and two other candidates were eliminated for failing the output schema before any quality question was asked. Behavior that was never specified — the prompt's implicit assumption that "think out loud" is allowed — is where the regression lives, which is Yang et al.'s finding in [*What Prompts Don't Say*](https://arxiv.org/abs/2505.13360): under-specified prompts are twice as likely to regress across model or prompt changes.

**One rule from the deploy itself.** In our deploy tooling, a Lambda deploy applies the new environment before the new code — seconds apart on a good day, indefinitely if the code step fails. A build that still sends the old generation's request shape, handed the new generation's profile by its environment, fails every call for the gap. So the role-to-profile map is versioned by *name*: the old build does not read the new map, finds no profile, and falls back to its own defaults, self-consistent until its code is replaced. A rollback deploy is the mirror image. The versioning costs one suffix and a test that pins it.

The anatomy of the largest of the nine pull requests says what version independence costs. In the orchestration platform: the registry file, 76 lines added and 43 removed; its tests, 177 of the pull request's 311 added lines, spread across six files; the chat route, 27 lines; the deploy configuration, 31. In the content-strategy product, seven call sites changed two lines each, which is what a registry is for. The identifier was cheap. The contract around it — the request shape, the ceilings, the profile names, and the tests that pin all three — was the migration.

---

## 6. Verify on Your Own Prompts Before the Flip

> **Answer:** Before a generation change reaches a user, the system's own archived prompts are replayed against the new model through the deployed profile: real visitor turns, real agent sessions on every job kind, real drafts and digests. The replay is a contract check — shape, declines, ceilings, latency — not a quality evaluation, and the harness it runs in, replay plus a small set of probes, surfaced all four behavior changes of section 5 before any user did. Aggregate benchmarks do not do this job: in a 2026 item-level study, migration edges with aggregate gains of up to 7.3 points contained up to 8.3% reliably regressed items.

A vendor's benchmark delta is the wrong instrument for a migration decision. Xu and Wu, in [*What Aggregate Scores Miss*](https://arxiv.org/abs/2608.17719), queried 900 benchmark items fifty times each across three pairwise upgrades within one commercial model line and found reliable improvements and reliable regressions coexisting in every cell: edges with aggregate gains of up to 7.3 points held up to 8.3% reliably regressed items, and edges with aggregate losses held up to 10.7% reliably improved ones. The aggregate compresses a bidirectional change into a sign. The only prompts whose regressions matter to a system are its own.

**The replay.** Before the October flip, the harness replayed the fleet's archived production prompts against the new generation through the deployed profiles: 220 archived visitor turns across 25 tenants of the public concierge; 16 operations-agent sessions covering every job kind the three agents investigate; 45 lead-response drafts; 16 weekly insight digests; and the prompt-generation and content-gap paths. Zero declines across all of them — a bound on the prompts measured, not a guarantee. The four changes of section 5 — the request shape, the truncation, the latency, and the declined prompt class, which a probe prompt in the same harness tripped and no production prompt did — each surfaced before the flip, with a fix landing before it rather than a ticket after it.

**What it is and is not.** The replay checks the contract: does the request succeed, does the output parse, does it fit the ceiling, is the latency within budget, is anything declined. It does not score quality, because scoring quality properly is a different and larger instrument. Casey et al.'s six-step framework — candidate selection, schema validation, correctness against the incumbent with human-calibrated automatic metrics, latency and refusal filters, style, coverage — is that instrument, run on a question-answering service of 5.3 million interactions a month, and it is the right next rung for a role whose quality has to be defended numerically. The contract check is the floor every role gets; the quality evaluation is the ceiling the few roles that need it earn. [RETAIN](https://arxiv.org/abs/2409.03928), from 2024, is the tooling shape for the prompt-side half of that work.

The archived prompts are the asset here, and they are already paid for: a system that keeps its production prompts — under the privacy handling the data layer already imposes — has its migration test set already written, by its users, in their words.

---

## 7. Watch the Serving Path From Outside

> **Answer:** A registry makes a model change cheap; it does not tell you when one has happened. Hosted models change behind unchanged identifiers, and the current frontier family returns undated identifiers in its responses, so a build roll is invisible from inside the call. The control is a daily canary: a fixed battery of synthetic probes across three serving doors and three models, scored against frozen baselines. In its first 65 runs — 9,360 calls — it recorded zero semantic failures, caught one served-identifier change on one door within a day, and recorded four days on which calls failed on one door while the other two served.

Kim's data closes the loop on section 4: an abstraction layer shrank the fix but did not make it timely, and the factors that did predict a timely migration were the provider's notice policy, identifier externalization, and *retirement watches* — something watching the dependency from outside. Chishti, Oyinloye and Li frame the broader problem in [*Test Before You Deploy*](https://arxiv.org/abs/2604.27789): hosted models evolve through provider-side updates with no version change, so a deployer needs production contracts and compatibility gates of its own, because the vendor's version string is not a contract. Our own measurements add a sharper reason: the current frontier family returns undated model identifiers in its responses on every door we measured ([*The Same Model, Three Doors*](https://isimplifyme.com/whitepapers/the-same-model-three-doors)), so a build roll behind the same identifier is invisible from the response.

**The canary.** Every day at a fixed hour, a battery of 144 calls runs across three serving doors — Amazon Bedrock, Claude Platform on AWS, and the vendor's direct API — and three models. The probes are the frozen fixtures of the determinism studies: an extraction with one correct answer, a classification with one correct label, and a structured-JSON task scored for parse, semantics, and formatting rate. Baselines were computed from the three-door study's confirmatory records — 13,950 calls, the one study that measured all three doors — and frozen on July 30, 2026. Each run scores against them: a wrong extraction or label, or a changed served-model identifier, is red; a formatting rate outside its band or a byte-variant never seen in the corpus is yellow, logged and batched into the daily post rather than paged. The baselines and every run's log are committed to the [public harness repository](https://github.com/iSimplifyMe/determinism-harness/tree/main/canary).

**The record, July 30 to October 2, 2026.** Sixty-five runs, 9,360 calls. Zero semantic failures: no probe ever returned a wrong extraction or a wrong label on any door. Twenty-two green days. Thirty-six yellow days, all formatting-rate drift or a novel byte-variant with correct semantics — the serialization cosmetics the series' reanalysis predicted would dominate. Three red days, August 29 to 31, on one finding: the Bedrock door began echoing the small model's identifier in a different form, the full foundation-model id instead of the short name, with every semantic probe still green. The canary caught it on the first day — a red posts to the operations channel when the run finishes — and it was decoded as a served-identifier form change, not a model change, and accepted as a new baseline on the third.

Four more red days were transport. On two of them, September 29 and October 1, every one of a single door's 48 calls failed while the other two served normally — once a provider incident, once a fault on our side of the call that the canary scores the same way; on the other two, August 19 and September 10, part of one door's battery failed and the rest of the day's calls served. The fleet does not route around a failed door — each role is bound to one — so the record argues for keeping a second door measured, not for failover: if a door goes dark for long, moving a role to a door the canary has been scoring daily is a tested change rather than an untested one.

**What it does not do.** Three fixed semantic tasks are a tripwire, not an evaluation. The canary can say that extraction, classification, and a structured task on fixed inputs still behave; it cannot say that a model's open-ended generation has not drifted, and it does not try. Its job is the one a registry cannot do: notice, from outside and within a day, that the dependency moved.

---

## 8. What Did Not Move, and Why

> **Answer:** A fleet-wide model flip has deliberate non-moves, each recorded next to the role it protects. In the October flip three roles stayed where they were. A measurement instrument stays on the generation it was calibrated on; it is also the one role that must use the vendor's direct API, because the web-search tool it needs is not offered on the Bedrock door. Two frontier-model vision roles stay until they have been compared on real images. And the whole fleet waited three days for a US-bounded inference profile rather than use the global one, because residency outranks the upgrade.

Four things did not move in October, each for a recorded reason: three roles, and — for three days — the fleet. A registry that can move everything at once needs a way to say what must not move, and why.

**The instrument.** The weekly citation check asks a browsing model whether it names a client in answer to a search-style question. The model writes the search query, and the query is the measurement: on one of two production prompts probed, two models given the same question wrote different queries and only two of their cited domains overlapped; on the other they wrote the identical query and still cited different pages. Moving that role changes the series, so it stays pinned in the registry on the generation it was calibrated on, with the reason beside it, until the series is deliberately re-based. A measurement instrument does not follow the fleet. The same role is also the registry's one exception to the US-bounded identifier rule: it must call the vendor's direct API because the web-search tool it depends on is not offered on Bedrock, and the contract test allows exactly that one.

**The vision roles.** Two frontier-model roles that read radiographs and intra-oral photographs — the dental platform's language-model vision paths, not the trained SageMaker models of section 2 — stayed where they were, because no comparison on real images had been run. The text roles beside them moved. "Not yet compared" is a legitimate state for a role to be in, and the registry makes it a one-line state rather than a fork.

**Residency.** On launch day, the new generation was available on Bedrock only through a global inference profile that routes worldwide. The fleet's rule is US-bounded inference for regulated tenants, so the flip waited — three days, with a watcher paging when the US-bounded profile appeared — rather than take the global path. Residency is a routing constraint, and it outranks the upgrade.

Each exception is a line in the registry with its reason beside it. The file that moved six roles in October — and renamed all thirteen of its profiles for their model — held its one exception on the same page; the dental platform's registry held the other two. A reviewer can read each without leaving the file.

---

## 9. Limitations

> **Answer:** This paper reports one fleet, one model family, and one cloud, with a single generation hop as its worked case. The argument against training language models rests on outside measurement and an inventory, not on a trained-model arm of our own. The pre-flip replay is a contract check, not a quality evaluation. The canary's semantic probes are three fixed tasks on synthetic fixtures. The outside studies are cited for their published figures and none has been replicated here.

What a careful reader should carry out of this paper:

1. **Scope.** One orchestration fleet on AWS Bedrock, one vendor's model family, three serving doors. The registry pattern transfers; the specific behavior changes in section 5 belong to one generation pair and will not repeat exactly. Nothing here covers a move between vendors, where the request shape changes again; Kim found only 8% of migrations switched provider.
2. **The worked case is one hop.** A retirement forced by a vendor, with a hard deadline, would add pressure the October flip did not have. Kim's data says that pressure is where most teams fail; this paper's claim is only that the architecture removes the technical reasons to.
3. **No trained-model arm.** The fleet has never fine-tuned a language model, so the comparison in section 2 is between outside measurements of fine-tuned migration cost and our own inventory. It is an argument from published measurement and an inventory.
4. **The replay is not an evaluation.** It verifies the contract. A role whose quality must be defended numerically needs the larger instrument section 6 points to.
5. **The canary is a tripwire.** Three semantic probes on frozen fixtures detect gross change on fixed tasks, not drift in open generation. Its nine-week record is short, and the one change it caught was a served-identifier form, not a model roll.
6. **Routing is argued from others' measurement.** The two 2026 router evaluations are cited as published; we have not run a learned router against the table.

---

## Companion Papers

This is a reference-architecture paper in a series across the layers of an LLM-native system:

- ***Layer 1: The Model Is a Dependency With a Retirement Date*** *— this paper.* Frontier model selection and routing: why we don't train, why we route per task, and how the architecture stays model-version-independent.
- [**Layer 2: The Detector Sets the Precision**](https://isimplifyme.com/whitepapers/layer-2-investigate-only-agents) *— shipped.* Investigate-only agents: the Sentinel pattern in detail; typed agent registry; SQS-driven generic runner; conditional-write locks for race protection; cost ceilings at the agent level; what five months of production tickets showed about precision.
- [**Layer 3: Data + Retrieval**](https://isimplifyme.com/whitepapers/layer-3-data-retrieval) *— shipped.* Pipelines, permissioned retrieval, hybrid search, context engineering, memory, and feedback loops.
- [**Layer 4: Reliability Engineering for Regulated AI**](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) *— shipped.* Guardrails, atomic integrity, investigate-only audit, circuit breakers, retries, and quality gates.
- [**Keeping AI Spend Flat: Caching and Model Routing**](https://isimplifyme.com/whitepapers/ai-spend-caching-and-model-routing) *— shipped.* The cost-and-latency half of Layer 4, and the first statement of per-task routing.
- [**Layer 5: Multi-Tenant Business Integration**](https://isimplifyme.com/whitepapers/layer-5-business-integration) *— shipped.* Single-table multi-tenancy, domain routing, the unified lead pipeline, permissioned dashboards, and synchronized billing.
- [**The Routing Table**](https://isimplifyme.com/whitepapers/the-routing-table) *— shipped.* The measured doctrine behind section 3: tier as a precision control, the door as a configuration decision.
- [**Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock**](https://isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock) *— shipped.* The SageMaker side of the train-only-what-you-cannot-route split.

Each paper stands alone; together they map the full stack of an LLM-native system in production.

---

## Conclusion

The model layer is the thinnest layer in the stack and the one most teams lean on hardest. A frontier model is a dependency with a retirement date, and the measured record says most applications meet that date after it has passed, with the identifier hard-coded and a fallback hiding the failure. The architecture that survives is not complicated, and this paper has described it section by section. In the fleet it describes, a generation change is an edit to one file, a verification run, and a watch — and no language model is ever trained, because the only dependency you cannot reconfigure is the one you built yourself.

---

## Notices

**Not legal, compliance, or financial advice.** This paper is for informational purposes only. Architectural decisions in regulated workflows require qualified counsel and a formal review.

**Dated observations.** The worked case in section 5 and the canary record in section 7 describe specific model generations over specific spans. Model behavior, availability, and serving-path characteristics change; the pattern is the claim, not the numbers for any one generation.

**Capabilities change.** AWS Bedrock service capabilities, model availability, inference-profile coverage, and the surrounding tooling evolve continuously; verify current state before implementation.

**Outside studies.** Figures attributed to external papers are quoted from the cited publications and have not been independently replicated by iSimplifyMe.

**Trademarks.** AWS, Amazon Bedrock, and Amazon SageMaker are trademarks of Amazon.com, Inc. or its affiliates. Claude is a trademark of Anthropic, PBC. References are descriptive and do not imply endorsement.

---

**About the author.** Joseph W. Elstner is the founder and principal architect of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific. iSimplifyMe is bootstrapped, deploys production AI on AWS Bedrock, and runs a multi-tenant orchestration platform across healthcare, legal, financial, and editorial verticals.

**Contact.** ai@isimplifyme.com — for engineering teams that need to know what happens to a production AI system when its model is retired, we offer a model-layer review at no cost.

**Cite this paper.** Elstner, J. (2026). *Layer 1: The Model Is a Dependency With a Retirement Date.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/layer-1-model-selection-and-routing
