> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/ai-spend-caching-and-model-routing](https://isimplifyme.com/whitepapers/ai-spend-caching-and-model-routing)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-06-28 · License: [CC BY 4.0](LICENSE)

# Keeping AI Spend Flat While Token Usage Grows: Caching and Model Routing on AWS Bedrock

A reference architecture for controlling production AI cost on AWS Bedrock — prompt caching, per-task model routing, cache-aware routing, cheaper defaults, and spend observability: the cost layer that holds spend flat as usage scales across an organization.

---

## Abstract

> **Answer:** You hold AI spend roughly flat as token usage grows by attacking both terms of the bill — the number of billable tokens and the price paid per token — rather than by capping usage. Prompt caching cuts billable tokens: a stable prompt prefix is reused across requests and read back at roughly a tenth of the input price, so each call pays full rate only on what is new. Model routing cuts price per token: each task runs on the cheapest model that can do it well, with frontier models reserved for genuinely hard reasoning. A cost-aware gateway applies both — plus failover, redaction, and per-call spend logging — across every team and tool at once.

This is a reference-architecture paper in the same series as *Layer 3: Data + Retrieval* and *Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock*. It documents the cost layer — the part of an AI deployment that decides whether spend tracks usage one-for-one, or whether usage can grow ten times while the bill stays roughly where it was.

The intended reader is technical and budget-accountable: a CTO, VP of Engineering, or the architect who has just watched an AI line item double in a quarter and been asked to explain it. The argument of the paper is that AI spend is an architecture decision, not a usage problem — and that the levers that control it are the same levers that make the deployment governable.

---

## 1. The Cost Curve Is an Architecture Decision

> **Answer:** AI spend rises with token usage by default, so the instinct is to slow usage down with caps and spend alerts. That is the wrong lever. Capping usage rations access and generates friction without changing the cost of a request. The engineering goal is to flatten the curve — hold spend roughly flat while usage grows — by lowering the cost of every request at the source. Two levers do most of that work: prompt caching and model routing. Holding spend flat as usage climbs is an infrastructure outcome, not a rationing one — and both already run in production on the multi-tenant AWS Bedrock platform iSimplifyMe operates.

Most AI bills grow the same way: a pilot ships, usage climbs, the line item climbs with it, and the first response is to add usage caps and spend alerts. This is the friction approach, and it has a predictable failure mode. The majority of users never approach their caps, so tightening caps generates alerts and support tickets without saving much — and the few power users who do hit the ceiling are usually the ones producing the most value.

The better path is to stop treating spend as a usage problem. Spend is the product of two numbers: how many tokens you are billed for, and the price you pay per token. Usage caps attack neither — they attack the *count of requests*, the one term you most want to grow. The architecture approach attacks the other two directly.

- **Prompt caching** lowers the *billable token count* of each request, by reusing a prefix you have already paid to process.
- **Model routing** lowers the *price per token*, by running each task on the least expensive model that can do it well.

Applied together, these two levers decouple spend from usage. That is the whole game: not fewer tokens used, but fewer tokens *wasted*, and a lower blended price on the tokens that remain. The rest of this paper is how each lever works, why they have to be designed together, and what the surrounding gateway looks like in production.

---

## 2. Prompt Caching: Pay Full Rate Only on What Is New

> **Answer:** Prompt caching reuses a stable prompt prefix across requests so you pay full input price only on the new tokens in each call. The mechanism is a prefix match: the model caches the rendered prompt up to a marked breakpoint, and any later request whose prefix matches byte-for-byte reads that span back at roughly a tenth of the input price. Cache reads cost about 0.1× the base input rate; cache writes cost 1.25× for a five-minute window or 2× for an hour, so a cached span pays for itself in as few as two requests. The biggest wins come from workloads with a large, stable prefix — a system prompt, retrieved context, and tool definitions — reused across many turns.

The economics are favorable enough that caching is the first thing to engineer, and the part most teams leave on the table.

| Token class | Relative price |
|---|---|
| Uncached input | 1.0× |
| Cache write — 5-minute window | 1.25× |
| Cache write — 1-hour window | 2.0× |
| Cache read | ~0.1× |

The break-even is fast. With a five-minute window, a span that is written once (1.25×) and read once (0.1×) costs 1.35× against the 2.0× you would have paid to process it uncached twice — so caching wins from the second request onward. With the one-hour window, the doubled write cost needs about three requests to pay off, but it keeps the span warm across gaps that the shorter window would let expire.

What makes caching work is also what breaks it: the prefix match is exact. The cache key is the rendered bytes up to the breakpoint, and a single changed byte anywhere in the prefix invalidates everything after it. Render order is fixed — tool definitions, then the system prompt, then the messages — so the discipline is straightforward to state and easy to violate:

- **Keep stable content first.** A frozen system prompt and a deterministic tool list belong at the front, before any breakpoint.
- **Keep volatile content last.** Timestamps, per-request identifiers, and the varying user question go after the final breakpoint, where they invalidate nothing ahead of them.
- **Watch for silent invalidators.** A current-date string interpolated into the system prompt, an unsorted JSON blob, or a per-user identifier early in the prompt will quietly drop the hit rate to zero with no error — just a bill that never improves.

Two further constraints matter in practice. There is a minimum cacheable prefix — roughly two to four thousand tokens depending on the model — below which a prefix silently will not cache. And in long agent turns that append many tool-call blocks, a breakpoint only looks back a bounded number of blocks for a prior cache entry, so very long turns need an intermediate breakpoint or the next request misses.

**One caveat specific to AWS Bedrock.** Prompt caching is supported on Bedrock, but the automatic caching available on the first-party API is not. On Bedrock the breakpoints have to be placed deliberately in each request. This is precisely why caching is an engineering engagement rather than a setting: the prefix has to be ordered, the breakpoints placed where the reusable span ends, and the silent invalidators kept out of the cached region. Done well, the same conversation that paid full rate on every turn now pays full rate only on the handful of new tokens each turn adds.

---

## 3. Model Selection: Match the Model to the Task

> **Answer:** Model routing lowers the blended price per token by running each task on the cheapest model that can do it well. The price spread across a model family is wide — a frontier model can cost several times what a smaller one does per token — and the capability spread is wide on hard reasoning but narrow on routine work. So the policy is to default to a smaller, cheaper model for execution, extraction, classification, and query rewriting, and reserve frontier models for planning and genuinely hard reasoning. Humans should not be picking a model per call; the gateway routes by task.

The price spread alone is large enough to move a bill materially. Approximate US rates for one current family, per million tokens:

| Model | Input | Output |
|---|---|---|
| Claude Opus 4.8 | $5 | $25 |
| Claude Sonnet 4.6 | $3 | $15 |
| Claude Haiku 4.5 | $1 | $5 |

A high-volume classification step that runs on the frontier model by default is paying up to five times what it needs to. The inverse error is just as real: routing a complex planning step to the cheapest model to save a few cents produces a worse plan, more retries, and a higher total cost. Routing is not "use the cheapest model" — it is "use the cheapest model that clears the bar for this task."

In our own work, model selection is per task by default rather than a tuning afterthought. Lightweight steps — the rewrite that turns a user question into structured retrieval queries, the classification that tags an inbound message — run on a small, fast model, while a step that synthesizes a regulated answer from retrieved context or plans a multi-step action runs on a heavier one. Our content pipeline enforces a per-run token budget and selects its model accordingly; our concierge runs across client sites on a mid-tier model chosen as the right default for that workload.

The organizing principle is cheaper defaults, not caps — which is the subject of section 5. Before that, the two levers have to be reconciled, because routing naively will quietly undo the savings from caching.

---

## 4. Cache-Aware Routing: The Two Levers Interact

> **Answer:** Prompt caching is per-model, so the two cost levers interact and must be designed together. When a conversation moves from one model to another, the warm cache built on the first model cannot be read by the second — the next request pays a cold cache-write premium instead of a cheap cache read. A router that scores each turn in isolation and sends it to whichever model looks cheapest will quietly raise spend by abandoning warm caches mid-conversation. The correct policy is cache-aware: keep a conversation on its model while the cache stays warm, and only re-route once the conversation goes quiet long enough for the cache window to lapse.

This is the part most cost-optimization efforts miss, and it is the reason caching and routing cannot be built by two teams who never talk.

Consider the naive router. It scores each turn on its own — this one looks simple, send it to the cheap model; the next looks hard, send it to the frontier model — which seems reasonable and is exactly wrong in a cached system. Each switch invalidates the per-model cache, so a conversation that bounces between models pays a fresh cold-write premium on every hop. The per-turn price looks optimized; the per-conversation bill goes up.

A cache-aware router weighs cache state alongside task difficulty. The default behavior is stickiness: a conversation stays on the model it started on for as long as the cache is warm, because the cached prefix is worth more than the per-turn price difference. The opportunity to re-route arrives only when the conversation goes idle long enough for the cache window to expire — at which point the warm prefix is gone anyway, and the router is free to pick the best model for what comes next with no cache penalty to pay.

The practical rule: route at conversation boundaries, not at turn boundaries. Within a warm conversation, the cheapest path is almost always to stay put and keep reading the cache.

---

## 5. Cheaper Defaults Without Rationing

> **Answer:** The lever that scales across an organization is the default model, not the usage cap. Because most users never approach their caps, tightening caps produces alerts and friction without much saving. Moving the default to a cheaper model — while leaving the frontier model one escalation away for tasks that need it — lowers spend across every request at once without rationing anyone's access. The goal is not to suppress usage; it is to build the infrastructure that lets usage grow sustainably.

Defaults are quiet and pervasive; caps are loud and narrow. A default is felt by everyone, on every request, without anyone having to think about it, while a cap changes behavior only at the ceiling. That asymmetry is why the default is the lever worth pulling.

The pattern: set the default model to the cheapest tier that handles the common case well, and make escalation to a frontier model automatic when the task warrants it rather than a decision the user has to make. Engineers and end users keep full access to the capable models; they simply stop paying frontier prices for work a mid-tier model does just as well. When usage grows, it grows mostly on the cheaper default, so the bill grows far slower than the token count.

This reframes the entire spend conversation. The question is no longer "how do we get people to use less," which fights the value of the system, but "how do we make each unit of use cost less," which compounds with adoption instead of fighting it.

---

## 6. The Reference Architecture: A Cost-Aware Gateway

> **Answer:** The architecture that makes these levers durable is a single gateway that all model traffic flows through. The gateway exposes one endpoint and one request format for many models, and applies routing, prompt-cache breakpoint placement, cross-provider failover, redaction, logging, and cost controls before any request reaches a provider. Because every team and tool calls the same gateway, an optimization made once — a better default, a fixed cache breakpoint, a new routing rule — applies everywhere at once, rather than being re-implemented per application.

A cost-aware gateway concentrates the optimization work in one place so it does not have to be repeated in every service. Its responsibilities:

- **One endpoint, many models.** A single request format the application targets, with the gateway translating to whichever provider and model the routing layer selects.
- **Routing.** Per-task model selection, cache-aware, as described in sections 3 and 4.
- **Caching.** Deterministic breakpoint placement and prefix ordering, so the cache actually hits — the work that Bedrock does not do automatically.
- **Failover.** Cross-provider fallback when a model is rate-limited or unavailable, so a provider hiccup degrades rather than fails.
- **Redaction and logging.** Sensitive fields stripped or masked before egress; every request logged for audit and cost attribution.
- **Cost controls.** Per-tenant, per-team, or per-user budgets enforced at the gateway, with usage recorded for the spend board described next.

The gateway is also where governance lives, which is why the cost layer and the compliance layer are the same architecture seen from two angles — the subject of section 8. Building these controls once, at the gateway, is the difference between an optimization that holds and one that erodes as each new application re-introduces the old defaults.

---

## 7. Observability: You Cannot Optimize a Bill You Cannot See

> **Answer:** The instrument that makes cost optimization real is per-call usage logging that separates uncached input, cache writes, cache reads, and output tokens, surfaced as a live spend board broken down by call, module, and user. The single most useful number on that board is the cache hit rate — cache-read tokens as a share of total input tokens. If it sits near zero across requests that should share a prefix, a silent invalidator is breaking the cache and the savings model has quietly collapsed. The spend board is both the optimization instrument and the trust signal: there is no black box, and the curve flattening is visible to the people paying the bill.

You cannot manage what you cannot measure, and AI spend is unusually easy to leave unmeasured because the bill arrives as a single monthly number with no attribution. The fix is to log usage at the call level and separate the token classes that price differently:

- **Uncached input** — full rate.
- **Cache writes** — the 1.25× or 2× premium paid to warm the cache.
- **Cache reads** — the ~0.1× tokens served from cache.
- **Output** — generated tokens.

With those four numbers per call, the spend board answers the questions that matter: which module is most expensive, which user or tenant drives the most cost, and — the headline metric — what the cache hit rate is. A healthy retrieval-augmented or agentic workload reads most of its input from cache; if the board shows mostly uncached input on requests that should be reusing a prefix, that is a defect to fix, not a cost to accept.

Our production platform logs exactly these four token classes per call and rolls them up by session, client, and model, with cost attached — the spend board made concrete, with cache hit rate as its headline number. The principle is that whoever pays for an AI system should be able to see, per call and per module, what it cost and why; that visibility is the proof there is no black box. The same investigate-only pattern we run in our Sentinel platform extends to spend: scheduled agents that watch for anomalies — a hit rate that drops after a deploy, a module whose blended price creeps up — and surface them for review rather than acting automatically. Detection is automated; remediation stays with a human.

---

## 8. Cost Control and Governance Are the Same Architecture

> **Answer:** In regulated and mid-market work, the gateway that controls cost also carries the governance the buyer requires. Every request logged and auditable, redaction applied before egress, a model-agnostic routing layer that survives model deprecation, and per-tenant isolation are cost-control features and compliance features at the same time. A regulated wealth-management or healthcare buyer does not have to choose between a cheaper bill and a defensible one — the same single chokepoint delivers both.

The reason the cost layer is worth building well in regulated work is that it is not only the cost layer. Every property that makes spend controllable also makes the deployment governable:

- **Per-call logging** is a cost-attribution tool and an audit trail. The question "what did this cost" and the question "what did the system do for this user on this date" are answered by the same records.
- **Redaction at the gateway** keeps sensitive fields out of provider requests — a privacy control that happens to live exactly where cost is measured.
- **Model-agnostic routing** removes single-vendor lock-in. When a model is deprecated or a better one ships, the routing layer re-points without an application rewrite. The same indirection that lets you route for price lets you route for portability.
- **Per-tenant isolation** at the gateway is a cost-control boundary and a data-isolation boundary at once.

We apply this discipline across a client base that runs from mid-market to large enterprise — many production AI applications on shared multi-tenant infrastructure, including large enterprise engagements where governance is the buyer's first concern: every model interaction auditable, sensitive data confined to a controlled boundary, no system held hostage to a single vendor's roadmap. For that buyer, the cost discipline arrives as a byproduct of the same architecture. The same per-call record set that attributes cost is the audit surface that proves the system is under control. That convergence is the point: in regulated work, the cheapest defensible architecture and the most defensible cheap architecture are the same building.

---

## Companion Papers

This paper sits in a planned series of reference architectures across the layers of an LLM-native system:

- [**Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock**](https://isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock) *— shipped.* Model isolation and compliance in a HIPAA workflow.
- [**Layer 3: Data + Retrieval**](https://isimplifyme.com/whitepapers/layer-3-data-retrieval) *— shipped.* Pipelines, permissioned retrieval, hybrid search, context engineering, memory, and feedback loops.
- ***Caching and Model Routing*** *— this paper.* The cost layer: prompt caching, per-task routing, cache-aware routing, cheaper defaults, and spend observability.
- [**Layer 4: Reliability Engineering for Regulated AI**](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) *— shipped.* Guardrail architecture, atomic-write pipelines, and the investigate-only audit pattern.
- [**Layer 5: Multi-Tenant Business Integration**](https://isimplifyme.com/whitepapers/layer-5-business-integration) *— shipped.* Single-network, multi-vertical client architecture and per-tenant isolation.

Models will keep getting cheaper and more capable on their own. The architecture that decides which model runs, what it reads from cache, and what every call costs is the part that compounds — and the part a serious firm owns.

---

## Conclusion

AI spend is not a usage problem to be rationed; it is an architecture decision to be engineered. The two levers that decouple spend from usage are prompt caching, which lowers the billable token count of each request, and model routing, which lowers the price paid per token — and the two have to be designed together, because caching is per-model and naive routing throws warm caches away. A cost-aware gateway concentrates routing, caching, failover, redaction, logging, and cost controls in one place so the optimization holds across every team and tool. Per-call observability turns the bill from an opaque monthly number into a live, attributable spend board, with cache hit rate as the headline metric. And in regulated work, every one of those controls is also a governance control — which is why the cost layer is worth building like infrastructure rather than bolting on like a discount. The result is the curve every engineering leader wants to show: token usage climbing, spend holding flat.

---

## Notices

**Not legal, compliance, or financial advice.** This paper is for informational purposes only. Architectural and cost decisions in regulated workflows require qualified counsel and a formal review.

**Implementation details vary.** The architecture here is a reference pattern. Production deployments adapt it to per-client constraints — provider preferences, existing infrastructure, workload mix, and team composition. Savings depend entirely on workload shape; a workload with no reusable prefix and no task variance has little to optimize. Any cost figure quoted in an engagement follows a measured audit, not a promise made in advance.

**Pricing and capabilities change.** Per-token prices, caching economics, minimum cacheable prefix sizes, and per-feature availability across AWS Bedrock and the first-party API reflect US rates as of mid-2026 and evolve continuously. Verify current state before implementation.

**Trademarks.** AWS and Amazon Bedrock are trademarks of Amazon.com, Inc. or its affiliates. Claude is a trademark of Anthropic, PBC. References are descriptive and do not imply endorsement.

---

**About the author.** Joseph W. Elstner is the founder and principal architect of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific. iSimplifyMe is bootstrapped, deploys production AI on AWS Bedrock, and runs a multi-tenant orchestration platform across healthcare, legal, financial, and editorial verticals.

**Contact.** ai@isimplifyme.com — for engineering teams whose AI spend is growing faster than they can explain, we offer a cost-architecture review at no cost.

**Cite this paper.** Elstner, J. (2026). *Keeping AI Spend Flat While Token Usage Grows: Caching and Model Routing on AWS Bedrock.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/ai-spend-caching-and-model-routing
