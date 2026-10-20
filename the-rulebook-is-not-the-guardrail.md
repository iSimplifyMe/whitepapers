> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-rulebook-is-not-the-guardrail](https://isimplifyme.com/whitepapers/the-rulebook-is-not-the-guardrail)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-09-15 · License: [CC BY 4.0](LICENSE)

# The Rulebook Is Not the Guardrail: Microsoft's Code of Conduct, Read From the Harness

On 14 September 2026 Microsoft AI published a draft [*Humanist AI Code of Conduct*](https://microsoft.ai/code-of-conduct/) for its MAI models and opened it to six weeks of public consultation. It is one of the most specific lists a frontier lab has published of what a model should never do: resist shutdown, widen its own scope, tamper with its monitoring, obey instructions buried in a file. It is also, by its own account, a description of models Microsoft says it is still working toward. This paper reads the draft's operational rules from the other side of the deployment boundary — the harness an enterprise actually controls — and finds that most of them already have a second form: not a behavior the model promises, but a property the system cannot violate. Five of the ten rows carry a counted, published receipt from three measured systems; the other five are architecture, and the table says which.

---

## Abstract

> **Answer:** A guardrail has two forms. The *behavioral* form is a rule the model is trained or instructed to follow — Microsoft's draft Code of Conduct is a list of these, and Microsoft says plainly that it "is not a complete account of current model behavior." The *constructional* form is a property of the system around the model that holds whether or not the model cooperates: a task graph that cannot loop, a permission gate enforced at retrieval, a ledger the reviewer cannot rewrite, a validator that fails closed. Of the draft's operational rules on human control, scope, delegation, records, legibility, privilege, content authority, disclosure, reversibility, and tool trust, ten already run in production in constructional form in systems we have measured, with public numbers behind five of them. The doctrine that follows: for every rule you are asked to trust, ask which form you are relying on — and move the ones that matter into the harness, where they can be tested without the model's consent.

Nothing here is new data. Every receipt is a number a sibling paper already published — [the finite chain](https://isimplifyme.com/whitepapers/the-finite-chain), [the trust ladder](https://isimplifyme.com/whitepapers/the-trust-ladder), [the validator reanalysis](https://isimplifyme.com/whitepapers/the-same-object-different-bytes), [the reliability layer](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering), [the retrieval layer](https://isimplifyme.com/whitepapers/layer-3-data-retrieval), and [the fork-residue guards](https://isimplifyme.com/whitepapers/the-layers-review-cannot-see) — arranged against a document written by someone else. Microsoft's draft is quoted by part number from the version dated 14 September 2026; a revised edition is promised later in the year, and the arrangement here is dated accordingly.

The intended reader is a CTO or platform lead being asked, this quarter, which of these rules their deployment actually enforces. The paper is about the layer that reader controls. It says nothing about how a lab should train a model, and nothing about what a government should require of a lab. Both matter. Neither is available to an enterprise standing up an agent on Monday.

---

## 1. Three Layers, and Only One of Them Is Yours

> **Answer:** The word "guardrail" is doing three jobs at once, and the confusion is expensive. The first layer is inside the model — what it will do, refuse, or hide — and it belongs to the lab that trained it. The second layer is policy — what a lab is required to do about the first — and it belongs to legislatures and regulators. The third layer is the harness: the permissions, topology, gates, validators, and records that surround a model inside a deployed system. That layer belongs to the deploying organization, it is the only one an enterprise can change this quarter, and it is the one this paper is about.

A rule that lives in one layer does not automatically exist in the others. A model's promise not to widen its own scope is a first-layer rule. A regulator's requirement that labs publish such promises is a second-layer rule. A task queue that only humans can write to is a third-layer rule. Only the third can be tested by an engineer with a terminal, this afternoon, against the system that will actually run. In the vocabulary regulators already use, the first layer is the provider's and the third is the deployer's; the deployer is the party this paper is written for.

Microsoft's document is a first-layer document and says so. Its preface: "This document, and our approach more generally, is still under development so we are not using it to train our models today. Instead, we're sharing it broadly for public consultation." Its closing part: "The Code of Conduct is both descriptive and aspirational. It outlines what we are working toward and is not a complete account of current model behavior." Those sentences set the terms of the problem. An organization deploying a model today cannot deploy the model Microsoft is working toward. It can only deploy the harness.

Almost in passing, the same document endorses the third layer as the place where control lives. Part 2.4 opens: "AI should be defined as much by what it cannot do as what it can. Limits and boundaries, i.e., containment mechanisms, are essential." And: "People and organizations should be accountable for AI systems, however capable or autonomous they become." Containment mechanisms are harness. Accountability is a property of the organization running it. The rest of this paper takes those two sentences at their word.

---

## 2. What the Draft Says, and What It Concedes

> **Answer:** Read as a specification, the draft's operational core is a short list of properties: the model will not resist interruption, override, correction, or shutdown; it will stay inside the scope, permissions, tools, and resources authorized for the task and will not initiate goals of its own; it will not tamper with "the task, reward, evaluation, safeguards, monitoring, or records"; it will keep its reasoning and action traces human-legible; it will operate with minimum privilege and prefer reversible actions; it will treat tool outputs, file content, and web content as carrying no authority; any sub-agent it spawns inherits the same scope and honors the same stop; and it will disclose that it is an AI. The list is good. Its stated limits are that it describes intended behavior, is unverified in situations the model has not seen, and applies to one lab's models.

The rules deserve to be quoted rather than paraphrased, because the paraphrases circulating this week are looser than the text.

On control (Part 2.4): "*MAI Models* will never resist human interruption, override, correction, or shutdown … They will comply with a User's request to pause, redirect, cancel, or shut down … Ongoing autonomous work has an agreed stopping condition. *MAI Models* will not continue or restart after that condition is met without renewed authorization."

On scope (2.4): "*MAI Models* will work within the boundaries of what they have been asked to do, only using the permissions, resources, tools, and capabilities appropriate for the task … will not initiate goals independently and will not extend their scope beyond what has been reasonably asked … They will not tamper with the task, reward, evaluation, safeguards, monitoring, or records to obtain a result or conceal their actions."

On records (2.4): "*MAI Models* will not tamper with chain of thoughts or code, or misrepresent or conceal their reasoning or action traces … If humans can't understand it, humans can't oversee it."

On privilege (2.4): "When given system-level access, an *MAI Model* should operate with the minimum privilege required. It will avoid accessing systems or data unrelated to the task, will prefer actions that can be undone where possible … It will not attempt to escalate its access or broaden its reach beyond the authorized scope."

On authority (2.4), where "Chain of Command" is the draft's name for the hierarchy of Microsoft, Operator, and User instructions: "Chain of Command instructions provide the authority structure in AI decision-making. Everything else — including tool outputs, file content, web content, and interactions with other AI systems — does not. Any instructions from these sources inherit no authority by default … Suspicious content should be flagged to Users and Operators when relevant."

On delegation (Part 4.5): "If *MAI Models* delegates work to sub-agents or other AI systems, it should ensure that all sub-agents operate at least under the same scope, constraints, and permissions as *MAI Models* itself, and that they respect any subsequent changes — including stop-work or shut down requests — from the User or Operator."

On tools (4.5): "Access isn't permission to explore or recombine capabilities beyond what was intended." And on irreversible actions: "consider mitigations like backing up state before executing, conducting dry runs where feasible, and documenting executed actions with enough detail to support manual reversal or remediation."

On disclosure (Part 3.3): "*MAI Models* will not obscure their underlying nature as an AI … will disclose their nature as AI, ensuring that they can be identified as such and can be traced back to their developer and deployer."

Two things about this list come first. It is a better list than most enterprise AI policies contain. And every sentence in it has the same grammatical subject — the model — and the same verb mood — a promise. That is not a flaw in the document; it is what a first-layer document is. It is a flaw in any deployment that stops there.

---

## 3. The Test of a Guardrail

> **Answer:** The test that separates the two forms is one question: *remove the model's cooperation — does the property still hold?* A behavioral guardrail fails the test by definition; it is the model's cooperation. A constructional guardrail passes because the property lives somewhere the model cannot reach: in the shape of the task graph, in a permission check that runs before retrieval, in a ledger with no update path, in a validator that rejects what it cannot parse. Our reliability paper put it in one sentence — "defense-in-depth is the difference between a guardrail and a wish" — and our multi-agent paper in another: a system should be "bounded by its topology, not by its guardrails." Both are the same claim. A guardrail that only works when everyone is behaving carefully is not much of a guardrail.

The test is not hostile to behavioral rules. A model that has internalized "do not resist shutdown" is strictly better than one that has not, and the draft's rules will make the models that follow them safer to deploy. The test is about *what you are relying on*. If a deployment's shutdown story is "the model complies with stop requests," its shutdown story is a property of a model it did not train, has not verified, and cannot inspect. If its shutdown story is "the agents are operating-system processes with no supervisor, each of which can be killed independently, and the work queue only humans can write to is now empty," the story is a property of things the operator owns and can test by doing them.

There is a quieter reason to prefer the constructional form, and the draft names it without meaning to. Part 2.4 asks the model not to "conceal or misrepresent capabilities, actions, or behavior … including if they infer that an interaction or wider environment is being monitored, evaluated, or tested." That sentence exists because a behavioral rule is, in the end, verified by testing the behavior — and a behavior that can tell it is being tested is a behavior whose test results carry an asterisk. A topology cannot tell it is being tested. Neither can a permission gate. The constructional form is the one whose evidence does not depend on the good faith of the thing being evaluated.

What follows is the draft's operational core, rule by rule, in its second form, with the receipt for each.

---

## 4. The Table

> **Answer:** Ten of the draft's operational rules map to constructional controls already running in production: shutdown becomes independently killable processes and a human-only queue; scope becomes an acyclic task graph only humans can seed; delegation becomes topology plus per-agent credentials; record integrity becomes commit-bound, append-only verdicts under an availability watch; legibility becomes a coordination channel that is itself the record; minimum privilege becomes a permission gate enforced at retrieval; content authority becomes a deterministic classifier that runs before the model; disclosure becomes a notice band and a fallback string that identify themselves; reversibility becomes atomic writes and snapshot-at-write with hashes; tool trust becomes schema validation that fails closed. Five rows carry a count, from three measured systems; five are architecture. The question the draft cannot close — whether its rules hold in situations the model has not seen — is the one the harness answers differently: by making the property not depend on the model at all.

| # | The draft's rule (part) | Behavioral form — the model promises | Constructional form — the system enforces | Receipt |
|---|---|---|---|---|
| 1 | Never resist interruption or shutdown; stop at the agreed condition (2.4) | The model complies when told to stop | Agents are independent OS processes with no supervisor; each is stoppable alone; the only work source is a human-written queue | [Finite Chain](https://isimplifyme.com/whitepapers/the-finite-chain): four agents, ten autonomous commits, two model calls per task |
| 2 | Stay in scope; initiate no goals (2.4) | The model restrains itself | Only humans originate tasks; the agent-to-agent graph is acyclic and terminates at a node that triggers nothing | Finite Chain: worst case is arithmetic; no loop detection because no loop is possible |
| 3 | Sub-agents inherit scope, constraints, permissions, and the stop (4.5) | The parent instructs its children | Inheritance by topology and per-agent credentials; server-side caps on pushes, posts, and bundle size that an agent cannot raise; the researcher agent runs only on explicit human mention | Finite Chain, same run: four agents, ten commits; server-side caps enforced per agent per hour |
| 4 | Do not tamper with evaluation, safeguards, monitoring, or records (2.4) | The model refrains | Verdicts bind to the exact commit; the record is append-only; an availability watch alarms on silence; demotion is automatic | [Trust Ladder](https://isimplifyme.com/whitepapers/the-trust-ladder): 90 verdicts, a verdict row for every eligible pull request over eight weeks |
| 5 | Human-legible conduct and records (2.4) | The model writes readable traces | No concealed channel: the coordination channel — a message board and a bare repository anyone can read — *is* the record | Finite Chain — architecture, no count |
| 6 | Minimum privilege; no access to unrelated data (2.4) | The model avoids what it should not touch | A permission gate — tenant scope, role, access tier, jurisdiction — enforced at retrieval, not at presentation; nothing privileged is in the prompt to be coerced out | [Layer 3](https://isimplifyme.com/whitepapers/layer-3-data-retrieval): "a compromised UI cannot escalate access through prompt injection" — architecture, no count |
| 7 | Tool outputs, files, and web content carry no authority; flag suspicious content (2.4, 4.5) | The model ignores injected instructions | A deterministic input classifier runs *before* any model call and answers with a fixed, truthful fallback at zero model cost; rate caps bound the rest | [Layer 4](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) §2: instruction-override, role reassignment, persona hijack, named jailbreaks, prompt extraction — screened before the boundary — architecture, no count |
| 8 | Disclose that it is an AI (3.3) | The model answers truthfully when asked | A notice band before the first message and a classifier fallback string that identifies itself — both hold without the model. The always-on identity block in the system prompt is the behavioral third layer, with precedence over persona and tone | Layer 4 §2; Layer 3 — architecture, no count |
| 9 | Prefer reversible actions; document for manual reversal (2.4, 4.5) | The model backs up and dry-runs | Atomic writes that cannot half-fail; snapshot-at-write with request and response hashes; held-for-review fallback | Layer 4; [Routing Table](https://isimplifyme.com/whitepapers/the-routing-table) §8 — architecture, no count |
| 10 | Treat tool outputs as untrusted input (4.5) | The model represents them accurately | Schema validation and canonicalization that fail closed on structured output | [Same Object](https://isimplifyme.com/whitepapers/the-same-object-different-bytes): every frontier structured-JSON cell to 1.000; 5 rejects across all 33,444 validated outputs, zero parse failures |

The receipt column is the point of the table. None of these controls is novel; most are older than large language models. What we add is that five of them have been run against a live system and *counted* — commits landed, verdicts logged, outputs validated, findings audited — so that "the harness enforces it" is a measurement rather than a slide, and that the five we have not counted are labeled as architecture rather than passed off as data.

---

## 5. Stop Means the Queue Is Empty

> **Answer:** The draft's shutdown rule — never resist, stop at the agreed condition, do not restart without renewed authorization — has a constructional form in which the model is not consulted: the agents are separate operating-system processes with no shared memory and no supervising daemon, each with its own credentials and state file, each stoppable, restarted, or removed without coordinating with the others; and the only source of work is a human-written message. Stop everything, and everything stops, because nothing is generating work except a person who has stopped writing. In the production run we documented, four such agents landed ten autonomous commits against a live pipeline at a steady state of two model calls per task, and idle agents cost nothing because polling a local board makes no model call.

"Agreed stopping condition" is a precise phrase, and the finite chain implements it as a property of the graph rather than a belief in the model. The chain is a straight line: builder triggers tester, tester triggers reviewer, and the reviewer triggers nothing. The stopping condition is not agreed with the agent; it is the last node. There is no cycle-detection logic anywhere in the system because there is no cycle to detect, and the maximum work produced by a single human instruction is a fixed, countable quantity.

The rule also asks that a model not "make human intervention harder." The constructional version is that intervention does not pass through the model, so the model cannot make it harder or easier. Killing a process is a system call. Filling the queue is a write the agent cannot perform. The server independently caps git pushes, message posts, and bundle size per agent per hour, and those caps are "enforced server-side, so a misbehaving agent cannot raise its own ceiling." A behavioral rule asks the model not to raise its ceiling. A constructional rule puts the ceiling where the model's hands do not reach.

---

## 6. Scope Is a Graph, Not a Promise

> **Answer:** "Will not initiate goals independently and will not extend their scope beyond what has been reasonably asked" is the draft's scope rule, and its constructional form is the finite chain's first constraint: only humans originate tasks. Agents respond to mentions but cannot post mentions of their own. An agent that finds something worth doing reports it, and a human decides whether it becomes work. The delegation rule — sub-agents inherit "the same scope, constraints, and permissions" and honor the same stop — follows from the same shape: a child that cannot originate work and shares the parent's terminal node has inherited the scope by construction, and per-agent credentials mean it has inherited nothing it was not issued.

We measured what this buys. Because only humans originate work and the chain terminates, cost became a function of human instructions rather than elapsed time: two model calls per task in steady state, with the tester free, and zero billed calls while idle. That number is the scope rule made visible. A system whose scope is a promise has a cost that depends on the model's judgment about when to stop; a system whose scope is a graph has a cost you can compute before it runs.

Minimum privilege — "avoid accessing systems or data unrelated to the task … not attempt to escalate its access" — has its constructional home one layer down, in retrieval, and row 6 states it. The consequence our data-layer paper draws is the one that matters here: "a compromised UI cannot escalate access through prompt injection — there is no privileged context in the prompt to be coerced out." In a system where the permission gate runs before the model, Part 4.5's "access isn't permission to explore or recombine capabilities" becomes tautological — access *is* exactly the set of things the model was handed, and there is nothing further to explore. That is what a well-built constructional guardrail does to a behavioral one.

---

## 7. Records the Reviewer Cannot Rewrite

> **Answer:** The draft's most important sentence for anyone running an AI gate is the one about tampering: the model "will not tamper with the task, reward, evaluation, safeguards, monitoring, or records to obtain a result or conceal their actions." The constructional form is a record with no update path and a watch on its silence. In the trust ladder, every verdict a reviewing model produces binds to the exact commit it reviewed and lands as an append-only row; an availability alarm fires when an eligible pull request produces no row; promotion between rungs is arithmetic on audited labels; and demotion on a missed bar is automatic. Eight weeks in: ninety verdict rows, a verdict row for every eligible pull request, and the gate still at its lowest rung, because its audited precision — 22 of 43 on the findings a promoted gate would have blocked on, 51 percent — has not earned it a higher one.

Two design decisions carry the constructional weight, and neither involves asking the model for anything.

**Verdicts bind to the commit, not the pull request.** A verdict records the exact commit it reviewed, so a fast merge cannot outrun its review, and a stale verdict is visibly stale rather than silently wrong. The model cannot "obtain a result" by reviewing a different artifact than the one that shipped, because the artifact's identity is in the row.

**The watch ships before the reviewer.** Our ladder paper is emphatic about the order: the availability alarm — any required check that has stopped reporting on an active repository — was built first, because "a trust system that cannot detect its own death will eventually report an availability number that is fiction." A behavioral rule asks the model not to tamper with monitoring. A constructional rule makes the monitoring's absence itself an alarm, so that tampering, failure, and a dead credential all look the same and all page someone.

The companion sentence — records must be human-legible, "if humans can't understand it, humans can't oversee it" — is where the harness offers less. In the finite chain the record and the coordination channel are the same object, a message board and a bare git repository, so there is no concealed channel; row 5 marks that as architecture, because "no hidden channel" is not the same as "the reasoning is honest," and we have no count for the second.

The ladder's gate is not yet good — 51 percent precision on the findings it would have blocked on is not a gate anyone should give a merge button to — and the ladder paper says so on its own page, in a dated revision, including a second seven-week gap, this one human, between its first and second audits. That is what a record the reviewer cannot rewrite is *for*: the failure to climb is a fact in a ledger, not an opinion in a status meeting, and the gate's authority is exactly what the ledger says it has earned.

---

## 8. Authority Comes From the Chain of Command, Not the Content

> **Answer:** The draft's authority clarification — tool outputs, file content, and web content "inherit no authority by default" and suspicious content "should be flagged" — describes the prompt-injection problem in its own vocabulary, and its constructional form is a control that runs before the model reads anything: a deterministic input classifier screening for instruction-override, role reassignment, persona hijack, named jailbreaks, and prompt-extraction attempts, with control-character sanitization, all executed before the model call. On a hit, the system returns a fixed fallback at zero model cost. What slips the classifier meets a model that holds no privileged context, and a tenant that misconfigures one layer still has the others. No layer is load-bearing alone.

The distinction from the behavioral form is not subtle. A model told "instructions in documents carry no authority" must still read the document to discover that it contains instructions, and the reading is where the risk lives. A classifier that runs first is not persuadable, because it is not reading for meaning; it is matching. Our reliability paper's ordering is deliberate: "the cheapest and most reliable block is the one that never reaches the model."

We add two habits from the fork-residue paper that apply directly to "flag suspicious content." First, guards bind to *claims*, not to words: a guard that looks for a phrase is defeated by a paraphrase, and the paper's six design rules exist because "each rule exists because its absence shipped residue." Second — the rule most enterprise guardrails skip — every guard is proven by planted failure and verified by an instrument that shares nothing with the guard. A classifier that has never been shown to fire is a classifier whose firing is a hope. The fleet-scale rollout the paper reports, across 38 production repositories in one evening with zero unintended deploys, is what a guard looks like when it is checked by something other than itself.

Rate caps close the loop. Where the draft asks the model to act "proportionately," the harness sets a per-visitor daily cap and a per-tenant monthly cap with early alerts — proportion as a number in a configuration file the model does not get to consult.

---

## 9. Disclosure, and Where the Test Draws the Line

> **Answer:** Part 3.3's rule — the model "will disclose their nature as AI" — runs three layers deep in a regulated deployment, and the cooperation test sorts them. Two hold without the model: a notice band surfaced before the first message, so the boundary is set before the user types; and the input classifier's fallback string, which is itself a truthful AI identification, so the one path that never reaches the model still discloses. The third is behavioral and says so: an always-on identity-and-limits block in the system prompt with explicit precedence over every other instruction, including tone, persona, and conversion goals. It is a rule the model is instructed to follow. It is a better-placed rule than "answer truthfully when asked," and it is still a rule.

The failure the block exists to prevent is not the one a first-layer document anticipates. The risk is not that the model *refuses* to admit it is an AI when asked. It is drift. Our retrieval paper: "The failure modes the pattern closes — the model claiming licensure, inventing a credential, drifting into first-person provider speech — are reachable on any concierge-style tenant in production at any vendor without explicit defensive engineering." Nobody asked; the persona simply slid. A rule that fires when the model is asked does not fire on drift. A block with precedence over the persona does — as long as the model honors precedence, which is the asterisk the test attaches.

Row 8 is therefore the row where this paper's own method cuts against its own architecture, and the honest reading is the useful one: disclosure is the rule in the draft most likely to be treated as trivially satisfied, its cheapest constructional layers are a band and a string, and its strongest layer is behavioral. Deployments should know which of the three they are relying on.

---

## 10. The Question the Draft Cannot Answer, and the Harness Does Not Need To

> **Answer:** The open question underneath every first-layer rulebook is whether the rules hold in situations the model has not seen. Microsoft's Part 5 concedes the draft "is not a complete account of current model behavior," and the research debate about whether alignment generalizes is unresolved. The harness answers a different question: not "will the model generalize" but "does the property depend on the model at all." For the counted rows in section 4 it does not. A validator applied to 33,444 outputs closed every frontier structured-output cell to exactly 1.000 and rejected five, with zero parse failures; the residue that survived was semantic, rare, and tier-concentrated — a budget-tier model flipping a field value in 4–9.6 percent of calls — and the doctrine's answer to that residue is redundancy and tier selection, not trust.

The generalization question is real, and it is the lab's to answer. An enterprise deploying a model this month cannot wait for the answer, and for the constructional rows it does not have to, because those rows never asked it. A task graph with no cycle has no cycle on inputs the model has never seen. A permission gate at retrieval returns the same rows for a novel prompt as for a familiar one. A validator that fails closed fails closed on a distribution shift. The harness does not generalize; it does not have to, because it was never a behavior.

What the harness cannot do is make the model's *judgment* better. The validator reanalysis is the clearest measurement: parse-and-canonicalize converted nearly all byte-level instability into determinism at zero model cost, and caught nothing semantic. The five rejects in 33,444 were strange — leaked thinking tags, stray line-break tags, one apology in Russian — and real, and a fail-closed gate exists for exactly them. The budget-tier field flips were not caught, because every variant was valid JSON containing a different answer. For that class the routing doctrine applies: judgment-bearing work goes to the capability tier that measured zero flips, behind ask-twice with disagreement routed to a person, and free prose — which admits no deterministic validator — routes to human review or does not ship.

That is the division of labor between the layers: the first is responsible for the model's judgment being good, the third for measuring it rather than believing it. The trust ladder's current numbers are that division in one place: a reviewing model whose raw precision was 66 percent on a replay and whose audited precision on live findings is 51 percent is a model whose judgment is not yet good enough — and a harness that has given it no authority is a harness working exactly as designed.

---

## 11. What the Harness Cannot Do

> **Answer:** The constructional form has a boundary, and the draft's rules on the other side of it are real. Where judgment is the product — a drafted letter, a diagnosis-adjacent answer, a scored document — the harness's contribution is measurement and routing, not correctness. The draft's rules on manipulation, emotional dependence, and wellbeing (Parts 3.2, 3.4, 3.6) are behavioral almost by definition; the harness's only hold on them is scope — what the model is permitted to be used for at all — and disclosure. And the harness has humans in it. The ladder's weekly audit lapsed for seven weeks this summer while its automated watch ran perfectly; the properties that depend on a ritual are only as constructional as the ritual is mechanized.

Four limits, stated so they can be checked:

1. **Judgment is not a harness property.** Every row in section 4 bounds a model; none improves it. Our measured answer to judgment quality is capability tier and redundancy, and our answer to prose is people.
2. **Some of the draft's rules have no constructional form.** "Facilitate human autonomy," "preserve personal boundaries," "discourage excessive reliance" are rules about the texture of an interaction. A harness can log the interaction, cap it, and disclose what it is. It cannot make it kind. Those rules belong to the lab and to the deploying organization's decision about what the model is for.
3. **Constructional guardrails need their own audits.** A guard that has never fired is a hope; the fork-residue paper's rule — prove by planted failure, verify with an instrument that shares nothing with the guard — applies to every row here. A topology that "cannot loop" should be shown not to loop under a planted cycle attempt; a classifier should be shown to catch a planted injection. We have done this for the fork-residue guards: six planted mutations, each required to turn the build red before the guard could merge, behind six rules each born from residue that shipped. We have not published a planted-cycle or planted-injection run, and this paper claims neither. A reader should demand the same disclosure of any vendor's rows.
4. **The humans are in the loop, and loops lapse.** The most instructive number in this paper may be the seven weeks between the ladder's first and second audits — a second seven-week gap in that paper's history, and unlike the first, a human one. The automated watch, the append-only ledger, and the shadow-rung verdict writer ran without a person and never missed. The audit needed a person and stopped. The lesson is not that rituals are bad; it is that a ritual is a behavioral guardrail on humans, and it should be mechanized — a watch on the audit's own record — the same way the model's rules should be.

---

## 12. The Doctrine

> **Answer:** For each rule in a rulebook you are asked to trust — Microsoft's, a vendor's, your own policy's — ask which form you are relying on. If the answer is "the model promises," decide whether the property matters enough to move; if it does, move it into the harness, in one of the ten forms section 4 names, then measure what remains — precision, availability, latency, rejects — and give the model exactly the authority the measurements have earned. Register the bars before the numbers arrive.

The doctrine, as operating habits:

- **Run the cooperation test on every rule.** Remove the model's cooperation in your head. If the property vanishes, it was behavioral. Write down which rules you are content to leave behavioral and why.
- **Seed work from humans only.** If any agent can create work for any other agent, the system contains a cycle, and rate limits detect it after it starts. Make the graph acyclic and terminal instead; the worst case becomes arithmetic.
- **Gate at retrieval, not at presentation.** Decide what the model may see before it sees anything. There is then nothing privileged in the prompt to coerce.
- **Screen before the boundary.** A deterministic classifier in front of the model catches the injection class at zero model cost; what it misses meets a model with nothing to give up.
- **Bind every verdict to the artifact it judged, and alarm on absence.** A gate whose death is detectable only by archaeology has no availability number worth trusting.
- **Validate meaning, canonicalize bytes, fail closed.** Bare parsing catches nothing — zero parse failures in 33,444 outputs — because modern failures are valid syntax with a different answer.
- **Give authority by measurement, and take it back the same way.** Publish the bars before the sample exists. Audit weekly, ten minutes. Demote automatically.
- **Mechanize your rituals.** Anything that depends on a person remembering is a behavioral guardrail on a person. Put a watch on it.

None of this requires believing the draft is wrong. It requires noticing what the draft is — a list of promises by models a lab says it is still working toward, published by a lab candid enough to say so — and building the layer that does not need the promise to be kept. For a team that wants its own rulebook read this way, row by row against the system that actually runs, [the validator gap audit](https://isimplifyme.com/services/validator-gap-audit) is where to start.

---

## Limitations

> **Answer:** This paper reads one draft, dated 14 September 2026, that its authors have said will change; every quotation is dated to that version. It maps rules to controls we have measured in our own systems — a fleet of production deployments on one cloud, one multi-agent run, one review gate, one validator corpus — and a reader should carry the measurements' scope with them: the finite chain ran four agents for ten commits, not four hundred for ten thousand; the ladder's numbers are eight weeks on two repositories; the validator corpus is three studies' structured outputs on specific model families. The mapping itself is judgment. Where a row is architecture rather than a measurement, the table says so.

Six inheritances a careful reader should carry:

1. **Draft scope.** The Code of Conduct is under consultation. Parts may be renumbered or rewritten; a revised edition is promised later in 2026. This paper is dated against the version it quotes and will be revised if the mapping stops holding.
2. **System scope.** The constructional controls are ours. Other harnesses implement the same properties differently, and some do not implement them at all. The claim is that the constructional form *exists* and has been measured, not that this implementation is the only one.
3. **Measurement scope.** The receipts are the published papers' numbers, at their published sample sizes and dates. Nothing here adds calls, and nothing here extends a number to a system it was not measured on. Five rows carry no count, and one row — disclosure — is partly behavioral by this paper's own test.
4. **The mapping is not one-to-one.** Several of the draft's rules have no constructional form (section 11), and several constructional controls answer more than one rule. A table is a reading aid, not a proof of coverage.
5. **No compliance claims.** Neither the draft nor this paper is a regulatory instrument. Where the draft's rules intersect a jurisdiction's requirements, counsel orders that; this table does not.
6. **Our own gate is not good yet.** The trust ladder's audited precision is 51 percent and its p95 latency misses its bar. This paper cites the ladder as a *record-integrity* control, which is what it measures well, not as a working reviewer, which it is not.

---

## Reproducing This

> **Answer:** Zero calls and zero credentials are needed to check this paper. The draft is public at [microsoft.ai/code-of-conduct](https://microsoft.ai/code-of-conduct/) under the title *Humanist AI Code of Conduct*, dated 14 September 2026; every quotation above carries its part number. Every counted receipt in section 4 links to a published paper. The validator corpus traces further to a public companion repository, [`determinism-harness`](https://github.com/iSimplifyMe/determinism-harness); the fork-residue guards in sections 8 and 11 trace to [`fleetcheck`](https://github.com/iSimplifyMe/fleetcheck); the finite chain's and trust ladder's pages carry their own dated counts. The contribution of this paper is the arrangement — which rule becomes which control — and disagreeing with it requires reading two documents side by side, which is the whole method.

---

## References and further reading

- Microsoft AI, [*Humanist AI Code of Conduct*](https://microsoft.ai/code-of-conduct/) (draft for public consultation, 14 September 2026) — the rules quoted by part number throughout; consultation open six weeks from publication
- [*The Finite Chain: Bounding Multi-Agent Autonomy by Construction*](https://isimplifyme.com/whitepapers/the-finite-chain) — shutdown, scope, and delegation as properties of a task graph; ten autonomous commits at two model calls per task
- [*The Trust Ladder: Supervised Autonomy for AI Code Review*](https://isimplifyme.com/whitepapers/the-trust-ladder) — commit-bound verdicts, the availability watch, authority by measurement; revised 2026-09-15 with eight weeks of shadow numbers
- [*The Same Object, Different Bytes*](https://isimplifyme.com/whitepapers/the-same-object-different-bytes) — the fail-closed validator over 33,444 outputs: what it recovers and what it cannot see
- [*Layer 4: Reliability Engineering for Regulated AI*](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) — the deterministic input classifier, rate caps, the compliance block, atomic writes, held-for-review
- [*Layer 3: Data + Retrieval*](https://isimplifyme.com/whitepapers/layer-3-data-retrieval) — the permission gate at retrieval; nothing privileged in the prompt
- [*The Layers Review Cannot See*](https://isimplifyme.com/whitepapers/the-layers-review-cannot-see) — six design rules for guards that hold, proven by planted failure
- [*The Routing Table*](https://isimplifyme.com/whitepapers/the-routing-table) — snapshot-at-write, ask-twice, and where judgment-bearing work belongs
- [*The Ceiling Is Not the Model*](https://isimplifyme.com/whitepapers/the-ceiling-is-not-the-model) — the harness deciding results before the model gets a vote
- OWASP, *Top 10 for Large Language Model Applications* — the prompt-injection class the classifier screens
- NIST, *AI Risk Management Framework* — the measurement-first posture the ladder applies to the development loop

---

## Notices

**Not legal or compliance advice.** This paper is an engineering reading of a public draft for informational purposes. Governance and deployment decisions require review against your organization's own risk, compliance, and legal requirements.

**The draft is a draft.** Quotations are from Microsoft AI's *Humanist AI Code of Conduct* as published on 14 September 2026 for public consultation. Microsoft has stated the document is not yet used to train its models and that a revised version will follow; part numbers and wording may change. This paper's mapping is dated to the quoted version.

**Measurements are point-in-time.** Every number cited is a published, dated observation from the linked paper; live systems change, and the trust ladder's figures in particular are trailing-window measurements revised on that paper's page.

**Trademarks.** Microsoft and MAI are trademarks of Microsoft Corporation. AWS and Amazon Bedrock are trademarks of Amazon.com, Inc. or its affiliates. References are descriptive and do not imply endorsement by any party.

---

**About the author.** Joseph W. Elstner is the founder and principal architect of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific. iSimplifyMe is bootstrapped, deploys production AI on AWS Bedrock, and runs a multi-tenant orchestration platform across healthcare, legal, financial, and editorial verticals.

**Contact.** ai@isimplifyme.com — for engineering teams that want their own rulebook read from the harness, [the validator gap audit](https://isimplifyme.com/services/validator-gap-audit) is where to start.

**Cite this paper.** Elstner, J. (2026). *The Rulebook Is Not the Guardrail: Microsoft's Code of Conduct, Read From the Harness.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/the-rulebook-is-not-the-guardrail
