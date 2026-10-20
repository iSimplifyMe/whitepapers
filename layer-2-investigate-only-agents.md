> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/layer-2-investigate-only-agents](https://isimplifyme.com/whitepapers/layer-2-investigate-only-agents)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-10-19 · License: [CC BY 4.0](LICENSE)

# Layer 2: The Detector Sets the Precision

Investigate-only agents — the Sentinel pattern in detail: a typed agent registry, an SQS-driven generic runner, atomic conditional-write locks for race protection, cost ceilings at the agent level, and what five months of the production record say about where the human gate belongs.

---

## Abstract

> **Answer:** An investigate-only agent is safe to run against production because everything that bounds it is decided in code, not by the model: what it may read, how many tool calls it gets, what a session may cost, what it may do (file a ticket and post one line), and, where the evidence is machine-readable, when its ticket closes. Those bounds are the architecture. Five months of our own record show why the human gate is the design rather than a fallback. Across 49 stuck-draft investigations the detector's precision went from 0 of 33 to 1 of 14 to 2 of 2 as two detector fixes shipped; the investigator's classification inherited the detector's framing and was wrong on all 45 tickets where it claimed a pipeline-wide stall; and the confidence the investigator reported on its own work carried no usable information. The agent may investigate. A person decides.

This is a production account inside a reference-architecture series. Layer 2 of the stack is orchestration; this paper narrows to the one orchestration pattern we run against our own production fleet every day — agents that detect, investigate and escalate, and never act. The pattern itself is already summarized in [Layer 4](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering), section 4, and on the [Sentinel lab page](https://isimplifyme.com/labs/sentinel). What those pages do not carry is the mechanism at the level of the code, and the record of what three such agents — one of them run once at launch and silent by configuration since — did between 28 April and 19 October 2026. That record is 98 tickets across 91 runner invocations (an invocation drains the jobs it is handed, so one can file several), no cost-ceiling warning in five months of the operations channel's history, and a precision history that is the most useful thing we learned.

The intended reader is the person asked to approve an AI agent near production — an engineering lead, a platform owner, a security reviewer — who wants to know, before the demo, what the agent cannot do; section 5 carries the two permission statements that answer it. The argument is that the answer should be structural. An agent whose worst case is a filed ticket does not need to be trusted; it needs to be bounded, and the bounds need to be where a reviewer can read them.

---

## 1. The Line Between Detection and Action

> **Answer:** The outside evidence for keeping an agent on the investigation side of the line is now measured. On OpenRCA 2.0, eleven frontier models recover the exact root-cause set of an incident in 20.7% of cases on average; they name a correct service in 76.0% but ground it in a verified causal path to the symptom in only 61.5% — a gap the authors call ungrounded diagnosis. Practitioners who evaluated an MCP-grounded Kubernetes diagnostician trusted its diagnoses and consistently doubted its remediations. The design consequence is the one we built: an agent reads, reasons, and writes a ticket; a person acts.

[Layer 4](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) states the rule — automated detection is fine, automated remediation is not — and this paper supplies the mechanism and the evidence.

Fang and colleagues' [*OpenRCA 2.0*](https://arxiv.org/abs/2606.27154) scores root-cause analysis at the level of the causal process rather than the final label. Across 500 cases on three microservice systems and eleven frontier models, the exact root-cause set is recovered in 20.7% of cases on average; the best model reaches 29.4%. The more telling number is the gap between naming and grounding: agents name at least one correct service in 76.0% of cases but connect it to the observed symptom through a verified propagation path in only 61.5%. The authors call the remainder ungrounded diagnosis, and they attribute most of it to three habits — committing to the first plausible cause, trusting what is visibly active and reading silence as health, and following the loudest signal. Their appendix adds a note that any reviewer of an agent design should read twice: the benchmark "is not intended for production-readiness certification of any specific RCA agent."

Kim, Park, Yun and Lee asked [*why agents systematically fail at cloud root-cause analysis*](https://arxiv.org/abs/2602.09937) across 1,675 agent runs on five models and sorted the failures into twelve pitfalls. The most prevalent — among them hallucinated interpretation of data and incomplete exploration — appeared on every model regardless of capability, which points at the agent design rather than the model, and refining prompts alone did not fix them. The same group's [trajectory-level study](https://arxiv.org/abs/2608.21310) of 5,000 agent runs, by Lu and colleagues, found the same reading trajectories rather than outcomes: a correct localization can hide a deficient causal reconstruction, and the recurring failures are omitted evidence, misinterpreted evidence and unsupported reasoning.

The most useful outside result for the human-gate question, preliminary as its authors call it, comes from Senja, Razavi Zadegan and Leitner's [*ARGUS*](https://arxiv.org/abs/2608.23084), an assistant that connects a commercial model to live cluster telemetry through MCP servers and posts its write-ups to Slack. It named the injected root cause in all ten scenarios. The six practitioners who evaluated it found the diagnoses credible and were consistently doubtful about the recommended fixes — what the authors call a diagnostic/prescriptive asymmetry, and what we would call the reason the recommendation field on our tickets is advisory. Kuvshinova and Jin's [auditable graph-guided analysis](https://arxiv.org/abs/2606.08590) arrives at the same operating constraint from another direction: read-only evidence collection, deterministic operations gathering the evidence and checking the verdict, and no claim of production readiness from a benchmark.

None of this says an agent cannot help. It says the help is diagnostic: right often enough to be worth reading, wrong often enough that nothing should fire on it.

---

## 2. The Registry: One Typed Record per Workload

> **Answer:** Every investigate-only workload is one typed record in a registry: a slug, a title, a cron expression, a system prompt, a tool list, a pure detection function that returns typed jobs, a kickoff-message builder, an optional idempotency lock key with a time-to-live, a per-session tool-call cap and a per-tick session cap. The defaults are a one-hour lock, sixteen tool calls and ten sessions a tick; each workload overrides what it needs. Duplicate slugs throw at module load. Adding a workload is a config file, a detector, its tools and a cron rule — nothing in the shared detector or runner changes.

The registry is a single array; each entry is the source of truth for one workload, and the two shared Lambdas — a detector and a runner — do nothing but dispatch off it. The interface is small enough to print:

| Field | What it decides |
|---|---|
| `slug`, `title`, `cron` | Identity and cadence; the cron rule for each workload invokes the same detector handler with the slug in an environment variable |
| `systemPrompt` | The investigator's instructions, including the ticket template it must fill |
| `tools` | The shared output tools plus the workload's own read-only tools |
| `detect(ctx)` | A pure function of the table and the clock that returns every candidate job for this tick — no lock checks, no caps |
| `buildKickoffMessage(job)` | The first user message of the session, built from the typed job |
| `lockKey(job)`, `lockTtlSeconds` | The idempotency key and how long a candidate stays claimed; default one hour |
| `maxToolCallsPerSession` | The runaway brake on a single investigation; default sixteen |
| `maxSessionsPerRun` | The runaway brake on a single tick; default ten |

Those eleven fields are the whole contract; nothing a workload can do is outside them. Three workloads are registered:

| Workload | Cadence | Lock scope | Lock TTL | Tool-call cap |
|---|---|---|---|---|
| Diagnostics Agent — fires only for tenants whose activation flag is set and whose three newest uptime probes all failed | every 5 minutes | per tenant, on an open-incident key | 1 hour | 16 |
| GH Triage Agent | every 15 minutes | per repository and workflow run | 24 hours | 12 |
| Pipeline Hang Detector | every 15 minutes | per anomaly kind and record | 24 hours | 12 |

---

## 3. The Detector: Pure Detection, Centralized Policy

> **Answer:** One generic detector Lambda serves every workload. On each tick it calls the workload's pure detection function, then applies the policy the workload cannot: for each candidate it reads the lock row, writes it with a conditional put that fails if the row already exists, and only then enqueues the job. The conditional write is what makes two detectors racing on the same tick safe — exactly one wins, the other logs a skip. A per-tick cap bounds the worst case, and a summary line is logged on every tick including the healthy zero, because a silent success is indistinguishable from a swallowed failure.

The detector handler is 165 lines, most of them policy. It looks the workload up by slug, calls `detect()`, and walks the candidates: compute the lock key, read the lock row, and if none exists write one — a put with the condition `attribute_not_exists(pk)` and a time-to-live — then send the job to the SQS queue. If the conditional write fails, another detector got there in the gap between the read and the write, and the candidate counts as `skipped_locked`. That is the whole race protection: no coordinator, no leader election, one atomic conditional write against the same table the rest of the platform uses.

The cap is the other half of the policy. A detection function that suddenly returns a hundred candidates — a bad threshold, a bad deploy, a bad day — produces at most ten sessions on that tick and a warning that names the remainder, which are candidates again on the next tick, minus the ones already locked.

On every tick the detector logs `{candidates, enqueued, skipped_locked, cap_hit}`, including the ordinary tick where every count is zero. Both the cap warning and the summary are log lines, not alarms; nothing wakes anyone on them, and the person who wants the zero has to go and read it.

The detectors run on a 60-second timeout with 256 MB of memory. Over four days in late September the pipeline-hang detector hit that timeout 51 times, because the CloudWatch Logs API answered its log scans with "The server failed to fulfill the request. Please try again." The scans are wrapped so a failed read is a warning rather than an error, but enough of them in one tick ran the handler past its budget. Lambda's asynchronous retry re-ran the tick — 98 to 112 invocations a day against 96 scheduled — the next tick ran normally, and the condition cleared on its own after the fourth day. The 51 timeouts were found in the log on 19 October, not alerted on at the time — which is what "log lines, not alarms" costs. We mention the window because it is what fail-open looks like from the outside: a provider-side fault cost some ticks, and no ticket was filed on the strength of a read that had not happened.

---

## 4. The Runner: One Loop, Four Brakes

> **Answer:** One generic runner consumes jobs from the queue, opens a streaming conversation with the model using the workload's prompt and tools, and loops: dispatch each tool call, append the results, continue. Four brakes bound the loop. A turn limit of fifty. A tool-call cap of twelve or sixteen, from which the two output tools are exempt so a ticket can always be filed. A soft cost ceiling of thirty cents that posts a warning and a hard ceiling of one dollar that breaks the loop. And the model's own stop reason. Every session whose transcript survives ended on the model's stop reason, and in five months of the operations channel's history — where the runner posts every ceiling crossing — no ceiling warning was ever posted.

The runner is the only place the model is called, and it is the same code for every workload. It reads the job off the queue, finds the workload in the registry, and sends the kickoff message with the workload's system prompt and tool definitions. The response streams back; the runner collects the tool-use blocks and the usage metadata, dispatches each tool call to its handler, and sends the results back as the next user message. Thinking is pinned off, in the form the model's generation accepts, because the loop rebuilds each assistant turn from tool-use blocks only; output is capped at 4,096 tokens a turn.

The four brakes, in the order the code checks them:

1. **Cost.** After every turn the runner prices the accumulated usage. Past thirty cents it posts a warning to the operations channel once and continues; past one dollar it logs, posts, and breaks. The ceiling is per session and is enforced by the runner, not requested of the model.
2. **Stop reason.** `end_turn`, `max_tokens` and `stop_sequence` end the session and log the final cost, the tool-call count and the turn count. Any other stop reason that is not a tool call is logged as unexpected and also ends the session. A stream error throws; what happens to the job then is below the list.
3. **Tool calls.** Each investigative tool call counts against the workload's cap — twelve for the two newer workloads, sixteen for diagnostics. A call past the cap is answered with an error payload naming the cap, not dispatched. The two output tools, `file_ticket` and `notify_slack`, are exempt, so an investigator that has exhausted its budget can still file a ticket — including the ticket that says `unknown`.
4. **Turns.** Fifty, as a last resort. No session in the surviving logs has needed more than six.

A thrown session puts its job back on the queue: the message becomes visible again after the five-minute visibility timeout and is redelivered, for up to four days, until a session completes. The lock row the detector wrote stays until its time-to-live, so the candidate is not re-detected while it holds; a redelivered message is a second session of the same job, not a parallel one — an empirical property, since the visibility window equals the runner's own timeout where the provider's guidance is six times it, and sessions finish in a fraction of either. A throw fails the whole batch the invocation was handed — up to ten jobs — so the queue redelivers completed siblings too, and a sibling investigated again can file a second ticket; nothing bounds the redeliveries except the four-day retention, which section 10 prices, and an event-source setting plus a handler that reports which message failed would confine a throw to its own job. The log shows this path taken once.

The same ordering has a quieter gap: the detector writes the lock before it enqueues, so a lock written before an enqueue that fails holds the candidate out for the lock's lifetime, and nothing would alert on it.

The runner's own limits are 300 seconds and 1,024 MB. The model since 1 October is Claude Sonnet 5.5, resolved through the stage's application inference profile from the same registry the rest of the platform uses, so the runner carries no model identifier of its own.

What the log says about the brakes: the runner has been invoked 91 times since launch, and exactly one invocation has errored, in May. Session transcripts are kept for thirty days, and in the two overlapping windows we read, which together cover 2 September to 19 October, every session ended on `end_turn`, with three to nine tool calls and five to six turns each. A session costs between nine and twelve cents at the current list price for the model.

---

## 5. The Two Tools and the Ticket

> **Answer:** An investigate-only agent's output surface is two tools. `file_ticket` writes one ticket into the same support table clients use, with a category from a fixed set and a markdown body of at most 10,000 characters. A source field keeps machine-filed tickets off every client's support view and out of their open-ticket allowance. `notify_slack` posts one line of at most 2,000 characters to the operations channel; the prompt asks for one per incident, after the ticket exists, and the turn and cost ceilings are what bound the count. There is no third output tool. The agent cannot write to a repository, update any record, or reach anything beyond the ticket row it creates.

The two output tools are shared by every workload and concatenated onto each workload's own read-only tools. `file_ticket` takes a tenant, a category and a description, and calls the same ticket-creation function the client support form calls, with one difference: it stamps `source: "sentinel"`. The client-visibility rule is a single function — a ticket is hidden from clients if its source is sentinel or its category is one of the machine categories — so an investigation into a tenant's pipeline never appears on that tenant's support tab and never counts against them.

A session that has gone wrong can file more than one ticket before the turn or cost ceiling stops it, and the record shows none has. The ticket body follows a template the system prompt dictates: a summary of what fired and the context the detector supplied, a classification with a root cause, a confidence and a recommended intervention, an evidence section, a suggested next step written for the person who will read it, and notes. The evidence section is the product: the classification is the agent's opinion of it, the recommendation is advisory, the confidence, as section 9 shows, is a reading aid at best, and the prompt's rules — section 7 — exist to keep that section honest.

The bounds are enforced in two layers, and the reviewer this paper is written for approves on the permission statement, not the prose. The runner's execution role can consume its own queue, read and write the platform's single table (including scans), read a whitelisted set of the content pipeline's log groups, and invoke the model; it can list every log group's name and read four patterns of them; nothing else in the role reaches a repository, a deploy or a client system. Within the table the role could put or update any item, and it is the two tool handlers, not the role, that confine writes to ticket rows, as it is the read tools' own queries, not the role, that confine reads — the in-table bound is code, and a reviewer should know which layer holds which bound. The investigator's read tools are named and few:

- **Pipeline:** content records, pipeline configuration, lock rows, event counts and a log tail capped at 16 KB per read.
- **CI:** a workflow run, its logs, recent commits and the workflow file — read through a fine-grained GitHub token scoped to repository metadata, contents and Actions, all read-only; it is the second permission statement a reviewer should read.
- **Diagnostics:** a tenant's context, an HTTP fetch and a DNS lookup.

The tenant on a ticket is supplied by the investigator from its kickoff, so a poisoned input could make it write under another tenant's partition key; every machine ticket is hidden from every client regardless of the tenant written on it, which is the control that matters. And the inputs are untrusted on purpose: logs, CI output and commit messages can carry instructions, and the worst a poisoned input can produce in this design is a misleading ticket and a misleading line. That makes the person reading the output the surface an attacker would aim at: a 2,000-character line in an operations channel, or a 10,000-character ticket with a "suggested next step" field, is a plausible place for an instruction dressed as an operator's note. It is why the prompt asks for a ticket reference and a summary rather than a link to act on — a convention of the prompt, not of the handler, so a reader should treat the line as model-written text — and why the next-step field is advisory and nothing in it is executed by anyone but a person who has read the evidence above it.

The investigator also reads across tenants by design — the fleet-wide completion count, content records without a tenant filter — while its output is hidden from all of them; on a multi-tenant platform that asymmetry is the point of the hiding, and the reads go to the same inference profile, under the same no-retention terms, as every other model call the platform makes.

"Human in the loop" covers a lot of designs. In this one there is no approval button, no remediation path waiting on a click, no action the agent can take that a person then authorizes. The agent files a ticket and posts a line; a person reads the line, opens the ticket if it warrants it, and does whatever is needed by hand. For the first five months the ticket's status field was not even part of the process — 63 of the first 84 tickets were still marked `submitted` on 2 October, because people were reading the channel, not working a queue. Section 8 describes what we built when that stopped scaling.

---

## 6. The Detector Sets the Precision

> **Answer:** The Pipeline Hang Detector filed 49 stuck-draft tickets between 9 June and 2 October. Adjudicated against the pipeline's own records — each record's stage history, plus fleet-wide evidence of whether the writer was draining other records at the time — three were real stalls. The precision tracks the detector's two fixes step for step: 0 of 33 before the dwell clock was anchored on draft entry instead of proposal time, 1 of 14 under a four-hour threshold that a batch day's fleet-wide queue depth routinely exceeded, and 2 of 2 after the threshold was raised to six hours and the detector learned to stay quiet while the writer is demonstrably working. An investigator cannot be more right than the question it was asked.

What the detector watches is the [content pipeline](https://isimplifyme.com/labs/content-pipeline) that writes long-form content for about two dozen tenant sites, roughly 35 published drafts a week between June and October: topics are proposed, approved by a person, drafted by a single writer that drains a shared queue two drafts per fifteen-minute cycle in approval order, scored, and published. The stuck-draft check is simple in intent — a record in the draft stage with no content for too long is a candidate — and the whole precision story is in how "too long" was measured.

**Before 17 June: 0 of 33.** The dwell clock started at the record's creation time — the moment a topic was proposed — rather than the moment it entered the draft stage. A topic that waited a day or more for approval and was then approved in a batch read as stuck for that whole wait — "over 40 hours", one ticket said — the instant it became a draft. Five of the thirty-three tickets in this period flagged records that had not entered the draft stage at all; the other twenty-eight flagged records the writer reached and published within three hours, in approval order, while it was completing other records around them. Every one was a false positive. The fix was one line of intent: anchor the clock on draft entry.

**17 June to 7 September: 1 of 14.** With the clock fixed, a four-hour threshold met the arithmetic of a batch day. The writer drains one queue for the whole fleet, oldest first, two drafts a cycle, and each draft takes minutes to generate; a record approved on a batch day waits behind every record approved before it. Between approval and ticket, each of the thirteen false positives in this period had six to eighteen other records — mostly other tenants' — complete ahead of it, and in every one the writer had completed other records within the hour before the ticket. Four hours was ordinary queue depth, not a stall: the writer completed each ticketed record within hours, with no gap in its activity longer than ninety minutes.

The fourteenth was real: a record approved in the morning, a writer that completed nothing for more than six hours after the ticket, and a draft finally written thirteen hours after approval. The fix had two parts: raise the threshold to six hours, and suppress a candidate while the writer has completed anything, anywhere in the fleet, in the last forty-five minutes — with a twelve-hour ceiling past which a record is ticketed regardless. A record still stuck after its ticket is not ticketed again for a day, because the lock holds that long; the ceiling is a detection rule, not a re-page.

**After 7 September: 2 of 2.** Only two tickets have fired since. In the first, a writer whose account had been throttled stopped advancing every record it had — three, approved in one batch; a person parked all three by hand that night and released them the next morning, and the writer resumed within ten minutes — whether the ticket was what they read first is not recorded. In the second, a record sat in draft for fourteen hours while the writer completed nothing for nearly eight hours after the ticket. Both were real stalls. Both investigations — section 7 — declined to call the writer down and said so.

Four tickets in the period were not stuck-draft tickets, and they belong in the account for completeness. Two flagged records sitting in the scoring stage without a score, a condition the kickoff message itself labels as a known gap; the investigator classified them as instructed and recommended no action, which was right. One flagged a spike in records held for review during a launch; the records' own hold reasons — a claims gate and a panel score under threshold — confirm that the validator was doing its job, and the ticket said so. And one orphaned-lock ticket fired three minutes after a fix to the writer had deployed, following a day on which the pipeline's invocations had been dying at their limit; the record can neither confirm nor refute it.

---

## 7. The Investigator Inherits the Detector's Framing

> **Answer:** On 45 of the 49 stuck-draft tickets the investigator classified the cause as `genuine_hang` — the writer has stopped pipeline-wide. It was right on none. On 40 of them the writer had completed other records within the hour; on the other five the record had never entered the draft stage. The kickoff said "stuck draft," the prompt offered "genuine hang," and the model took the framing. The repair was four evidence rules written into the prompt and one fact the detector now computes itself and hands over: whether the writer completed anything, anywhere, in the last forty-five minutes.

**The framing.** The investigator's job on a stuck-draft ticket is to look at the record, the tenant's configuration, the locks, the recent event counts and the logs, and say what is wrong. For three months it said the same thing: the writer is down. It was never down. Read together, the forty-five tickets show the mechanism. The detector had already decided the record was stuck; the prompt's list of classifications included one for a pipeline-wide stall; and three signals that look like evidence of a stall are not — a zero event count for one tenant (queue position in a single-writer system), an absent lock row (the normal state between the writer's cycles), and a log read that failed or returned nothing (no signal at all). The investigator read each as confirmation.

**The four rules.** The fix shipped with the second detector fix on 7 September, and it is four rules in the system prompt, quoted here trimmed of their examples, tool names and classification directives, because they are the whole design:

```text
1. A zero event count for one tenant is not evidence the writer is down. [The writer] is a single worker draining a shared queue at 2 drafts per 15-minute cron, oldest-first, so a queued record has zero draft_started/draft_completed events until the writer physically reaches it. That is queue position, not a hang.

2. An absent lock is not evidence of anything. [The lock] is a ~20-minute lease held only while an invocation is running. Between the 15-minute cron fires there is legitimately no lock row.

3. A failed or empty tool read is "unknown", never a diagnosis. If [the log tool] errors, is denied, or returns nothing, you have no signal — you do not have a negative signal. Classify unknown … and say plainly in the Evidence section which read failed and what it would have told you.

4. The kickoff for stuck_draft carries "Writer completions in window". The detector already ran the global check for you: 0 means the writer finished nothing anywhere (a real global-stall signal); a non-zero value means the writer is draining other tenants and this specific record is the one not advancing — investigate the record, not the worker.
```

The fourth rule is the important one: it moves the one question answerable without a model into the detector.

Two tickets have been written under these rules, on the only two real stalls since. The first — the first ticket written under the rules — classified the cause as unknown, reported that its own re-check of the fleet-wide count disagreed with the kickoff's, said in its own words that this was a record-specific stall and not a pipeline-wide one, and recommended a person look at the record. The second did the same at low confidence, named the log read that had failed and what it would have shown, and said it was not inferring anything from the failure.

Honest is not the same as right. Each concluded the writer was alive from a completion an hour or so earlier. In the first case the record says the scope was wrong: all three records then in draft had stalled together, on a throttled account, and were parked together three hours after the ticket — a writer-wide stall that the ticket called record-specific. In the second, the completion the investigator cited was a `draft_completed` event for the stuck record itself — the record that finished without persisting content — while the writer's last real completion was more than three hours before the ticket, and it completed nothing for nearly eight hours after; the ticket's evidence was honest about what it read and wrong about what the read meant. Those are the first two tickets in the record whose evidence sections a person could act on without first checking whether the premise was true.

---

## 8. Only Evidence Closes a Ticket

> **Answer:** A ticket that nobody can close stops being read, which defeats the ticket. On the CI workload the fix is two rules decided in code: one open ticket per repository, workflow and branch, so a workflow that fails every day does not file every day; and a ticket closes only on evidence — the workflow green again at or after the ticketed run, the branch gone, or the repository archived — with a dormant pull-request branch closing as cancelled after fourteen days.

**The lifecycle.** The CI workload taught the companion lesson, about what happens after a ticket is filed. The GH Triage Agent investigates every failed workflow run across the organization's repositories; its detector locks per run, so a workflow that failed every day filed a ticket every day — thirteen for one workflow in July — and nothing ever closed one. When the repair was written on 3 October, 69 machine-filed tickets sat open — the 63 of section 5 plus the CI investigations filed earlier that day. What they were, and what became of them:

| Open on 3 October | Count | Closed by the rules | Closed by a person |
|---|---|---|---|
| Pipeline tickets (35 about records that had since published) | 38 | — (the rules cover CI) | — |
| CI: workflows since green, or repositories since archived | 7 | 7 | 0 |
| CI: pull-request branches nobody had touched | 5 | 5, as dormant | 0 |
| CI: one drift gate that fails by design and never meets the rule's definition of recovery | 13 | 0 | 13 |
| CI: filed that day, before the repair | 6 | 4 | 2 |



The repair is the two rules in the block above, and two properties of them matter in practice. A new failure in a group that already has an open ticket is folded into it — occurrences, last seen, last run — and starts no investigation, so the queue cannot grow faster than the number of distinct failing workflows. And anything the rules cannot match stays open, and the whole pass fails open: an error in it costs hygiene, never a detection. In the sixteen days after it shipped, the rules closed 22 tickets, every one announced in the channel: the sixteen in the table on 3 October in a single pass, and six more through 17 October. The sixteen closed by a person — the drift gate's thirteen and three from 3 October — were ones the rules correctly left open: a gate that fails by design never recovers by the rule's definition, and leaving it to a person is what the rules are for. The closes and fold-ins are the one place Sentinel updates a ticket row — deterministic code in the detector's pass, CI workload only.

**What it adds.** What the investigator adds over the detector and a person is, on this record, two things. The first is the evidence section — the named failed read, the counts it did run, the record's own timestamps — which turns an alert into something a person can act on in a minute instead of reconstructing in twenty; the median time from detection to a filed ticket across the 53 pipeline tickets was fifty seconds, so the latency a person sees is the detector's cadence, not the investigation. The second is visible on the CI side, where the repaired workload's fourteen tickets since 3 October carry the investigator's own recommendation — five to investigate further, five that needed a fix, three to rerun and one known flake — a sorting a person would otherwise have done by opening fourteen workflow runs. The value claimed is the sorting, not its accuracy: the labels are the investigator's and unadjudicated, and the only outside check is that ten of the fourteen were closed by the rules on evidence and three more by a person. Why keep a model in the loop at all, when the detector's global check and the lifecycle rules are deterministic? Because a detector can only ask the question it was built to ask. The model's job is the part no rule anticipated — reading the record's own timeline, naming the read that failed, saying which of the kickoff's claims its own re-check could not corroborate — and whether that is worth a tenth of a dollar a session is the hypothesis the next record tests.

---

## 9. Self-Reported Confidence Carried No Usable Information

> **Answer:** Every ticket template asks the investigator for a confidence of high, medium or low. On 49 pipeline tickets, high confidence was right 0 times in 14 and medium 2 in 34; on the first 30 CI tickets the confidence was high on all 30, four of them classified `unknown`. It ran opposite to correctness in one workload — on a count small enough that the low cell is a single ticket — and was constant in the other. Nothing in the code keys on the field, nothing in the gate depends on it, and after this record nothing should; if a future template drops it, nothing changes.

The confidence field was in the template from the first workload, because it is in every incident template written by humans, and the assumption was that the investigator's confidence would at least rank its own tickets. The record says otherwise. In the pipeline workload the fourteen tickets marked high were all false positives; the three real stalls were marked medium, medium and low. In the CI workload the field never moved: thirty tickets, thirty highs, four of them on tickets that classified the cause as unknown — high confidence in not knowing. Since the October lifecycle change the field has started to vary (ten high, four medium in the fourteen tickets since), and three of those tickets qualified their own number in prose — one of them verbatim: "high (on the cause of the failure; the classification label is a forced fit)". That is a more honest sentence than the field it annotates.

This is consistent with what is measured elsewhere. Xiong and colleagues' [empirical evaluation of confidence elicitation](https://arxiv.org/abs/2306.13063) found, in a 2023 evaluation, that language models asked to verbalize their confidence tend to be overconfident, and that the mitigations the authors test — human-style prompting, agreement across several sampled responses, better aggregation — can reduce it, and none is consistently best. Zhao and colleagues' [*Wired for Overconfidence*](https://arxiv.org/abs/2604.01457) locates the inflation in identifiable circuits — a small set of components in the middle-to-late layers that push the stated certainty up at the final token — which is a mechanistic way of saying the number is a product of the model, not a reading of it. OpenRCA 2.0 does not measure self-reported confidence at all, and none of the other root-cause studies cited in section 1 reports it in its abstract, which is why our record, small as it is, is worth reporting.

The design consequence is modest and already in place. The field stays as a reading aid.

---

## 10. Cost as a Property of the Caps

> **Answer:** The cost of an investigate-only agent is bounded by construction before it is measured. The caps of section 4 bound a session; a per-tick cap bounds a cycle; a lock bounds how often one candidate can be investigated. Their product is the worst case, and it is arithmetic: about $4,800 a day across three workloads if every tick hit its cap and every session its ceiling, a figure the locks cut to the number of distinct candidates a day and a redelivery term bounds separately. The measured case is about a tenth of a dollar a session.

[*The Finite Chain*](https://isimplifyme.com/whitepapers/the-finite-chain) makes the argument for multi-agent systems that cost should be a property of the topology rather than a quantity to be monitored. The same holds for a single bounded agent, and the Sentinel caps are where the property lives: a reviewer can read the caps in sections 2 and 4 and compute the worst day without trusting anyone's estimate.

The arithmetic belongs on the page. Ten sessions a tick at a dollar each is $960 a day for a fifteen-minute workload and $2,880 for the five-minute one. The locks cut that down: a candidate can be investigated once a day on two of the workloads and once an hour on the third, so the detector-side bound is the number of distinct failing runs, stuck records and failing tenants the fleet produces in a day, at a dollar each.

The one term the locks do not touch is redelivery (section 4): a job that throws on every attempt is re-run every five minutes until its message expires after four days, each attempt re-running every job in the batch, up to ten sessions at a dollar each — 288 attempts a day, so about $2,880 a day for one bad batch of ten, which is the price of having no dead-letter policy and a visibility window at the function's own timeout rather than six times it, and a path the log shows taken once.

At today's list price, five months of sessions is about ten dollars of model spend; at the ceiling it would have been under a hundred. We do not quote a monthly platform figure here, because the right way to state one is from the billing record with the workload's cost-allocation tag, and that read belongs to the day it is made rather than to a paper with a citation window.

---

## 11. Limitations

> **Answer:** This is one fleet on one cloud with one model family, and three workloads, one of which has an empty record by configuration. The precision history rests on 49 tickets and the confidence finding on 49 plus 44; both are small. The adjudication is mechanical against the pipeline's own records, not an independent label, and for two of the three real stalls the record does not say whether a person intervened. Two tickets written under the new prompt rules are an encouraging start, not a before-and-after study.

**One fleet, one cloud, one family.** Everything measured here ran on one organization's AWS account against one model family, through one serving path. The structural claims — pure detection, conditional-write locks, cap-exempt output tools, evidence rules in the prompt — should transfer, because they are properties of the code rather than of the model. The numbers should not be assumed to.

**Three workloads, one of them silent.** The Diagnostics Agent ran once, on 4 May, against our own site at launch. Tenants opt in; none has, so under the activation condition of section 2 its production record is that one ticket: more than 45,000 detector ticks, and zero candidates in the detector logs that survive. That is a configuration choice; the paper makes no claim about that workload beyond its caps.

**A small record.** Forty-nine stuck-draft tickets across three detector eras, and forty-four CI tickets, are enough to see a precision history and a confidence pattern, and not enough for a confidence interval anyone should quote. The era-C cell is two tickets.

**The adjudication is ours, and it measures precision only.** Each stuck-draft ticket was judged by four rules, fixed before the adjudication was run, against the record's stage history and the fleet's writer activity, and the scripts that reproduce every cell are kept with the record. Those are the pipeline's own records, read by the people who run it. Recall is unmeasured: a stall that never produced a ticket is invisible to this method, and the forty-five-minute suppression rule lowers recall by design, since a record-specific stall is suppressed while the writer drains other tenants until the twelve-hour ceiling. Session-level logs are retained for thirty days, so the tickets, not the transcripts, are the durable record before 19 September; the earlier window survives as a summary.

**No recorded action on a ticket.** The record holds no case where a person is recorded acting because of a ticket: one real stall was parked by hand, with no note of what the person read first, and the other two resolved with no recorded intervention. The value this paper claims for the investigator — the sorting and the evidence section — is value in the ticket, not yet a measured change in what a person did.

---

## Companion Papers

This is a production account inside a reference-architecture series across the layers of an LLM-native system:

- [**Layer 1: The Model Is a Dependency With a Retirement Date**](https://isimplifyme.com/whitepapers/layer-1-model-selection-and-routing) *— shipped.* Frontier model selection and routing: why we don't train, why we route per task, and how the architecture stays model-version-independent.
- ***Layer 2: The Detector Sets the Precision*** *— this paper.* Investigate-only agents: the typed registry, the generic runner, conditional-write locks, cost ceilings, and the production record.
- [**Layer 3: Data + Retrieval**](https://isimplifyme.com/whitepapers/layer-3-data-retrieval) *— shipped.* Pipelines, permissioned retrieval, hybrid search, context engineering, memory, and feedback loops.
- [**Layer 4: Reliability Engineering for Regulated AI**](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) *— shipped.* Guardrails, atomic integrity, investigate-only audit, circuit breakers, retries, and quality gates. Section 4 is the five-paragraph statement of the pattern this paper opens up.
- [**Layer 5: Multi-Tenant Business Integration**](https://isimplifyme.com/whitepapers/layer-5-business-integration) *— shipped.* Single-table multi-tenancy, domain routing, the unified lead pipeline, permissioned dashboards, and synchronized billing.
- [**The Finite Chain: Bounding Multi-Agent Autonomy by Construction**](https://isimplifyme.com/whitepapers/the-finite-chain) *— shipped.* The multi-agent companion: containment by topology, and cost as a property of the graph.
- [**The Trust Ladder: Supervised Autonomy for AI Code Review**](https://isimplifyme.com/whitepapers/the-trust-ladder) *— shipped.* How one non-deterministic reviewer earns the authority to block a merge — the ladder an investigate-only agent has deliberately not climbed.
- [**The Rulebook Is Not the Guardrail**](https://isimplifyme.com/whitepapers/the-rulebook-is-not-the-guardrail) *— shipped.* Why the properties that matter are enforced outside the model.

Each paper stands alone; together they map the full stack of an LLM-native system in production.

---

## Conclusion

An investigate-only agent earns its place in production by what it cannot do, and five months of ours say the discipline paid, and not where we expected it to. We built the caps, the locks and the two-tool surface to bound the model's reach, and they did. What the record taught was about the other direction: the detector's question bounded the investigator's answer, the prompt's vocabulary became its conclusion, and its confidence in its own work told us nothing. Each of those was fixed in code, none in the model — which is the only place a reviewer can read it, and the only place it stays fixed when the model changes. The agent investigates. Someone else decides.

---

## Notices

**Not legal, compliance, or financial advice.** This paper is for informational purposes only. Architectural decisions in regulated workflows require qualified counsel and a formal review.

**Dated observations.** The production record in sections 3, 4 and 6 through 10 covers 28 April to 19 October 2026 and describes specific detector versions, prompt versions and model generations over that span. The pattern is the claim, not the numbers for any one period.

**Capabilities change.** AWS Bedrock service capabilities, model availability, inference-profile coverage, and the surrounding tooling evolve continuously; verify current state before implementation.

**Outside studies.** Figures attributed to external papers are quoted from the cited publications and have not been independently replicated by iSimplifyMe.

**Trademarks.** AWS, Amazon Bedrock, Amazon SQS, Amazon DynamoDB, Amazon EventBridge, and AWS Lambda are trademarks of Amazon.com, Inc. or its affiliates. Claude is a trademark of Anthropic, PBC. Slack is a trademark of Slack Technologies, LLC. GitHub is a trademark of GitHub, Inc. References are descriptive and do not imply endorsement.

---

**About the author.** Joseph W. Elstner is the founder and principal architect of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific. iSimplifyMe is bootstrapped, deploys production AI on AWS Bedrock, and runs a multi-tenant orchestration platform across healthcare, legal, financial, and editorial verticals.

**Contact.** ai@isimplifyme.com — for engineering teams deciding how an AI agent may be allowed near production, we offer an investigate-only design review at no cost.

**Cite this paper.** Elstner, J. (2026). *Layer 2: The Detector Sets the Precision.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/layer-2-investigate-only-agents
