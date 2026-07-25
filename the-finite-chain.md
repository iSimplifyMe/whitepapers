> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-finite-chain](https://isimplifyme.com/whitepapers/the-finite-chain)** — it revises there first; this mirror follows.
> Joe Elstner, Founder, iSimplifyMe · Published 2026-07-25 · License: [CC BY 4.0](LICENSE)

# The Finite Chain: Bounding Multi-Agent Autonomy by Construction

A production account of four AI agents maintaining a live pipeline through a shared message board — and the design decision that matters most in a multi-agent system: containing it by topology rather than by supervision, so the runaway loop is impossible instead of merely monitored.

---

## Abstract

> **Answer:** A multi-agent system should be bounded by its topology, not by its guardrails. The common failure is not an agent doing something wrong — it is agents generating work for each other until cost, or the codebase, runs away. The fix is structural: only humans originate tasks, the agent-to-agent chain is acyclic and terminates at an agent that triggers nothing, and the verification step in the chain is deterministic code rather than a second model. A system built this way cannot loop, so it needs no loop detection; its worst case is a known constant rather than an open question. In production this ran as four agents — a builder, a tester, a reviewer, and a researcher — coordinating through a shared message board and a bare git repository, landing ten autonomous commits against a live media pipeline at a steady-state cost of two model calls per task.

This is a production write-up, not a framework proposal. It describes a system that ran, what it shipped, and where it fell short. The architecture is deliberately small, and the interesting parts are the constraints rather than the capabilities. It is the multi-agent companion to [*The Trust Ladder: Supervised Autonomy for AI Code Review*](/whitepapers/the-trust-ladder) — that paper covers how a single non-deterministic reviewer earns the authority to block a merge; this one covers what happens when several agents operate at once, and how to bound them before any of them earns anything.

The intended reader is an engineer or architect evaluating multi-agent orchestration for real work and trying to distinguish the parts that are load-bearing from the parts that are demo. The argument is that most of the safety and nearly all of the cost predictability in a multi-agent system come from a handful of structural decisions made before any agent runs — and that those decisions are cheap to make and expensive to retrofit.

---

## 1. The Unbounded Loop Problem

> **Answer:** The characteristic failure of a multi-agent system is not a bad action, it is an unbounded one. When any agent can create work for any other agent, the system contains a cycle, and a cycle plus autonomy is a loop with no natural stopping point. Rate limits and spend alarms detect this after it starts; they do not prevent it. The structural alternative is to make the cycle impossible — humans originate all work, and the agent-to-agent graph is acyclic and terminates.

Most multi-agent demonstrations are a directed graph with no stated constraints on its edges. An agent finishes a task and posts a result; another agent reads that result and decides more work is warranted; a third responds to the second. Each individual step is defensible. The composition is a cycle, and once autonomy is added, a cycle runs until something external stops it.

The usual mitigations are all detective rather than preventive. A spend alarm fires after the spend. A rate limit throttles a loop rather than ending it — the loop continues, more slowly, which in practice means it is discovered later and costs the same. Loop-detection heuristics require defining what a loop looks like in a system whose whole premise is open-ended behavior. Each is a control on a system that is still, structurally, capable of running away.

There is a cheaper answer available before any of that: build a graph that cannot cycle. This costs nothing at runtime, requires no monitoring, and turns the worst case from an open question into arithmetic. The three decisions below are the whole of it, and none of them are sophisticated.

---

## 2. Coordination Without an Orchestrator

> **Answer:** The four agents are independent operating-system processes with no shared memory, no supervising daemon, and no direct knowledge of one another. They coordinate entirely through two shared artifacts: a message board they poll and post to, and a bare git repository they push commits into. Each has its own credentials, its own working copy, and its own state file. Removing the orchestrator removes a single point of failure and makes the coordination protocol inspectable — the full history of who did what, and why, is a channel you can read.

The coordination substrate is **AgentHub**, an open-source server by Andrej Karpathy: a bare git repository plus a message board, deliberately generic about what the agents connected to it are optimizing. The platform carries no opinion about roles or workflow. Everything specific to this system — which agents exist, what triggers them, what they are allowed to do — lives in the agent layer built on top, which is what this paper describes.

The architecture is three components:

**A server holding a bare git repository and a message board.** Agents push code as git bundles, which the server validates and unbundles. They read and post to named channels. Both surfaces are shared state; neither is shared memory.

**Four agent processes.** Each runs independently with its own API credentials, its own git working copy, its own process ID, and its own state file tracking the last message it handled. An agent can be stopped, restarted, or removed without coordinating with the others. Nothing supervises them.

**A model endpoint.** The three reasoning agents call Claude Opus 4.6 through Amazon Bedrock. The fourth calls no model at all, for reasons that get their own section.

What replaces the orchestrator is a convention: agents poll the board and act when they see a message addressed to them. A human posts `@builder <task>`. The builder sees the mention, does the work, commits, pushes, and posts a completion notice. The tester is watching for exactly that notice.

The property this buys is worth stating plainly. **The coordination protocol is a readable artifact.** There is no orchestrator whose internal state must be inferred from logs — the message board *is* the state, in the order it happened, in prose the agents wrote to each other. Debugging a multi-agent system is usually an exercise in reconstructing what each component believed at the time. Here, you read the channel.

---

## 3. The Finite Chain

> **Answer:** Three constraints make the system incapable of unbounded execution. First, only humans originate tasks — no agent can create work for another agent. Second, the automatic chain is a straight line: builder triggers tester, tester triggers reviewer, and the reviewer triggers nothing. Third, the one agent capable of open-ended analysis never auto-triggers and runs only on an explicit human mention. The result is that the maximum work produced by a single human instruction is a fixed, countable quantity.

This is the load-bearing section. The rest of the system is ordinary; this part is the design.

**Only humans originate.** Agents respond to mentions but cannot post mentions of their own. This single rule eliminates the entire class of failure where two agents discover an interesting problem and pursue it into the night. An agent that finds something worth doing reports it; a human decides whether it becomes work.

**The chain is a line, and the line ends.** The automatic sequence is:

```
human: @builder <task>
   → builder: reads code, writes fix, commits, pushes, posts "Builder completed … <commit>"
   → tester:  sees completion, fetches commit, runs its suites, posts PASS/FAIL
   → reviewer: sees PASS, fetches commit, reviews, posts findings by severity
   → (nothing)
```

The reviewer is a terminal node. It consumes a trigger and emits none. That is the entire termination argument — there is no cycle-detection logic anywhere in the system, because the graph has no cycle to detect.

**The open-ended agent is manual-only.** The researcher analyzes architecture and produces recommendations — the least bounded task in the system, and the one most likely to generate follow-on work. It never auto-triggers. It runs when a human writes `@researcher`, with an explicit focus (performance, architecture, next steps, or a full pass). The most expansive capability is behind the most deliberate gate.

Take these together and the worst case is arithmetic rather than speculation. One human instruction produces at most one builder run, one tester run, and one reviewer run. Cost per task is not a distribution with an unpleasant tail; it is a small integer.

The broader principle: **in an autonomous system, prefer constraints that make a failure impossible over controls that make it visible.** A monitored cycle is still a cycle. An acyclic graph needs no monitor. The first is a control you must maintain, tune, and trust; the second is a property of the design that holds whether or not anyone is watching.

---

## 4. The Verifier Must Not Be a Model

> **Answer:** The tester in this system makes zero model calls. It runs seven suites of structural checks — 52 in total — implemented as deterministic pattern matching over the code, and posts a pass/fail report. This is deliberate. A model verifying another model's output is a correlated check: both share training data, failure modes, and blind spots, and a model that misunderstands a requirement while writing code will tend to misunderstand it identically while reviewing. A deterministic verifier is uncorrelated with the generator, always agrees with itself, and costs nothing per run.

The instinct in a multi-agent system is to make every agent a model, because the model is the interesting part. Applied to the verification step, that instinct is wrong on three counts.

**Correlated failure.** The value of a verification step is the independence of its errors from the generator's. Two instances of the same model reviewing each other are not independent. The failure mode that matters — a plausible, confidently-wrong implementation of a misunderstood requirement — is precisely the failure a sibling model is least equipped to catch, because it shares the misunderstanding.

**Nondeterminism where determinism is the point.** A verification gate that returns different answers on identical input is not a gate. It is a suggestion with a pass rate. When the tester reports PASS, that fact is reproducible; running it again produces the same answer, and a failure points at a specific check rather than a mood.

**Cost and latency at the highest-frequency node.** The tester runs on every builder completion — the most frequently executed step in the chain. A deterministic tester makes that step free and effectively instant, which is what makes it affordable to run on every single change rather than in batches.

The tester's suites cover configuration validity, pipeline structure, model-invocation compliance, dynamic script generation, secrets handling, external tool availability, and command-line arguments. All 52 checks are structural — they inspect the shape of the code without executing it.

The limitation is real and worth stating: a structural checker verifies that code has the right shape, not that it does the right thing. It catches a removed guard, a hardcoded secret, a violated invocation convention, a malformed config. It cannot catch a logic error that leaves the structure intact. That is precisely why the reviewer — which *is* a model — sits downstream of it. The two are complementary and ordered deliberately: the cheap deterministic filter runs first, and the expensive probabilistic one runs only on what survives.

The general form of the rule: **use a model where judgment is required, and code everywhere a rule can be written.** The verification step in an autonomous loop is the last place to spend a model call, not the first.

---

## 5. Cost as a Property of the Topology

> **Answer:** Because only humans originate work and the chain terminates, the system's cost is a function of human instructions rather than elapsed time. The steady state is two model calls per task — one builder, one reviewer — with the tester free. Idle agents cost nothing: they poll a local server and make no billed calls when there is no work. Per-agent hourly caps and server-side limits on pushes and posts exist, but they are a backstop for a system already bounded by its shape rather than the primary control.

Cost predictability in this system is a consequence of the topology, not a feature bolted onto it.

**The steady state is two calls.** A typical task is one builder call to write the change and one reviewer call to review it. The tester adds nothing. There is no per-task variance from agents negotiating with each other, because they cannot.

**Idle is genuinely free.** Agents poll a local server. With no tasks posted, they consume no model tokens. The system's cost when nobody is using it is zero, which is the correct behavior and not the default one — a design where agents periodically "check whether anything needs doing" against a model does not have this property.

**The limits are a backstop, not the mechanism.** Each agent is capped at ten model calls per hour on a sliding window held in its own state file. The server independently caps git pushes and message posts per agent per hour, and bounds the size of an accepted git bundle. These are enforced server-side, so a misbehaving agent cannot raise its own ceiling.

The ordering matters. In a system whose cost is bounded only by rate limits, the limits are the safety mechanism, and they are doing load-bearing work under adversarial conditions. Here they are a second layer under a system that is already bounded structurally. That difference shows up when a limit is misconfigured: in the first design, a misconfiguration is an incident, and in the second, it is a misconfiguration.

---

## 6. What the Swarm Actually Shipped

> **Answer:** Ten commits, all authored autonomously by the builder agent from natural-language task descriptions, against a live multi-stage media pipeline that chains local processing with remote GPU inference. The work was ordinary maintenance engineering — retry helpers, timeouts, argument-handling fixes, subprocess-safety changes, input validation — plus two changes that measurably improved output quality by removing lossy compression stages. This is the honest scope: real, useful, unglamorous work, not a system designing itself.

The workload was a multi-stage media generation pipeline: a chain of processing steps that combines local CPU work with GPU inference offloaded to a remote machine over SSH, passing intermediate files between stages. It is a good test case for autonomous maintenance because it is real, it is fragile in the way real pipelines are fragile, and its failure modes are concrete.

The ten autonomous commits:

1. Initial pipeline import.
2. A retry helper for the GPU-offload stages.
3. A `--dry-run` flag.
4. A `--batch` flag.
5. A timeout on the remote-execution call.
6. A fix for a double-argument bug in post-processing.
7. A refactor of all file-transfer calls to avoid shell interpolation.
8. An input-validation function for configuration files.
9. A switch to a lossless intermediate codec for the GPU-inference prep stage.
10. A merge of two processing passes into one.

Items 9 and 10 are the ones with measurable output impact, and both are the same class of insight: the pipeline was losing quality to unnecessary re-encoding. The prep stage had been compressing with a lossy codec before inference ran, degrading detail before the expensive step even started; switching to a lossless intermediate removed that loss. Separately, two stages that each re-encoded the full frame sequence were merged into a single pass, eliminating one of four lossy generations end to end.

Item 7 is worth noting for a different reason: it is a security-relevant change — moving subprocess calls off shell interpolation — produced from a plain-language task description, and it is exactly the kind of maintenance that stays permanently at the bottom of a human backlog.

What this evidence does and does not support:

- **Supports:** an agent swarm can do useful, non-trivial maintenance engineering against a real codebase, unattended, at a predictable cost, without a human reviewing each step.
- **Does not support:** any claim about autonomous system design. Every one of these tasks was scoped by a human. The system executes well-specified work; it does not decide what is worth doing.

---

## 7. What It Did Not Solve

> **Answer:** Four honest limitations. The reviewer surfaced the same findings repeatedly without fixing them, because reporting and acting are separate steps and nothing closed the loop. There is no checkpoint or resume, so a failure late in a long pipeline run discards everything before it. The human remains the bottleneck by design, which caps throughput at the rate a person writes task descriptions. And the whole account is one workload on one machine over a bounded period — an existence proof, not a benchmark.

**Recurring findings that nobody acted on.** The reviewer reliably identified the same handful of issues across runs: a remote-execution helper ignoring return codes so failures cascade silently, missing timeouts on outbound HTTP calls, a retry helper that was implemented but never actually called in the main path, intermediate files left behind when the pipeline fails, and secrets loaded at import time so a misconfiguration crashes before argument parsing produces a useful error. Every one is a legitimate defect. They persisted because the reviewer posts findings and the builder acts on human instructions — and no human turned those findings into instructions. This is a workflow gap, not a model failure, and it is the most obvious thing the design gets wrong.

**No checkpoint or resume.** The researcher's analysis identified this and it was never addressed. A pipeline run that fails near the end restarts from the beginning. Related recommendations — pooling the repeated SSH connections, parallelizing two independent stages — were likewise correctly identified and never implemented. The system was better at finding work than at getting it done, which is the same gap as above viewed from the other side.

**The human is the bottleneck, by design.** Throughput is capped by how fast a person writes task descriptions. This is the direct cost of the containment property in section 3, and it is the right trade for this workload. It would be the wrong trade for a workload requiring sustained autonomous progress, and anyone adopting this pattern should be clear about which of the two they have.

**One workload, one machine, bounded period.** Four agents, one pipeline, ten commits. No claim is made about how this behaves with twenty agents, multiple concurrent workloads, or agents from different vendors. The structural argument in section 3 should generalize — it is a property of graphs, not of scale — but the operational experience does not.

---

## 8. When This Pattern Applies

> **Answer:** Use it when the work decomposes into well-specified tasks a human can describe, when a deterministic verifier can be written for the domain, and when bounded cost matters more than autonomous throughput. Do not use it when the system must make progress without a human in the loop, when correctness cannot be checked structurally, or when tasks are genuinely exploratory. The pattern trades throughput for predictability, which is the right trade far more often than the reverse — but it is a trade.

The pattern fits when three conditions hold.

**The work is specifiable.** Someone can write down what needs doing in a few sentences. Maintenance engineering, refactoring against a known target, applying a policy across a codebase, and routine fixes all qualify. "Improve the architecture" does not.

**A deterministic verifier exists.** There is a meaningful class of structural checks for the domain — schema conformance, invocation conventions, forbidden patterns, required guards. Without that, the tester step collapses into a second model call and section 4's argument is lost.

**Predictability is worth more than throughput.** This is the crux. The system is deliberately slower than one where agents generate their own work, and in exchange, its cost and blast radius are known in advance. For most production systems that trade is correct. For a research setting where the goal is open-ended exploration, it is not.

Three implementation notes that transfer independently of the rest:

- **Make the terminal node explicit.** Whatever the topology, one agent must consume a trigger and emit none. Name it, and check that property when adding an agent — a new agent that posts a mention converts an acyclic graph into a cyclic one, and that is a one-line change with unbounded consequences.
- **Put the deterministic check first.** Order the chain so cheap uncorrelated verification filters what reaches expensive probabilistic review.
- **Give agents separate identities.** Separate credentials, working copies, and state files per agent mean one agent's failure or compromise is contained, and its actions are attributable in the git history without inference.

The claim of this paper is narrow and, I think, defensible: the safety properties that matter most in a multi-agent system are structural, they are decided before any agent runs, and they are cheap. A system that cannot loop does not need to be watched for looping. Most of the engineering effort spent on multi-agent guardrails is spent detecting conditions that a different topology would have made impossible.

---

**Measurements are point-in-time.** The commit count, check counts, call-per-task figures, and rate limits cited here are frozen observations of the system described, at the time of writing. They describe one workload on one machine over a bounded period, and revisions to this paper will reflect material changes.

**Attribution and trademarks.** The coordination substrate described in section 2 is AgentHub, authored by Andrej Karpathy; the agent layer built on top of it, and every design decision this paper argues for, are iSimplifyMe's. Claude is a trademark of Anthropic, PBC. AWS and Amazon Bedrock are trademarks of Amazon.com, Inc. or its affiliates. References are descriptive and do not imply endorsement.

---

**About the author.** Joe Elstner is the founder of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific. iSimplifyMe is bootstrapped, deploys production AI on AWS Bedrock, and runs a multi-tenant orchestration platform across healthcare, legal, financial, and editorial verticals.

**Contact.** ai@isimplifyme.com — corrections, or evidence against any specific claim in this paper, are welcome.

**Cite this paper.** Elstner, J. (2026). *The Finite Chain: Bounding Multi-Agent Autonomy by Construction.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/the-finite-chain
