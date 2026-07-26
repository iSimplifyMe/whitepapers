> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock](https://isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-05-05 · License: [CC BY 4.0](LICENSE)

# Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock

A reference architecture for deploying production AI inside HIPAA-regulated workflows, drawn from our work building healthcare AI infrastructure on AWS Bedrock and SageMaker.

---

## Abstract

> **Answer:** A private LLM architecture for mid-market healthcare runs every model inference inside the practice's own AWS account through Bedrock or SageMaker, with no data leaving the VPC, no traffic to public AI APIs, and no PHI ever entering a third-party training corpus. Bedrock provides foundation-model inference with a Business Associate Agreement, zero-data-retention configuration, and IAM-scoped access. SageMaker handles vision workloads (X-ray segmentation, intra-oral photo analysis) where custom models outperform foundation models on clinical pixels. The pattern works because every component — inference, storage, audit logging, identity — is an AWS service that already operates under HIPAA-eligible terms, removing the BAA-stitching problem that breaks public-API deployments.

This paper describes the architecture we deploy for healthcare workloads, the engineering tradeoffs that shape model selection and isolation, and the implementation roadmap from kickoff to production. Section 4 grounds the abstract pattern in a production deployment pattern — including imaging pipelines, OSCC (oral squamous cell carcinoma) screening, SOAP note generation, voice dictation, and AI-driven phone agents running on Bedrock Opus 4.6, Sonnet 4.6, and SageMaker Serverless endpoints. Section 7 is a 12-week deployment template engineering teams can lift directly into a planning doc.

The intended reader is technical: CTO, VP Engineering, CIO, CISO, Director of Clinical Informatics, or the architect tasked with putting AI into a regulated workflow without earning a compliance finding.

---

## 1. Why public LLM APIs fail healthcare

> **Answer:** Public LLM APIs (OpenAI ChatGPT, Anthropic Claude direct, Google Gemini) fail healthcare deployments on four axes: BAA exposure (most public APIs do not sign a Business Associate Agreement at standard tiers), training-data leakage (default terms of service permit retention and model improvement on submitted content), data residency (cross-border routing breaks state-level health-data requirements like California CMIA), and audit-trail gaps (no native CloudWatch-equivalent logging of exact request and response bodies). A healthcare deployment that sends PHI to a public API is a HIPAA breach by default, irrespective of how careful the prompt engineering is.

We see four failure modes consistently in practices that tried public APIs first:

1. **The BAA-stitching problem.** OpenAI, Anthropic, and Google offer BAA-eligible enterprise tiers, but they require dedicated agreements, minimum spend commitments, and contract review cycles measured in months. Mid-market practices (10–200 providers) do not clear those minimums. The default consumer tier — which is what most engineering teams reach for during prototyping — does not sign a BAA.

2. **The terms-of-service training-data clause.** Default consumer ToS for public APIs reserve the right to use submitted content for model training, evaluation, and abuse-detection logging. Even if the customer disables training in account settings, the abuse-detection retention window (typically 30 days) means PHI sits in third-party systems by default. A breach disclosure under §164.408 would be unambiguous.

3. **Data residency and CMIA.** California's Confidentiality of Medical Information Act requires patient information to remain under the practice's control. A request that traverses a public API endpoint outside California — and especially one that crosses national borders for inference — is a CMIA violation even when no breach occurs. Texas, New York, and Illinois have analogous statutes with different geographic constraints.

4. **The audit-trail gap.** HIPAA Security Rule §164.312(b) requires an audit control mechanism that records access to electronic PHI. Public APIs do not surface request and response bodies to the customer's logging infrastructure. The practice cannot answer the question "what did our AI say to this patient on this date?" because the only canonical record sits inside the third party.

Private LLM architecture solves all four by keeping inference inside the practice's AWS account.

---

## 2. The reference architecture

> **Answer:** The reference architecture has six layers: (1) a VPC-isolated AWS account dedicated to the practice's AI workload, (2) Bedrock for foundation-model inference with cross-region inference profile routing in `us-east-1`/`us-east-2`/`us-west-2`, (3) SageMaker Serverless endpoints for vision and custom-trained models, (4) Lambda for application logic and tool dispatch, (5) DynamoDB for state and audit, and (6) KMS-encrypted S3 for raw imaging and document storage. IAM enforces least-privilege execution per function. CloudWatch captures every Bedrock and SageMaker invocation as a structured log event. Cognito handles staff and patient identity in separate user pools.

### 2.1 Account isolation

Each healthcare client gets a dedicated AWS account, not a shared multi-tenant deployment. Account-level isolation is the cleanest blast-radius boundary AWS offers, and it removes an entire class of cross-tenant data-leakage risks before architecture review begins. Cost: about $50–$200 per month in baseline overhead (CloudTrail, GuardDuty, Config). For a practice spending $5,000+ per month on inference, baseline overhead is rounding error.

### 2.2 Bedrock with inference profiles

We deploy Bedrock with US-region inference profiles (the `us.<model-id>` form), not the global routing variant. Two reasons. First, data residency for regulated clients — keeping inference within `us-east-1`, `us-east-2`, and `us-west-2` matches the geographic posture HIPAA-covered entities expect, and we have the audit trail to prove it. Second, AWS Partner and SOC 2 posture — when we submit case studies for the Generative AI Competency badge, "all inference within US AWS regions" is a clean line that does not require footnotes.

Direct foundation-model invocation (without an inference profile) is rejected by Bedrock for newer Claude models with `ValidationException`. Inference profiles are not optional for Claude Sonnet 4.6 or newer; they are the way Bedrock routes capacity across regions while preserving the customer-region contract.

### 2.3 SageMaker for vision and custom models

Foundation models on Bedrock are excellent at language tasks. They are mediocre at clinical pixels. For X-ray segmentation, intra-oral photo analysis, and any task where a fine-tuned domain-specific model outperforms a general one, we deploy SageMaker Serverless endpoints. SageMaker Serverless billing is per-millisecond-of-inference, which fits the bursty traffic profile of clinical imaging — a busy practice runs hundreds of inferences during morning hangouts and close to zero overnight.

A typical dental deployment runs three production SageMaker endpoints: tooth detection, smile inpainting, and tooth segmentation. The detection model is a YOLOv8m architecture across 32 dental classes at the 0.7 mAP50 range, retrained quarterly on consented practice data. The retraining pipeline ingests labeled images from each practice's own S3 bucket and produces practice-specific fine-tuned variants for opt-in workflows.

### 2.4 Lambda for tool dispatch

Application logic — the layer that decides which Bedrock or SageMaker call to make, which knowledge base to retrieve from, and how to assemble the response — runs in Lambda. We use the AWS SDK's Bedrock Runtime client (specifically `ConverseStreamCommand` for streaming agentic patterns) and route tool calls back to handler functions in the same Lambda. Tool dispatch is in-process, not a separate microservice; it keeps the architecture simple and the latency low.

The Lambda layer also enforces per-request guardrails: an input classifier with regex patterns screens messages before they reach Bedrock, schema validators check tool-call arguments against expected types, and an injection-resistance prefix is prepended to every system prompt.

### 2.5 DynamoDB and S3

DynamoDB stores conversation state, audit records, and tenant configuration. A single table per practice with composite keys (`pk` = entity, `sk` = sort key) supports the access patterns without joins. S3 holds raw imaging, attachments, and any document the AI consumed during inference, with KMS encryption at rest and bucket policies that deny any cross-account access by default.

### 2.6 CloudWatch and audit logging

Every Bedrock invocation, every SageMaker inference, and every Lambda execution emits structured logs to CloudWatch. We retain Bedrock request bodies and response bodies for 30 days in CloudWatch Logs and ship them to S3 with a 7-year retention lifecycle for HIPAA Security Rule compliance. The audit log answers the question every CISO eventually asks: "what did our AI say to this patient, on this date, in response to this prompt, and which model produced the answer?"

### 2.7 Cognito for identity

Staff and patient identities live in separate Cognito user pools. The separation is not optional — it is the access control surface that prevents a misconfigured dental hygienist account from reading a patient's chart. Both pools support MFA. The patient pool supports passwordless email magic-link login for engagement workflows; the staff pool requires hardware MFA for clinical write operations.

---

## 3. Model selection: Opus, Sonnet, Haiku, Nova

> **Answer:** Model selection in healthcare is workload-driven: Claude Opus 4.6 for complex clinical reasoning that must be right (X-ray analysis, SOAP note generation, treatment planning); Claude Sonnet 4.6 for high-volume conversational and structured-output tasks (clinical assistants, scheduling agents, intake forms); Claude Haiku 4.5 for low-latency classification and routing (intent detection, message triage, tool dispatch); Amazon Nova Pro for patient-facing explanations where cost and latency matter more than the last increment of reasoning quality. We deploy multiple models per practice and route per workload — there is no single right model for healthcare.

The cost-per-1000-sessions math (assumed inputs: 5,000 input tokens, 1,500 output tokens per session, US Bedrock 2026-05 pricing) approximates:

| Model | Input $/MTok | Output $/MTok | Est. cost / 1000 sessions |
|---|---|---|---|
| Claude Opus 4.6 | $15 | $75 | $187 |
| Claude Sonnet 4.6 | $3 | $15 | $37 |
| Claude Haiku 4.5 | $1 | $5 | $13 |
| Amazon Nova Pro | $0.80 | $3.20 | $9 |

Opus is five-times the price of Sonnet and twenty-times the price of Haiku. The cost difference matters when a workload runs hundreds of thousands of times per month. The accuracy difference matters when one wrong answer is a clinical incident. The right discipline is to classify each surface — *does this need Opus, or does this need Haiku?* — and to deploy model assignments that match the criticality of the workload.

Our typical assignments in dental and small-practice healthcare deployments: Opus for X-ray analysis, SOAP transcription, and smile simulation (any clinical pixel or chart-altering output). Sonnet for AI phone agents (Amazon Connect + Lex V2 wiring), clinical assistants, call analytics, and scheduling. Nova Pro for patient-facing visit explanations where readability beats reasoning depth. Haiku for intent classification and tool routing inside larger agentic flows.

---

## 4. Production deployment pattern

> **Answer:** A production private-LLM deployment in dental and small-practice healthcare runs on AppSync GraphQL for the API surface, Cognito with separate staff and patient user pools, a single DynamoDB table per practice for tenant-scoped state, three SageMaker Serverless endpoints for the imaging pipeline, and Bedrock Opus 4.6 plus Sonnet 4.6 routed by workload. The pattern integrates an OSCC (oral squamous cell carcinoma) screening workflow, FDI tooth notation throughout the clinical chart, and the full ADA item code set at the billing layer. Imaging models follow a YOLOv8m architecture across the dental class range (about 32 classes), deployed to SageMaker Serverless and retrained quarterly on consented data.

### 4.1 Imaging pipeline

The imaging pipeline ingests bitewing, panoramic, and intra-oral photos from chairside capture devices. A SageMaker Serverless endpoint runs tooth detection (YOLOv8m, ~32 classes, mid-0.7 mAP50). A second endpoint runs tooth segmentation for treatment planning overlays. A third endpoint runs smile inpainting for cosmetic simulation. Every inference is logged with the model version, input hash, and output payload. Imaging volume on the platform aggregates into the six-figure range across consented practices; non-consented data is held in each practice's own S3 bucket with no cross-practice training.

### 4.2 OSCC screening workflow

Oral squamous cell carcinoma screening runs as a secondary inference stage on intra-oral photos. The flow: a Bedrock Opus 4.6 vision call inspects the image for lesion markers; if the model returns a confidence score above the configured threshold, a Sonnet 4.6 call assembles a structured screening note for clinician review; the note is added to the chart with a "pending review" status that requires explicit clinician acknowledgment before it appears on the patient-facing record. The workflow does not auto-diagnose. It surfaces candidates for human review with structured evidence — precisely the boundary HIPAA expects between machine output and clinical decision.

### 4.3 SOAP note generation

Clinical encounter notes follow a SOAP structure (Subjective, Objective, Assessment, Plan). The dictation flow uses Amazon Transcribe Medical for streaming transcription (clinically tuned vocabulary, speaker diarization, punctuation), then a Bedrock Opus 4.6 call structures the transcript into the SOAP fields. The clinician sees a preview with a confidence score per field and confirms or edits before applying. We do not auto-commit AI-generated notes to the chart — the clinician's signature is the gate.

### 4.4 Voice dictation and chart commands

Beyond SOAP notes, the deployment supports voice-driven chart commands ("tooth 14 distal occlusal, composite, two surfaces"). The parse-dictation API streams Transcribe Medical output to a Sonnet 4.6 call with a structured-output schema (tooth number, surface, condition, confidence). The model produces a typed payload that the chart preview rejects on confidence below a threshold. FDI tooth notation displays throughout the UI (Australian standard, 11–48 permanent, 51–85 primary); internal storage stays in sequential 0–31 arrays to avoid data migration when notation conventions evolve.

### 4.5 The AI phone agent

The AI phone agent is built on Amazon Connect for telephony, Lex V2 for intent and slot capture, and Bedrock Sonnet 4.6 for conversational generation. It books appointments, answers basic clinical questions ("we don't take that insurance, here's who does"), routes complex calls to a human, and surfaces a transcript to the practice's CRM after every interaction. Every call is recorded, transcribed, and stored in the practice's own S3 bucket with a configurable retention window. The pricing model treats the phone agent as an add-on rather than a core SKU — front-desk hours saved typically cover the inference and telephony costs.

### 4.6 ADA item codes and FDI notation

The platform ships with the full set of ADA item codes (363 codes, 13th Edition) and full FDI notation. The chart UI uses FDI by default. The billing UI uses ADA. The translation between them is a deterministic lookup, not a model call — clinical accuracy at the billing layer is non-negotiable, and a probabilistic translation would generate audit findings.

---

## 5. Compliance posture: BAA, SOC 2, HITRUST

> **Answer:** A private LLM architecture on AWS Bedrock satisfies HIPAA's technical safeguards (encryption, access control, audit logging, transmission security) by default — Bedrock, SageMaker, Lambda, DynamoDB, S3, KMS, CloudWatch, and Cognito all operate under HIPAA-eligible terms with a signed AWS Business Associate Agreement. The architecture does not satisfy SOC 2 or HITRUST automatically; those require attestation from a third-party auditor (typically Vanta or Drata for SOC 2 Type 1, with a 60-day implementation cycle and $10,000–$15,000 cost). HITRUST is a longer cycle measured in quarters, not months. We deploy with SOC 2 Type 1 as the first compliance milestone and HITRUST as a 12–18 month roadmap item.

### 5.1 The AWS BAA

We activate the AWS Business Associate Addendum on AWS accounts that handle PHI. The BAA covers the HIPAA-eligible services we use (Bedrock, SageMaker, Lambda, DynamoDB, S3, KMS, CloudWatch, Cognito, AppSync, Transcribe Medical, Connect, Lex V2). It does not cover services that are not on AWS's HIPAA-eligible list — we do not deploy those for healthcare workloads, full stop.

### 5.2 Zero data retention at the model layer

Bedrock supports a zero-data-retention configuration that prevents AWS from logging request and response payloads for service-improvement purposes. We enable this configuration at the account level for every healthcare deployment. The configuration is auditable via the Bedrock model invocation logs and the AWS Config rule that flags accounts with retention enabled.

### 5.3 Audit log lifecycle

CloudWatch Logs holds Bedrock and SageMaker invocations for 30 days. A Logs subscription filter ships entries to S3 with a 7-year retention lifecycle managed by S3 Lifecycle policies. The 7-year window matches the upper bound of state record-retention requirements (Texas, California, and several others). Audit records are encrypted with a per-account KMS key.

### 5.4 IAM least-privilege

Every Lambda function has a dedicated IAM role with the minimum permissions required for that function's job. Bedrock invocation permissions scope to specific foundation-model ARNs and inference-profile ARNs, not wildcards. SageMaker invocation permissions scope to the specific endpoint ARNs. The blast radius of a compromised function is the function's tools — not the entire AWS account.

### 5.5 SOC 2 Type 1

We are exploring SOC 2 Type 1 attestation as the first formal compliance milestone after the architecture is deployed. Type 1 covers control design (the controls exist on paper); Type 2 covers operating effectiveness over a 6–12 month observation window. Most mid-market healthcare buyers want to see Type 1 as a hygiene signal and Type 2 as a maturity signal. This paper will be updated as the firm's compliance posture evolves.

### 5.6 HITRUST CSF

HITRUST is a healthcare-specific control framework that maps HIPAA, HITECH, NIST 800-53, ISO 27001, and several state requirements into a single attestation. It is the compliance signal mid-market healthcare buyers reach for when they want stronger assurance than SOC 2. The full HITRUST CSF certification is a 12–18 month engagement and a meaningful cost ($60,000–$150,000 depending on scope). We position HITRUST as a roadmap item, not a launch requirement.

---

## 6. The "bootstrapped + certified" differentiator

> **Answer:** Most AI infrastructure firms in the mid-market healthcare space are venture-funded and built around a sales motion that depends on platform lock-in. iSimplifyMe is bootstrapped — no VC, no growth-stage burn rate, no need to convert clients into per-seat ARR before the architecture is right. Combined with a formal compliance posture (AWS BAA active for PHI workloads, SOC 2 Type 1 under evaluation, HITRUST on roadmap), the bootstrapped-plus-deliberate-compliance posture is the differentiator that makes "private LLM on AWS Bedrock" a real product rather than a marketing line. Buyers comparing iSM to VC-funded peers should ask: who controls the architecture decisions when growth pressure conflicts with security posture?

We say this without apology because it is the architectural truth of the company. AI Wrappers — firms that thinly resell access to OpenAI or Anthropic with light branding — cannot deliver the architecture this paper describes. They do not own the AWS account. They do not control the BAA chain. They cannot produce an audit log that includes the request body. When the regulator asks the inevitable question, the answer comes back through three providers and two NDAs.

A bootstrapped firm with an active AWS BAA on its healthcare deployments and a deliberate compliance roadmap is structurally better positioned to deliver regulated AI than a Series-B-funded competitor with the same architectural diagrams. The math runs on incentives, not slides.

---

## 7. The 12-week deployment template

> **Answer:** A typical mid-market healthcare deployment runs 12 weeks from kickoff to production traffic. Weeks 1–2: AWS account setup, BAA signing, network design, IAM role provisioning. Weeks 3–4: Bedrock and SageMaker enablement, model selection per workload, baseline cost modeling. Weeks 5–6: data ingestion pipeline, Cognito user pools, audit log lifecycle. Weeks 7–8: first agentic workload (clinical assistant or scheduling agent), guardrail layer, schema validators. Weeks 9–10: vision pipeline if applicable (X-ray, intra-oral photo), SageMaker endpoint deployment, accuracy validation against held-out test set. Week 11: load testing, failure-mode review, cost ceiling configuration. Week 12: production cutover, observability dashboards, on-call rotation handoff.

This is a template, not a guarantee. Practices with existing PMS or EMR integrations add 2–4 weeks for the integration cycle. Multi-location practices add 1–2 weeks for the per-location data residency setup. Practices that need HITRUST attestation before launch add 6–12 months — that engagement runs in parallel and gates the public-launch announcement, not the technical deployment.

The pacing here matters because most enterprise AI deployments fail not on architecture but on integration friction. Twelve weeks is the realistic timeline for an engineering team that owns its data, has a single AWS account, and is willing to cut scope on the first deployment. Teams that try to ship every workload at once consistently slip to 24+ weeks and rebuild parts of the architecture en route.

---

## Frequently Asked Questions

<details>
<summary>Can we bring our own model to this architecture?</summary>

Yes. SageMaker supports custom-trained models deployed as either real-time endpoints or Serverless endpoints. The most common pattern is to fine-tune an open-weights model (Llama, Mistral, or Qwen variants) on practice-specific data and deploy it alongside Bedrock foundation models. Routing logic in the Lambda layer chooses between the custom model and the foundation model per workload. For highly regulated workloads, a custom model can be the only model the architecture invokes — Bedrock is not required if the practice prefers full control of the model weights.
</details>

<details>
<summary>What is the latency budget for X-ray inference?</summary>

For a clinician-facing X-ray analysis flow, the practical budget is 1.5 to 3 seconds end-to-end. SageMaker Serverless adds a cold-start penalty (typically 2–4 seconds for the first request after idle, sub-second for warm requests). For high-throughput practices, we provision a small number of always-warm instances to keep the p99 under 2 seconds. The Bedrock vision call for OSCC screening adds 800ms to 1.4 seconds depending on image size and model choice. The flow assembles to a 3-second p95 in production; the dental clinician's perceptual budget is closer to 5 seconds, so the headroom is comfortable.
</details>

<details>
<summary>Does this require a dedicated VPC?</summary>

Yes. We deploy each healthcare client in a dedicated AWS account with a dedicated VPC. The VPC has private subnets for Lambda, S3 VPC endpoints to keep S3 traffic off the public internet, and Bedrock VPC endpoints for the same reason. The dedicated VPC is not strictly required by HIPAA, but it is the cleanest architecture for the audit trail and the cheapest insurance against cross-tenant data-leakage findings.
</details>

<details>
<summary>What are the BAA gotchas?</summary>

Three. First, the AWS BAA covers AWS-listed HIPAA-eligible services only — it does not cover third-party services you connect via PrivateLink or VPC peering. Second, the BAA is account-scoped, not organization-scoped, so a multi-account deployment needs the addendum signed per account. Third, the BAA does not relieve the practice of its own HIPAA Security Rule and Privacy Rule obligations — it covers AWS's role as a Business Associate, not the practice's role as a Covered Entity. The practice's own security-rule attestation is still required.
</details>

<details>
<summary>How does this compare to Epic's Microsoft AI partnership or Google's Healthcare API?</summary>

Epic's Microsoft AI integration is a closed-stack deployment — the practice gets AI features through Epic's product surface, with Epic and Microsoft holding the architectural control. Google Healthcare API is a different shape: a managed FHIR store and inference layer that the customer accesses through Google Cloud, with Google's BAA covering the listed services. Both are valid choices for practices already standardized on those vendors. The private-LLM-on-Bedrock pattern is the right choice when the practice wants architectural control, multi-cloud optionality, and a model-agnostic deployment that can route to whichever foundation model produces the best result on a given workload.
</details>

<details>
<summary>What does this cost in production?</summary>

Baseline AWS overhead (account, VPC, CloudTrail, Config, GuardDuty) runs $50–$200 per month per account. Inference cost scales with workload. For a typical 50-provider practice running clinical assistant + AI phone agent + imaging pipeline, expect $2,000–$5,000 per month in AWS bills, including SageMaker Serverless inference, Bedrock token billing, Connect minutes, and storage. The line item that dominates is usually Bedrock token spend on Opus for SOAP and smile-simulation workloads, which is why workload-aware model routing (Sonnet for high-volume conversational, Opus only for clinical-critical) matters.
</details>

<details>
<summary>How is this different from running a self-hosted LLM on EC2?</summary>

Self-hosted LLMs on EC2 work, but the operational cost is meaningful — GPU instance management, model versioning, autoscaling, observability — and it duplicates a lot of what SageMaker already offers. We deploy self-hosted only when the practice has a specific reason to control the model weights end-to-end (typically a legal or contractual requirement that prevents inference on a service even with a BAA). For most practices, Bedrock plus SageMaker delivers the same isolation properties with a fraction of the operational overhead.
</details>

---

## Conclusion

Private LLM architecture for healthcare is not a marketing posture — it is an engineering pattern that addresses a real set of regulatory, residency, and audit-trail requirements that public APIs cannot satisfy. The pattern has six layers, runs on AWS services that operate under a signed Business Associate Agreement, and ships in 12 weeks for a focused first deployment. We have the pattern in production at scale: imaging pipelines processing into the six-figure volume range, OSCC screening integrated into clinical flow, ADA codes and full FDI notation in the chart, three SageMaker endpoints, two Bedrock model tiers routed by workload, and a clean audit trail end-to-end.

If your practice or platform is evaluating AI for regulated clinical workflows, the architecture in this paper is the floor. The choices above the floor — model selection, retraining cadence, attestation roadmap, integration pacing — are practice-specific. We design those choices in partnership with the engineering team that will operate the system on day 91.

---

## Notices

**Not legal advice.** This paper is for informational purposes only and does not constitute legal, regulatory, or compliance advice. Consult qualified counsel before relying on any architectural pattern in a regulated deployment. References to HIPAA, BAA terms, state-level health-data statutes (CMIA and analogous laws), SOC 2, HITRUST, and HHS guidance are summaries of our current operational understanding and are not authoritative interpretations.

**Not medical advice.** Clinical workflows described in this paper are reference patterns, not clinical recommendations. The architecture surfaces candidates for human review with structured evidence; final clinical decisions rest with the clinician. Inference output should not be used as a substitute for licensed clinical judgment.

**Architecture and pricing change.** Pricing reflects US AWS Bedrock rates as of May 2026 and may change. Model availability, IAM action names, AWS service capabilities, and the AWS HIPAA-eligible services list evolve continuously; verify current state before implementation. Operational numbers (latency budgets, deployment timelines, cost ranges) describe typical engagements and are not guarantees.

**Trademarks.** AWS, Amazon Bedrock, SageMaker, Lambda, DynamoDB, S3, KMS, CloudWatch, Cognito, Connect, Lex, AppSync, and Transcribe Medical are trademarks of Amazon.com, Inc. or its affiliates. Claude is a trademark of Anthropic, PBC. Epic, Microsoft, and Google are trademarks of their respective owners. References are descriptive and do not imply endorsement.

---

**About the author.** Joseph W. Elstner is the founder and principal architect of iSimplifyMe, a Chicago-headquartered AI infrastructure firm operating since 2011 across North America and Asia-Pacific (Melbourne). iSimplifyMe is bootstrapped, activates the AWS Business Associate Addendum on healthcare deployments, and is exploring SOC 2 Type 1 attestation as the first formal compliance milestone for the firm.

**Contact.** ai@isimplifyme.com — for engineering teams evaluating private LLM architecture, we offer an architecture review session at no cost.

**Cite this paper.** Elstner, J. (2026). *Private LLM Architecture for Mid-Market Healthcare on AWS Bedrock.* iSimplifyMe Whitepaper. https://isimplifyme.com/whitepapers/private-llm-healthcare-aws-bedrock
