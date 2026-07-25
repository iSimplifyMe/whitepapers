> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/layer-3-data-retrieval](https://isimplifyme.com/whitepapers/layer-3-data-retrieval)** — it revises there first; this mirror follows.
> Joe Elstner, Founder, iSimplifyMe · Published 2026-05-06 · License: [CC BY 4.0](LICENSE)

# Layer 3: Data + Retrieval

A reference architecture for the data and retrieval layer of LLM-native AI systems on AWS Bedrock — pipelines, permissioned retrieval, hybrid search, context engineering, memory, and feedback loops — drawn from iSimplifyMe production deployments in regulated and mid-market work.

---

## Abstract

> **Answer:** Layer 3 is the data and retrieval layer of an LLM-native systems architecture: pipelines, permissioned retrieval, hybrid search, context engineering, memory, and feedback loops. It is the layer that decides whether a deployed AI system is reliable, defensible, and improvable. Frontier models and orchestration frameworks are commoditized; the data layer that decides what the model sees at inference, where it came from, who is allowed to see it, and how the system improves from production traffic is not. Layer 3 is the moat in production AI.

This is the second in a series of reference-architecture papers documenting the patterns iSimplifyMe deploys in production AI work. The first paper, *Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock*, covered model isolation and compliance. This paper covers the layer most "AI consultancies" skip — the layer that decides whether a deployed AI system holds up under production load and regulatory scrutiny, or decays into a demo.

The intended reader is technical: CTO, VP Engineering, Director of AI, or the architect tasked with deploying production AI in a regulated or mid-market workflow without earning a quality regression in the second quarter of operation.

---

## 1. The Five Layers

> **Answer:** The five layers of an LLM-native systems architecture are: (1) frontier models — Anthropic, OpenAI, Google, with model routing per task; (2) orchestration — multi-step workflows, agent loops, tool use, structured outputs; (3) data and retrieval — pipelines, permissioned retrieval, hybrid search, context engineering, memory, feedback loops; (4) infrastructure and reliability — logging, retries, queues, evaluation systems, cost and latency optimization; (5) business integration — APIs, internal tools, multi-tenant networks, billing. Firms that compound advantage own Layers 2 through 4 deeply, with Layer 3 as the load-bearing center.

A useful way to evaluate any LLM-native systems firm is to map their actual capability across the five layers:

- **Layer 1 — Frontier models.** Anthropic, OpenAI, Google. We do not train. Bedrock-hosted Claude is our default; per-task model routing for cost and capability fit.
- **Layer 2 — Orchestration.** Multi-step workflows, agent loops, tool use, structured outputs, memory hooks. Our Sentinel platform hosts a fleet of investigate-only Claude agents on this layer.
- **Layer 3 — Data + retrieval.** This paper.
- **Layer 4 — Infrastructure + reliability.** Logging, retries, queues, circuit breakers, evaluation systems, cost and latency optimization. Our layered regulatory guardrail architecture and atomic-write content pipeline are Layer 4 examples.
- **Layer 5 — Business integration.** APIs, internal tools, end-user workflows, multi-tenant networks, billing.

A firm that owns Layer 2 only ships wrappers. A firm that owns Layer 4 only ships infrastructure without intelligence. The firms that compound advantage own Layers 2 through 4 deeply, with Layer 3 as the load-bearing center.

Frontier models will continue to improve faster than any consulting firm can match. Architectures that depend on a specific model version are obsolete in 12 to 18 months. Architectures with strong Layer 3 are model-version-independent — when the model improves, retrieval quality, memory, and feedback loops compound the improvement; they do not block it.

---

## 2. Why Layer 3 Is the Moat

> **Answer:** Layer 3 is defensible because it is hard to demo (a good retrieval system looks the same as a bad one to a casual viewer; depth shows up in production at scale, in regulatory audits, and in the slope of quality over time), hard to copy (a competitor cannot replicate three months of cleaned, structured, permissioned data with provenance metadata, eval datasets, and feedback loops), and regulatory load-bearing (in healthcare, legal, and financial workflows the data layer carries five hard requirements that consumer RAG patterns ignore: per-role permissioning, source provenance, auditability, freshness SLAs, and conflict resolution).

Three properties make Layer 3 defensible in a way the layers above and below are not.

**It is hard to demo.** Layer 2 demos well — agents calling tools makes for a clean video. Layer 3 is a precision number. A good retrieval system *looks the same* as a bad retrieval system to a casual viewer. The depth of the work shows up in production at scale, in regulatory audits, and in the slope of quality over time.

**It is hard to copy.** A competitor can copy a prompt library in a day. A competitor cannot copy three months of cleaned, structured, permissioned data with provenance metadata, eval datasets, and feedback loops capturing real production corrections. Every week a Layer 3 system runs against real traffic, the moat widens.

**It is regulatory load-bearing.** In healthcare, legal, and financial workflows the data layer carries five hard requirements that consumer RAG patterns ignore: per-role permissioning, source provenance, auditability, freshness SLAs, and conflict resolution. A firm that has not solved these cannot deploy in regulated work. A firm that has solved them has a pattern that transfers across industries with minor configuration.

The framework summary: *models are commoditized; orchestration frameworks are commoditized; your cleaned, continuously improving data layer is not.* That is the load-bearing claim of this paper.

---

## 3. What Layer 3 Includes

> **Answer:** Layer 3 has six operational sub-pieces: (1) data pipelines — ingest, clean, deduplicate, version-tag with provenance metadata; (2) permissioned retrieval — tenant isolation as outer perimeter, role-based filtering as inner perimeter, enforced at retrieval not at presentation; (3) real retrieval systems — hybrid search (semantic plus keyword), metadata pre-filtering, query rewriting, second-pass reranking; (4) context engineering — chunking strategy, prioritization, citation injection, always-on regulatory blocks; (5) memory — session, user-scoped long-term with purge SLAs, operational audit trail; (6) feedback loops and eval — quality metrics (precision@k, recall, faithfulness), gold-standard eval datasets, continuous regression, user-feedback capture. Most AI consultancies do only the first.

A serious firm does all six sub-pieces; most "AI consultancies" do only the first.

1. **Data pipelines.** Pulling, cleaning, deduplicating, normalizing, and version-tagging data from databases, SaaS tools, document stores, and unstructured sources. Provenance metadata on every chunk: origin, timestamp, author, version, access tier.
2. **Permissioned retrieval.** Per-tenant isolation as the outer perimeter; per-role filtering as the inner perimeter. Permission enforcement at retrieval, not at presentation — the model never sees data the user is not authorized to see.
3. **Real retrieval systems.** Not "embeddings + top-k." Hybrid search (semantic + keyword), metadata pre-filtering, query rewriting, second-pass reranking.
4. **Context engineering.** Chunking strategy, context-window management, prioritization order in the prompt, citation injection, always-on context blocks for regulatory and safety reasons.
5. **Memory.** Short-term (current session), long-term (user-scoped with purge SLAs), operational (audit trail of every retrieval and serve).
6. **Feedback loops + eval.** Retrieval quality metrics, gold-standard eval datasets, continuous regression on every change, user-feedback capture, routing of feedback signals back into ranking and prompt construction.

The remainder of this paper covers each in production-pattern detail.

---

## 4. Reference Architecture

> **Answer:** The iSimplifyMe Layer 3 reference architecture on AWS Bedrock has eight stages: source systems (EMR, DMS, SharePoint, Drive, databases) → ingest and provenance layer (chunking, PII/PHI redaction, schema alignment, version tagging) → storage layer (S3 corpus and audit, DynamoDB memory and permissions, OpenSearch BM25, Bedrock embeddings) → permission gate (tenant scope, role filter, access tier, jurisdiction, recency) → retrieval pipeline (query rewrite, hybrid search, metadata filter, rerank) → context engineering (chunk priority, citation injection, always-on regulatory blocks) → Bedrock LLM with per-task model routing → audit and feedback (CloudWatch, S3 audit log, user feedback DDB, eval pipeline).

<style>
.layer3-flow { position: relative; margin: 32px 0 24px; padding-left: 64px; }
.layer3-flow::before { content: ''; position: absolute; left: 24px; top: 18px; bottom: 18px; width: 1px; background: #d4d4d4; }
.layer3-stage { display: grid; grid-template-columns: 96px 1fr; gap: 32px; align-items: start; padding: 48px 0; position: relative; border-top: 1px solid transparent; opacity: 0; transform: translateY(14px); animation: layer3-rise 700ms cubic-bezier(0.16,1,0.3,1) forwards; animation-delay: calc(80ms * var(--i, 0)); }
@keyframes layer3-rise { to { opacity: 1; transform: translateY(0); } }
@supports (animation-timeline: view()) { .layer3-stage { animation-timeline: view(); animation-range: entry 0% entry 60%; animation-delay: 0ms; } }
.layer3-stage + .layer3-stage { border-top-color: #d4d4d4; }
.layer3-stage:first-child { padding-top: 16px; }
.layer3-stage:last-child { padding-bottom: 16px; }
.layer3-dot { position: absolute; left: -52px; top: 56px; width: 14px; height: 14px; border-radius: 50%; background: #FAF9F5; border: 1.5px solid #737373; transition: background 360ms cubic-bezier(0.16,1,0.3,1), border-color 360ms cubic-bezier(0.16,1,0.3,1), transform 360ms cubic-bezier(0.16,1,0.3,1); }
.layer3-stage:first-child .layer3-dot { top: 24px; }
.layer3-num { font-family: var(--font-display, "jaf-bernino-sans-comp"), "Helvetica Neue", Arial, sans-serif; font-size: clamp(48px, 6vw, 64px); font-weight: 500; letter-spacing: -0.04em; line-height: 0.9; color: #c4c4c4; transition: color 360ms cubic-bezier(0.16,1,0.3,1); user-select: none; }
.layer3-content { min-width: 0; }
.layer3-tag { font-family: "IBM Plex Mono", ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 10.5px; letter-spacing: 0.1em; text-transform: uppercase; color: #737373; margin: 0 0 8px; }
.layer3-title { font-family: var(--font-display, "jaf-bernino-sans-comp"), "Helvetica Neue", Arial, sans-serif; font-size: clamp(22px, 2.4vw, 26px); font-weight: 500; letter-spacing: -0.015em; line-height: 1.15; margin: 0 0 8px; color: #0a0a0a; }
.layer3-desc { font-size: 15px; line-height: 1.55; color: #404040; margin: 0 0 12px; max-width: 56ch; }
.layer3-stack { font-family: "IBM Plex Mono", ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 11.5px; line-height: 1.5; letter-spacing: 0.04em; color: #737373; margin: 0; }
.layer3-stack b { color: #404040; font-weight: 500; }
.layer3-stage:hover .layer3-num { color: #404040; }
.layer3-stage:hover .layer3-dot { background: #737373; transform: scale(1.15); }
.layer3-stage.is-core .layer3-num { color: #EB1C23; }
.layer3-stage.is-core .layer3-dot { background: #EB1C23; border-color: #EB1C23; }
.layer3-stage.is-core .layer3-tag { color: #EB1C23; }
.layer3-stage.is-core:hover .layer3-num { color: #EB1C23; }
.layer3-stage.is-core:hover .layer3-dot { background: #EB1C23; transform: scale(1.2); }
.layer3-cap { font-family: "IBM Plex Mono", ui-monospace, SFMono-Regular, Menlo, monospace; font-size: 11px; letter-spacing: 0.08em; text-transform: uppercase; color: #737373; padding-left: 64px; margin: 24px 0 32px; max-width: 56ch; }
.layer3-cap b { color: #EB1C23; font-weight: 500; }
@media (max-width: 600px) { .layer3-flow { padding-left: 48px; } .layer3-flow::before { left: 16px; } .layer3-stage { grid-template-columns: 60px 1fr; gap: 16px; padding: 36px 0; } .layer3-num { font-size: 38px; } .layer3-dot { left: -42px; width: 11px; height: 11px; } .layer3-cap { padding-left: 48px; } }
</style>

<div class="layer3-flow">
<div class="layer3-stage" style="--i:0">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">01</div>
<div class="layer3-content">
<p class="layer3-tag">Pipelines</p>
<h3 class="layer3-title">Source Systems</h3>
<p class="layer3-desc">Where the corpus comes from. Structured databases, document stores, SaaS tools, unstructured files.</p>
<p class="layer3-stack"><b>EMR</b> · <b>DMS</b> · <b>SharePoint</b> · <b>Drive</b> · <b>Postgres</b></p>
</div>
</div>
<div class="layer3-stage" style="--i:1">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">02</div>
<div class="layer3-content">
<p class="layer3-tag">Pipelines</p>
<h3 class="layer3-title">Ingest + Provenance</h3>
<p class="layer3-desc">Chunk, redact, version-tag. Every record carries origin, timestamp, author, version, access tier.</p>
<p class="layer3-stack">Chunking · PII / PHI redaction · Schema alignment · Version tagging</p>
</div>
</div>
<div class="layer3-stage" style="--i:2">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">03</div>
<div class="layer3-content">
<p class="layer3-tag">Storage</p>
<h3 class="layer3-title">Storage Layer</h3>
<p class="layer3-desc">Tenant-scoped storage with both keyword and vector indexes. Permissions and memory live in DynamoDB.</p>
<p class="layer3-stack"><b>S3</b> corpus + audit · <b>DynamoDB</b> memory + permissions · <b>OpenSearch</b> BM25 · <b>Bedrock</b> embeddings KNN</p>
</div>
</div>
<div class="layer3-stage is-core" style="--i:3">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">04</div>
<div class="layer3-content">
<p class="layer3-tag">Permission Gate · Core</p>
<h3 class="layer3-title">Permission Gate</h3>
<p class="layer3-desc">Tenant scope, role filter, access tier, jurisdiction, recency — enforced at retrieval, not at presentation.</p>
<p class="layer3-stack">Tenant scope · Role filter · Access tier · Jurisdiction · Recency</p>
</div>
</div>
<div class="layer3-stage is-core" style="--i:4">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">05</div>
<div class="layer3-content">
<p class="layer3-tag">Retrieval · Core</p>
<h3 class="layer3-title">Retrieval Pipeline</h3>
<p class="layer3-desc">Query rewrite → hybrid search → metadata filter → rerank → top-K. The piece most "AI consultancies" skip.</p>
<p class="layer3-stack">Query rewrite · Hybrid (BM25 + KNN) · RRF fusion · Reranker</p>
</div>
</div>
<div class="layer3-stage is-core" style="--i:5">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">06</div>
<div class="layer3-content">
<p class="layer3-tag">Context · Core</p>
<h3 class="layer3-title">Context Engineering</h3>
<p class="layer3-desc">Chunk priority, citation injection, always-on regulatory blocks. Most AI failures are context failures.</p>
<p class="layer3-stack">Chunk priority · Citations · Disclaimer band · Regulatory block</p>
</div>
</div>
<div class="layer3-stage" style="--i:6">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">07</div>
<div class="layer3-content">
<p class="layer3-tag">Inference</p>
<h3 class="layer3-title">Bedrock LLM</h3>
<p class="layer3-desc">Per-task model routing. Architecture is model-version-independent — when models improve, the system inherits the lift.</p>
<p class="layer3-stack">Claude (Bedrock) · Per-task routing · Swappable</p>
</div>
</div>
<div class="layer3-stage" style="--i:7">
<span class="layer3-dot" aria-hidden="true"></span>
<div class="layer3-num">08</div>
<div class="layer3-content">
<p class="layer3-tag">Audit + Feedback</p>
<h3 class="layer3-title">Audit + Feedback</h3>
<p class="layer3-desc">Append-only audit log with replay capability. Feedback signal routes by failure type into ranking, prompts, and eval-set growth.</p>
<p class="layer3-stack"><b>CloudWatch</b> · <b>S3</b> audit log · <b>DDB</b> feedback · Eval pipeline</p>
</div>
</div>
</div>

<div class="layer3-cap">Stages <b>04 / 05 / 06</b> are where Layer 3 depth lives — the moat compounds here.</div>

The remainder of this paper details each component.

---

## 5. Pipelines

> **Answer:** The pipeline layer is the unsexy foundation: pulling, cleaning, deduplicating, normalizing, and version-tagging data from databases, SaaS tools, document stores, and unstructured sources. The atomic two-store pattern prevents silent failures by writing all required stores in the same atomic transaction (Tier A) and asserting parity at consumer build time (Tier C). Provenance metadata on every chunk — origin, timestamp, author, version, access tier — is non-optional in regulated work and load-bearing for retrieval quality. Schema evolution discipline tags every record with a version. PII and PHI redaction belongs at ingest, not at retrieval.

The pipeline layer is the unsexy foundation. If it is weak, everything downstream is unreliable regardless of model choice.

**The atomic two-store pattern.** A common silent-failure mode in content-and-AI systems: the pipeline writes one of two required stores, the build appears green, the page or retrieval returns a 404 or empty result. We hit this on iSimplifyMe's own blog system in May 2026 — the pipeline pushed a markdown file to the content directory but did not update the manifest TypeScript file, and the slug enumeration reads only the manifest. The result was a silent HTTP 404 for an hour and a half.

The fix shipped in two tiers:

- **Tier A** (apex-portal): the pipeline writes the manifest entry alongside the markdown file in the same atomic GitHub commit. New `buildBlogManifestEntry` and `insertBlogManifestEntry` functions; idempotent on retry.
- **Tier C** (consumer site): a content-manifest validator runs as a build step (before `next build`) and fails the build if any manifest entry is missing its markdown body — and `generateStaticParams` is derived exhaustively from the manifest, so the rendered slug set always matches it. Build fails loudly = no more silent 404s.

This is a generalizable pattern. Any pipeline writing to multiple stores needs both an atomic-write contract at the source (Tier A) and a build-time integrity guardrail at the consumer (Tier C). One alone is insufficient.

**Provenance metadata on every chunk.** Every record in the pipeline carries `{origin, timestamp, author, version, access_tier}`. This is non-optional in regulated work — the question "who wrote this and when" is asked by every audit and every litigation defense — but it is also load-bearing for retrieval quality. Recency-weighted ranking, conflict resolution between two sources, and authority filtering all require metadata to exist on the chunk.

**Schema evolution discipline.** Pipelines run for years. Schemas change. Without explicit versioning, a query that worked last quarter silently returns a different shape this quarter. Tag every record with a schema version and migrate forward in batches; legacy versions remain queryable until explicit deprecation.

**PII/PHI redaction at ingest, not at retrieval.** The cheaper architectural choice is to redact at retrieval — leave PHI in the corpus, filter on the way out. The correct choice is to redact at ingest. Redaction failures are easier to catch when the corpus itself is clean. Audit defense is stronger when the audit log shows the corpus never contained the unredacted data, only the redacted version.

---

## 6. Permissioned Retrieval

> **Answer:** Permissioned retrieval enforces access control at the retrieval layer, not at presentation. Tenant isolation is the outer perimeter: per-tenant S3 prefixes, per-tenant index aliases, tenant ID baked into the retrieval key, not a post-hoc filter. Role-based filtering is the inner perimeter for authenticated surfaces: an identity source (Cognito JWT, federated SSO from a CRM or upstream auth provider) provides a role claim that maps to a numeric `access_tier`; every chunk in the corpus carries an `access_tier` metadata field; retrieval pre-filters with `WHERE chunk.access_tier <= user.tier`. Public surfaces assert a default tier implicitly and rely on tenant isolation alone. The model never receives data the user is not authorized to see — so the model cannot leak what it never sees, and the audit log shows the retrieval-time decision rather than a presentation-time mask.

This is the Layer 3 sub-piece most "AI infrastructure" firms skip entirely.

**Tenant isolation is the outer perimeter.** In our Concierge platform, every retrieval is scoped to a single tenant. The infrastructure is multi-tenant; the data plane is not. A query to tenant A cannot retrieve a chunk from tenant B by construction — the tenant ID is part of the retrieval key, not a post-hoc filter. This is enforced at the storage layer (per-tenant S3 prefixes, per-tenant index aliases) so a permission bug at the application layer cannot leak across tenants.

**Role-based filtering is the inner perimeter** when authenticated surfaces require it. The mechanism has three layers:

1. **Identity source.** An authenticated surface provides a role assertion: a Cognito JWT with a `custom:role` claim, a federated JWT from an upstream CRM or SSO provider, or — for unauthenticated public surfaces — a static role baked into the deployed widget configuration. Every request carries one role claim, validated against the issuer's public key before any retrieval runs.
2. **Role-to-tier mapping.** A per-tenant table in DynamoDB normalizes vertical-specific role names into a numeric `access_tier` (e.g., `0=public, 1=intake, 2=paralegal, 3=attorney, 4=partner` for legal; `0=public, 1=front-desk, 2=scheduler, 3=clinician` for healthcare). Higher tiers see everything below. The mapping lives in DDB rather than in code so each tenant's role taxonomy can vary without redeployment.
3. **Chunk-level enforcement.** Every chunk in the corpus carries an `access_tier` metadata field, set at ingest. Retrieval pre-filters with `WHERE chunk.access_tier <= user.tier` before any similarity search runs. Defaults on ingest are the most restrictive tier — the system fails closed; new content is invisible until explicitly tiered down.

Public-facing Concierge surfaces (anonymous visitors with no auth) assert an `intake` or `public` tier implicitly and rely on tenant isolation as the only operative perimeter. Authenticated staff surfaces — clinical assistants behind SSO, paralegal-only knowledge bases, attorney case-research consoles — engage all three layers. The same retrieval pipeline serves both; only the role claim and the chunk filter change.

This filtering happens *at retrieval*, not at presentation. The model never receives data the user is not authorized to see. This matters because:

- The model cannot leak what it never sees.
- The audit log shows the retrieval-time decision, not a presentation-time mask.
- A compromised UI cannot escalate access through prompt injection — there is no privileged context in the prompt to be coerced out.

**Persona-aware scoping.** Beyond tenant + role, our concierge platform applies persona-aware retrieval: the persona configuration controls not just tone and identity but which subset of the knowledge base is in scope for the conversation. An editorial-desk concierge does not retrieve from clinical content even when both exist in the same tenant.

**Failure modes to engineer against.** Permission caches lag. Role escalations during a session create race conditions. Permission revocation must invalidate in-flight retrievals. Each of these is a separate test in the eval set.

---

## 7. Real Retrieval Systems

> **Answer:** Real retrieval looks like a search engine, not an embedding store. Hybrid search runs semantic embeddings (Bedrock Titan or Cohere via Bedrock) and keyword BM25 (OpenSearch) in parallel, then fuses the candidate sets via reciprocal rank fusion. Metadata pre-filtering applies hard constraints (jurisdiction, date, version, recency, access tier) before similarity search. Query rewriting turns user intent into multiple structured retrieval queries via a small fast Bedrock call. Reranking re-scores top-N candidates against the original query using a second-pass model (Cohere Rerank or a Bedrock-hosted cross-encoder). On regulated-content corpora, reranking lifts precision@5 by 10 to 25 percentage points over hybrid search alone.

The "vectors + top-k" pattern that ships in tutorials breaks under production load. The real-retrieval pattern looks more like a search engine than an embedding store.

**Hybrid search.** Pure semantic similarity (embeddings) fails on exact-match terms — drug names, statute numbers, ICD codes, license IDs, dollar amounts, proper nouns with low embedding density. Pure keyword search (BM25) fails on paraphrased intent. Hybrid search runs both in parallel and fuses the candidate sets:

- **Semantic** via Bedrock embeddings (Titan or Cohere via Bedrock). Captures concept matching, paraphrase, intent.
- **Keyword** via OpenSearch BM25. Captures exact match, named entities, structured terms.
- **Fusion** via reciprocal rank fusion or score-blending; tuneable weights per tenant.

**Metadata pre-filtering.** Before similarity search runs at all, the candidate pool is filtered on hard constraints: jurisdiction, date range, version, recency tier, access tier, source authority. A retrieval that is "right by similarity" but "wrong by jurisdiction" is wrong. Filtering pre-search is also dramatically cheaper than filtering post-search at scale.

**Query rewriting.** Raw user intent is rarely a good retrieval query. *"What does our policy say about expense reports for international travel"* rewrites into multiple structured queries: `["expense policy international travel", "travel expense reimbursement", "T&E policy"]`. A small fast Bedrock call (Haiku-class) does the rewrite; the cost is recovered many times over in retrieval quality.

**Reranking.** The top-N candidates from hybrid search are not necessarily the top-K best for the prompt. A second-pass reranker (Cohere Rerank, or a Bedrock-hosted cross-encoder) re-scores the candidate pool against the original query using full-text comparison. The shape of the lift on regulated-content corpora: precision@5 jumps 10 to 25 percentage points when reranking is added on top of hybrid search.

**Latency and cost budgets.** Each of the above adds latency. Targets the architecture is engineered for: p95 retrieval latency under 600ms end-to-end; per-query cost under $0.005 at the retrieval layer (model cost is separate). These budgets are non-negotiable for chat UX; a slow retrieval feels like a dead system.

---

## 8. Context Engineering

> **Answer:** Context engineering is the highest-leverage Layer 3 sub-piece — most AI failures are context failures, not model failures. Hierarchical chunking with parent-document linkage scores small chunks for relevance but delivers parent sections to the prompt for context. Prioritization order matters: authoritative sources, recent records, and tenant-specific rules go last in the prompt where models attend more. Citation injection ships every retrieved fact with a structured source pointer the model is instructed to reference. Always-on context blocks include regulatory compliance rules with explicit precedence over every other instruction in the prompt — without explicit defensive engineering, the model can drift into claiming licensure, inventing credentials, or speaking in first-person provider voice on any concierge-style tenant.

Context engineering is the highest-leverage sub-piece. *Most AI failures are context failures, not model failures.* The model is doing its best with what it can see. If what it sees is wrong, irrelevant, contradictory, or missing the load-bearing fact, the output will reflect that.

**Chunking strategy is not trivial.** Paragraph-level chunking loses cross-paragraph context. Section-level chunking introduces irrelevant content into every retrieval. The pattern that works: hierarchical chunking with parent-document linkage. Retrieval scores the small chunk for relevance, but the prompt receives the parent section for context.

**Prioritization order in the prompt.** Models attend more to context near the end of a long prompt. Authoritative sources, recent records, and tenant-specific rules go last. Background context, generic guidance, and disclaimers go earlier. This is empirically tuned per tenant; no firm formula.

**Citation injection.** Every retrieved fact ships with a structured source pointer the model is instructed to reference. This converts the model from "answers from training" mode to "answers from retrieved context" mode. It also produces audit-defensible output: every claim in the response can be traced to a specific source chunk.

**Always-on context blocks.** Some context is non-negotiable on every query. In regulated work this includes:

- A `<regulatory_compliance>` block stating identity rules, advice limits, and credential boundaries — with explicit precedence over every other instruction in the prompt.
- An emergency-routing block ensuring the model can recognize crisis signals and respond with the correct escalation path regardless of user phrasing.
- Disclaimer copy that surfaces in the UI before any user input.

In production regulated work the pattern ships in three layers:

1. **Prompt-level**: regulatory compliance block injected with precedence override.
2. **UX-level**: notice band rendered in the chat panel before the user types.
3. **Defensive-level**: when the input classifier blocks an injection attempt before the model sees it, the fallback string is itself truthful AI identification, not a bland hand-wave.

All three layers can ship in a single day on top of an existing system-prompt architecture. The failure modes the pattern closes — the model claiming licensure, inventing a credential, drifting into first-person provider speech — are reachable on any concierge-style tenant in production at any vendor without explicit defensive engineering. This is what context engineering looks like in regulated work.

**Structured + unstructured blending.** Tabular data and narrative text live in the same prompt. The pattern that works: render tables as compact markdown immediately preceded by a one-sentence schema description, with narrative context interleaved by relevance, not by source format.

---

## 9. Memory

> **Answer:** Memory in agentic systems is three things: session memory (current task, ephemeral), user-scoped long-term memory (preferences, history, prior decisions, with explicit purge SLAs that meet HIPAA, GDPR right-to-be-forgotten, or state SoL rules), and operational memory (the system's own audit trail of every retrieval and serve, append-only S3 storage). Operational memory is the litigation-defense layer: the question "what did the system do for this user on this date" must be answerable with full reconstruction, not approximation. iSimplifyMe hosts operational memory through Sentinel — an investigate-only AI ops layer that audits production AI workloads on EventBridge schedules, files structured tickets, and posts intelligent summaries. Investigate-only by design; no auto-remediation.

Memory in agentic systems is not one thing; it is three.

**Session memory.** The current task or conversation. State held in Redis or in the request lifecycle. Cleared when the session ends.

**User-scoped long-term memory.** Preferences, history, prior decisions. Stored in DynamoDB with explicit per-tenant purge SLAs that meet the relevant regulatory standard — HIPAA retention rules for healthcare, GDPR right-to-be-forgotten for European users, state SoL rules for legal work. Without explicit purge SLAs, long-term memory becomes a compliance liability.

**Operational memory.** The system's own audit trail. Every retrieval, every served output, every tool call. Stored in S3 with append-only semantics. This is the litigation-defense layer: the question "what did the system do for this user on this date" must be answerable with full reconstruction, not approximation.

We host operational memory through our Sentinel platform — an investigate-only AI ops layer that audits production AI workloads. Sentinel agents run on EventBridge schedules, observe production state, file structured tickets through a typed registry, and post intelligent summaries to operations channels. Investigate-only by design; no auto-remediation. The audit trail is the product.

The Sentinel pattern transfers to any retrieval-quality observability need: a Retrieval Audit Agent samples production sessions on a schedule, scores retrieval quality against a gold-standard rubric, and files tickets for low-quality retrievals. The rubric is the eval set — gold-standard query/expected-source pairs maintained per tenant by domain SMEs.

---

## 10. Feedback Loops + Eval

> **Answer:** Layer 3 feedback loops capture three retrieval quality metrics — precision@k (relevance of top-k), recall (coverage of all relevant chunks), and faithfulness (whether the response actually uses retrieved context) — measured continuously against gold-standard eval datasets owned by domain SMEs (clinician, attorney, financial advisor). Every retrieval-config change, every model upgrade, every prompt edit triggers a regression run before broadening. User feedback (thumbs up/down) routes by failure type: bad retrieval feeds reranker training; bad prompt construction feeds prompt updates; right retrieval with wrong response feeds model evaluation; unclear user intent feeds query-rewriter improvement. The feedback signal is too valuable to dump into a single bucket.

This is where the moat compounds.

**Retrieval quality metrics.** The non-negotiable three:

- **Precision@k**: of the top-k retrieved chunks, how many were actually relevant.
- **Recall**: of all relevant chunks in the corpus, how many made it into the top-k.
- **Faithfulness**: does the response actually use the retrieved context, or is it hallucinated.

Each is measured continuously against gold-standard eval sets, with regression alarms on every retrieval-config or model change.

**Gold-standard eval datasets.** Per-tenant, 30 to 50 query / expected-source pairs maintained by the domain SME (clinician, attorney, financial advisor, depending on vertical). Without SME ownership the eval set is hollow; with SME ownership it becomes the substrate the system improves against.

**Continuous regression in production.** Every retrieval-config change, every model upgrade, every prompt edit triggers a regression run against the eval set before broadening. A 2-percentage-point drop in precision@5 is a P1 ticket.

**User feedback capture.** A discreet thumbs-up/down affordance on responses, captured into DynamoDB and routed to the Sentinel Retrieval Audit Agent for periodic aggregation. Surfaces as both a quality metric and a corpus-improvement signal.

**Routing the feedback signal.** Different feedback types feed different downstream changes:

- "Wrong answer because of bad retrieval" → reranker training signal, eval-set growth.
- "Wrong answer because of bad prompt construction" → prompt update.
- "Right retrieval, wrong response" → model evaluation, possibly model swap or temperature tune.
- "User intent unclear" → query-rewriter improvement.

A serious Layer 3 system has the routing logic explicitly mapped. The feedback signal is too valuable to dump into a single bucket.

---

## 11. Auditability and Litigation Defense

> **Answer:** Auditability is the unspoken Layer 3 requirement in regulated work. Every retrieval must be reconstructable: raw user query, rewritten queries, candidate set with similarity scores, metadata filter decisions, reranking scores, final assembled context, model output, and any feedback signal captured. Stored append-only in S3 with per-tenant prefix isolation. Retention configured per regulatory standard: HIPAA at 6+ years, state legal SoL is jurisdiction-specific, GDPR requires deletion on request. Replay capability via investigate-only agents that read the audit corpus without touching production state. A firm that cannot reconstruct a past retrieval cannot defend a past output.

Every retrieval reconstructable. This is the unspoken Layer 3 requirement in regulated work.

The question every regulator and every litigation defense eventually asks is the same: *what did the system say to this user, on this date, based on what context, and how do we prove it.* A system that cannot answer with full reconstruction has no defense. A Layer 3 system in regulated work logs:

- The raw user query.
- The rewritten queries (after query-rewriter).
- The candidate set returned by hybrid search, with scores.
- The metadata filter decisions.
- The reranking scores.
- The final context assembled.
- The model output.
- Any feedback signal captured.

Stored append-only in S3 with per-tenant prefix isolation. Retention configured per regulatory standard: HIPAA at 6+ years, state legal SoL is jurisdiction-specific, GDPR requires deletion on request. Replay capability via investigate-only agents that read the audit corpus without touching production state.

A firm that cannot reconstruct a past retrieval cannot defend a past output. This is Layer 3, not optional.

---

## 12. The Buyer's Five Questions

> **Answer:** To evaluate an AI firm's Layer 3 depth, a buyer should ask: (1) How do you measure retrieval quality? — clean answers name precision@k, faithfulness, eval sets, and cadence; (2) How do you handle stale or conflicting data? — clean answers describe versioning, recency tagging, conflict-resolution rules, and propagation SLAs; (3) What is your approach to permissions and access control? — clean answers enforce at retrieval not presentation, with tenant + role + access-tier as separate dimensions; (4) How do you improve the system over time? — clean answers describe feedback capture, routing logic, eval-set growth, and regression discipline; (5) Can you reconstruct exactly what was retrieved for any past query? — clean answers point to an append-only audit log with replay capability. A firm that scores cleanly on all five is at Layer 3 depth.

If you are evaluating an AI firm, the following five questions surface Layer 3 depth quickly. Lack of a clean answer to any one of them is a signal that the firm is selling Layer 2 wrappers as if they were a system.

1. **How do you measure retrieval quality?** A clean answer names specific metrics (precision@k, faithfulness), specific eval sets, and specific cadence. A weak answer talks about "the model" or "users seem happy."
2. **How do you handle stale or conflicting data?** A clean answer describes versioning, recency tagging, conflict-resolution rules, and propagation SLAs from source-of-truth update to retrieval-cache invalidation. A weak answer waves at "we re-index periodically."
3. **What is your approach to permissions and access control?** A clean answer enforces at retrieval, not at presentation, with tenant + role + access-tier as separate dimensions. A weak answer relies on UI filtering or "the user can only see their own data."
4. **How do you improve the system over time?** A clean answer describes feedback capture, routing logic, eval-set growth, and regression discipline. A weak answer says "we iterate on the prompt."
5. **Can you reconstruct exactly what was retrieved for any past query?** A clean answer points to an append-only audit log with replay capability. A weak answer says "we have logs" or pivots to "we don't store user data" (which is the wrong answer if you are in regulated work).

A firm that scores cleanly on all five is at Layer 3 depth. A firm that scores on Layer 2 only is the wrapper risk this paper exists to make visible.

---

## Companion Papers

This paper is part of a series of reference architectures across the five layers:

- [**Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock**](https://isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock) *— shipped.* Layer 1 + Layer 4 isolation patterns.
- ***Layer 3: Data + Retrieval*** *— this paper.*
- [**Layer 4: Reliability Engineering for Regulated AI**](https://isimplifyme.com/whitepapers/layer-4-reliability-engineering) *— shipped.* Layered guardrails; atomic content integrity; investigate-only audit agents; circuit breakers; bounded retries; deterministic quality gates. (The cost-and-latency half — caching and model routing — is a companion paper.)
- [**Layer 5: Multi-Tenant Business Integration**](https://isimplifyme.com/whitepapers/layer-5-business-integration) *— shipped.* Single-table multi-tenancy; domain-routed tenant resolution; the unified lead pipeline; permissioned client dashboards; synchronized Stripe billing; per-tenant CRM integration.
- ***Layer 2: Investigate-Only Agents*** *— forthcoming.* The Sentinel pattern in detail; typed agent registry; SQS-driven generic runner; atomic conditional-write locks for race protection; cost ceilings at the agent level.
- ***Layer 1: Frontier Model Selection and Routing*** *— forthcoming.* The shorter paper. Why we don't train, why we route per task, and how the architecture stays model-version-independent.

A serious firm does Layers 2 through 4 deeply. The series documents what that looks like in practice.

---

## Conclusion

Layer 3 is the load-bearing center of LLM-native systems. The model layer is commoditized — Anthropic, OpenAI, and Google will continue to ship better foundation models faster than any consulting firm can match. The orchestration layer is commoditized — agent frameworks, tool-use libraries, and workflow engines are open source and converging on the same shapes. The data layer that decides what the model sees, where it came from, who is allowed to see it, and how the system improves from production traffic — that layer is not commoditized, and the firms that compound advantage are the firms that own it deeply.

This paper documents the iSimplifyMe Layer 3 reference pattern. The companion papers in the series cover Layers 4, 5, 2, and 1, and the existing healthcare paper covers a vertical case study at Layers 1 and 4. A serious firm does Layers 2 through 4 deeply. The series documents what that looks like in practice.

---

## Notices

**Not legal or compliance advice.** This paper is for informational purposes only. Architectural decisions in regulated workflows require qualified counsel and a formal compliance review. References to HIPAA, GDPR, state-level health-data and SoL statutes, audit-log retention requirements, and BAA terms are summaries of operational understanding and not authoritative interpretations.

**Implementation details vary.** The architecture in this paper is a reference pattern. Production deployments adapt the pattern to per-client constraints — vendor preferences, existing data infrastructure, workload mix, budget, and team composition. Operational numbers (latency budgets, retrieval lift percentages, cost ranges) describe typical engagements and are not guarantees.

**Architecture and pricing change.** Pricing reflects US AWS Bedrock rates as of May 2026 and may change. Model availability, IAM action names, AWS service capabilities, and the AWS HIPAA-eligible services list evolve continuously; verify current state before implementation.

**Trademarks.** AWS, Amazon Bedrock, Amazon OpenSearch Service, AWS Lambda, DynamoDB, S3, KMS, CloudWatch, and Cognito are trademarks of Amazon.com, Inc. or its affiliates. Claude is a trademark of Anthropic, PBC. Cohere is a trademark of Cohere Inc. Pinecone is a trademark of Pinecone Systems Inc. LangChain and LlamaIndex are trademarks of their respective owners. References are descriptive and do not imply endorsement.

---

**About the author.** Joe Elstner is the founder of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific (Melbourne). iSimplifyMe is bootstrapped, deploys production AI on AWS Bedrock, and runs a multi-tenant orchestration platform across healthcare, legal, financial, and editorial verticals.

**Contact.** ai@isimplifyme.com — for engineering teams evaluating Layer 3 architecture, we offer a retrieval architecture review session at no cost.

**Cite this paper.** Elstner, J. (2026). *Layer 3: Data + Retrieval.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/layer-3-data-retrieval
