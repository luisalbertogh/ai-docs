# AWS DevOps / Platform Engineering Best Practices for AI Agent Solutions

> **Audience:** DevOps and Platform Engineers managing AI agent workloads, RAG pipelines, and Amazon Bedrock–based solutions on AWS.
> **Date:** March 2026
> **Primary AWS references:** AWS Well-Architected Generative AI Lens (Nov 2025), Responsible AI Lens (re:Invent 2025), Machine Learning Lens (re:Invent 2025), AWS Prescriptive Guidance – Agentic AI Serverless, AWS SRA for AI.

---

## 1. Reliability — Resilient Agents and RAG Pipelines

### Core challenge
Generative AI workloads face two primary classes of transient errors: `429 ThrottlingException` (quota exhaustion — RPM/TPM) and `503 ServiceUnavailableException` (temporary service unavailability). Both require distinct mitigation strategies.

### Throttling and quota management
- Implement **exponential backoff with jitter** on every Bedrock API call. AWS SDKs provide built-in retry logic; configure it explicitly rather than relying on defaults.
- Deploy a **token-aware rate limiter** in front of model invocations. Track both RPM (requests per minute) and TPM (tokens per minute) usage metrics in real time via CloudWatch, and enforce limits before hitting Bedrock quotas.
- Right-size the `max_tokens` parameter per use case — oversized token budgets burn quota unnecessarily.
- Request quota increases proactively via the **Service Quotas console** before scaling to production; plan for the 5:1 burndown rate (output tokens consume 5x more quota than input tokens).

### Circuit breakers and failover
- Implement the **circuit breaker pattern** around every Bedrock API call. Use AWS Step Functions to build low-code circuit breakers that open on persistent errors and route to fallback paths.
- Define **fallback strategies** explicitly: static responses, simplified rule-based flows, or routing to a lower-cost/different model via Amazon Bedrock's model multiplexer pattern.
- Use **Amazon Bedrock Flows** to orchestrate multi-step prompts with explicit error classification and recovery paths — enabling graceful degradation rather than hard failures.
- Set **client-side timeouts** (both connection and request) on all remote service calls. Never rely on default infinite timeouts.

### Cross-region inference (CRIS)
- Use **Amazon Bedrock Cross-Region Inference (CRIS)** — both geography-specific and global inference profiles — to distribute load across AWS Regions and absorb unplanned traffic bursts.
- Specify a **global inference profile ID** instead of a region-specific model ID for workloads that can tolerate cross-region routing. The system intelligently routes to regions with available capacity.
- In **multi-account environments**, configure SCPs and AWS Control Tower carefully so cross-region inference is not inadvertently blocked by regional deny policies.
- CRIS incurs no additional cost and provides valuable throughput headroom during peak demand.

### Multi-AZ and multi-region architecture
- Amazon Bedrock is a **fully managed, multi-AZ service** by design. For application-layer resilience, deploy agent orchestration layers across multiple AZs.
- For vector stores (Amazon OpenSearch Service), enable **cross-cluster replication** to replicate indexes between regions in active-passive mode for disaster readiness.
- Use **Amazon Bedrock's model multiplexer** (open-source reference implementation) to encode operational best practices — timeouts, retries, failover, fault isolation — in a single client facade wrapping every Bedrock API call.

### Agentic reliability
- In the Well-Architected Generative AI Lens, **GENREL03-BP01** specifically mandates managing prompt flows with explicit error classification systems, recovery mechanisms, and fallback actions for external data sources.
- Agents should classify every response, verify external data source availability before retrieval, and define fallback actions when knowledge base queries return empty or low-confidence results.
- Validate completion of distributed computation tasks explicitly — do not assume success.

**Key references:**
- [Optimize for scale and reliability on Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/optimize-your-applications-for-scale-and-reliability-on-amazon-bedrock/)
- [Building self-recovering systems with Bedrock production resilience](https://aws.amazon.com/blogs/publicsector/building-self-recovering-systems-against-technology-and-business-risk-with-amazon-bedrock-production-resilience/)
- [GENREL03-BP01 — Generative AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/genrel03-bp01.html)
- [REL05-BP05 Set client timeouts — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_mitigate_interaction_failure_client_timeouts.html)
- [Cross-Region inference in multi-account environments](https://aws.amazon.com/blogs/machine-learning/enable-amazon-bedrock-cross-region-inference-in-multi-account-environments/)

---

## 2. Security — IAM, Prompt Injection, Network Isolation, Secrets, Encryption

### Account and network isolation (AWS SRA for AI)
- Deploy a **dedicated Generative AI OU** with a separate Generative AI account (hosting Bedrock and associated services) distinct from the Application account. This enforces OU-specific SCPs and simplifies least-privilege implementation.
- Separate Generative AI accounts by **SDLC environment** (dev / test / prod) and by risk profile of the user community. Use separate accounts when customized models contain sensitive training data or when regulatory data isolation is required.
- Use **VPC endpoints (AWS PrivateLink)** for all Bedrock API calls to keep traffic off the public internet. Enforce endpoint policies to restrict which principals and actions are allowed through each endpoint.
- Deploy agent orchestration layers and tool Lambda functions in **private subnets** with no public internet routes. Use NAT gateways or VPC endpoints for any required outbound access.

### IAM least privilege
- Apply the **principle of least privilege** across all phases: model access, model adaptation (fine-tuning), model customization, testing, and operation.
- Use **AWS Organizations SCPs** to restrict which foundation models are accessible at an organizational level — preventing teams from invoking unapproved models.
- Grant **dedicated, least-privileged IAM roles** for each distinct agent function (e.g., separate roles for KB ingestion, agent invocation, guardrail evaluation).
- Use **IAM condition keys** (e.g., `bedrock:ModelId`, `kms:ViaService`) to scope permissions to specific models and encryption contexts.
- Leverage **IAM Access Analyzer** to validate policies and identify unintended access. Regularly review permissions as new Bedrock APIs are released.
- **Amazon Bedrock Guardrails now supports IAM policy-based enforcement**: security teams can mandate guardrail attachment on every model inference call via IAM conditions, enforcing organizational safety policies centrally.

### Prompt injection defense
- Prompt injection is an **application-level concern** (analogous to SQL injection) — AWS secures the infrastructure; customers own the application layer.
- Implement **multi-layered defenses**: input validation, secure prompt engineering (explicit instructions to ignore adversarial content), user confirmation before tool invocations, and Amazon Bedrock Guardrails for content moderation of both inputs and outputs.
- For **indirect prompt injections** (malicious content embedded in retrieved documents, emails, or web pages): implement custom orchestration verifiers that inspect tool inputs and outputs for unexpected instructions; enforce access control and sandboxing so agents only access scoped resources.
- Enable **real-time alerts** on unusual patterns in agent interactions (unexpected tool invocation sequences, anomalous output content).

### Secrets and credential management
- Use **AWS Secrets Manager** for all authentication credentials used by agents to access external tools and APIs. Enable automatic rotation.
- Use **Amazon Bedrock AgentCore Identity** (powered by Amazon Cognito) for enterprise-grade agent identity and credential management — centralized token vault, OAuth 2.0 client credentials and authorization code flow support, identity-aware authorization that passes user context to agent code.
- Store **OAuth 2.0 access/refresh tokens, API keys, and client secrets** in AgentCore's token vault rather than in environment variables or agent configurations.

### Data encryption
- All Amazon Bedrock data is **encrypted at rest and in transit by default** (AES-256, TLS 1.2+).
- Use **AWS KMS customer-managed keys (CMKs)** for Bedrock model customization data, Amazon OpenSearch Service vector indexes, knowledge base data in S3, and AgentCore policy engines for additional control and auditability.
- Amazon Bedrock **never shares prompt/response data with model providers** and never uses customer data to train foundation models.
- Enable **model invocation logging** (to CloudWatch Logs and S3) for full audit trails of every inference — apply data protection policies to mask sensitive data in logs.

**Key references:**
- [AWS SRA for AI — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture-generative-ai/gen-ai-sra.html)
- [Implementing least privilege for Amazon Bedrock](https://aws.amazon.com/blogs/security/implementing-least-privilege-access-for-amazon-bedrock/)
- [Securing Bedrock Agents against indirect prompt injections](https://aws.amazon.com/blogs/machine-learning/securing-amazon-bedrock-agents-a-guide-to-safeguarding-against-indirect-prompt-injections/)
- [Prompt injection security — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-injection.html)
- [Securing AI agents with AgentCore Identity](https://aws.amazon.com/blogs/security/securing-ai-agents-with-amazon-bedrock-agentcore-identity/)
- [Bedrock Guardrails IAM policy-based enforcement](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-guardrails-announces-iam-policy-based-enforcement-to-deliver-safe-ai-interactions/)

---

## 3. Responsible AI and Compliance

### Amazon Bedrock Guardrails
Guardrails are the primary mechanism for implementing responsible AI controls at the application layer. Six policy types are available:

| Policy | Purpose |
|---|---|
| **Content filters** | Detect/filter harmful text or image content (Hate, Insults, Sexual, Violence, Misconduct, Prompt Attack). Standard tier extends to code. |
| **Denied topics** | Block user-defined undesirable topics in queries and responses. |
| **Word filters** | Block exact-match custom words and phrases. |
| **Sensitive information filters** | Detect and redact PII (probabilistic detection of SSNs, credit cards, health data, etc.). |
| **Contextual grounding checks** | Detect hallucinations by validating responses against source documents (RAG workloads). Filters >75% of hallucinated responses. |
| **Automated Reasoning checks** | Mathematically validate response accuracy against logical rule sets — up to 99% accuracy identification. Prevents factual errors without human review. |

- Apply guardrails **consistently across all foundation models** using the `ApplyGuardrail` API — can be invoked independently of model inference for RAG pipelines.
- Associate guardrails with Amazon Bedrock Agents at definition time; enable the **default pre-processing prompt** and use advanced prompt features to reinforce system instructions.
- Enforce guardrails via **IAM policies** to prevent any model invocation that bypasses required guardrails — critical for regulated environments.

### Regulatory compliance (GDPR, HIPAA, SOC, FedRAMP)
- Amazon Bedrock is certified for **GDPR, HIPAA, SOC 1/2/3, FedRAMP High, and FedRAMP Moderate**. Review the [AWS Compliance Programs page](https://aws.amazon.com/compliance/programs/) for the full list.
- Bedrock maintains all data **within the customer's chosen AWS Region** — no cross-border data movement unless explicitly using CRIS profiles.
- For healthcare: use **PII/PHI sensitive information filters** in Guardrails, deploy in HIPAA-eligible regions, execute BAAs with AWS.
- For GDPR: leverage **data residency controls** (geography-specific inference profiles prevent data leaving defined regions), implement right-to-erasure workflows for any stored conversation history.

### Audit logging and bias detection
- Enable **AWS CloudTrail** for all management-plane Bedrock API calls. Configure advanced event selectors to capture data events for `AgentAlias`, `KnowledgeBase`, `FlowAlias`, and `Model` resource types.
- Enable **model invocation logging** to capture full prompt/response pairs in CloudWatch Logs and S3 for compliance review, incident investigation, and continuous improvement.
- Amazon GuardDuty **monitors CloudTrail logs** to detect potential security issues in Bedrock APIs.
- Use **Amazon SageMaker Clarify** for bias detection in custom or fine-tuned models before deployment. Use the **ML Lens** (updated re:Invent 2025) for improved bias detection guidance.
- Leverage **Amazon Bedrock Model Evaluation** for automatic evaluation jobs measuring toxicity, robustness, semantic correctness, and accuracy against prompt datasets.

### Responsible AI Lens (re:Invent 2025)
- The new **Responsible AI Lens** provides a structured framework to assess AI workloads against responsible AI principles — covering fairness, transparency, accountability, safety, and controllability.
- Use the **AWS Well-Architected Tool** to run Responsible AI Lens reviews as a gate before production promotion.
- Define model **use-case governance**: document intended use, risk rating, evaluation results, and business context in **Amazon SageMaker Model Cards**.

**Key references:**
- [Amazon Bedrock Guardrails](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [Bedrock security, privacy, and responsible AI](https://aws.amazon.com/bedrock/security-privacy-responsible-ai/)
- [Safeguarding healthcare data with Bedrock Guardrails (HIPAA)](https://aws.amazon.com/blogs/publicsector/how-to-safeguard-healthcare-data-privacy-using-amazon-bedrock-guardrails/)
- [Three Well-Architected Lenses at re:Invent 2025](https://aws.amazon.com/blogs/architecture/architecting-for-ai-excellence-aws-launches-three-well-architected-lenses-at-reinvent-2025/)
- [Monitor Bedrock API calls using CloudTrail](https://docs.aws.amazon.com/bedrock/latest/userguide/logging-using-cloudtrail.html)
- [GENSEC01-BP04 Access monitoring — Generative AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/gensec01-bp04.html)

---

## 4. Observability and Monitoring

### AgentCore Observability (native)
- **Amazon Bedrock AgentCore Observability** provides built-in tracing, debugging, and monitoring for agent applications. Telemetry is emitted in **OpenTelemetry (OTEL)-compatible format**, enabling integration with existing observability stacks.
- Built-in metrics per agent session: **session count, latency, duration, token usage, error rates** — available in CloudWatch dashboards without any instrumentation.
- For memory resources: spans and logs can be enabled explicitly for deeper tracing of memory read/write operations.
- Instrument agent code with **custom spans and traces** via the AgentCore SDK to add business-level context to the OTEL telemetry stream.
- CloudWatch console provides **trace visualizations, custom span metric graphs, and error breakdowns** for agent runtime.

### Amazon CloudWatch metrics for Bedrock
Key runtime metrics to monitor and alert on:

| Metric | Alert threshold guidance |
|---|---|
| `InvocationLatency` | P95 / P99 thresholds per model; correlate with `TimeToFirstToken` for streaming |
| `InvocationThrottles` | Alert at >5% throttle rate; investigate quota sizing |
| `InvocationServerErrors` | Alert on sustained 503 rate; indicates capacity issues |
| `InputTokenCount` / `OutputTokenCount` | Track against TPM quota; alert at 70–80% utilization |
| `EstimatedTPMQuotaUsage` | Primary quota burn-rate metric for proactive capacity management |
| `ModelInvocationLogsS3DeliveryFailure` | Alert immediately; indicates audit trail gaps |

- Set CloudWatch **anomaly detection** on call error rates for adaptive alerting without fixed thresholds.
- Implement **CloudWatch Logs Insights** queries over invocation logs to detect patterns: high-latency invocations, specific model error clusters, token budget exceedance.

### Distributed tracing for agentic workflows
- Use **AWS X-Ray** for end-to-end distributed tracing across Lambda functions, Step Functions, API Gateway, and Bedrock invocations within agent workflows.
- The **Strands Agents SDK** automatically emits traces in **Embedded Metric Format (EMF)** — covering token usage, performance, tool usage, and event loop cycles — which can be shipped via the CloudWatch agent to X-Ray for visualization.
- Use **CloudWatch Transaction Search** to filter and analyze specific agent invocations by session ID, user, or error type.

### LLM-specific monitoring concerns
- **Hallucination rate**: track guardrail contextual grounding check `GroundingScore` metrics; alert when grounding scores fall below threshold.
- **Latency**: monitor `TimeToFirstToken` (critical for streaming UX) and end-to-end `InvocationLatency` separately — infrastructure and model-side latency have different root causes.
- **Cost**: monitor `InputTokenCount` + `OutputTokenCount` × model price per token. Use **application inference profiles** with cost allocation tags for per-team/per-application cost attribution.
- **Tool invocation patterns**: log and monitor which tools agents invoke, call frequency, and error rates — unexpected tool invocation patterns are a security signal.
- **Third-party observability**: Datadog LLM Observability integrates natively with Amazon Bedrock Agents for latency, error rate, token usage, and tool invocation telemetry.

**Key references:**
- [AgentCore Observability — Developer Guide](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html)
- [Generative AI observability — Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/GenAI-observability.html)
- [Observing Agentic AI workloads using CloudWatch agent (Strands)](https://aws.amazon.com/blogs/mt/observing-agentic-ai-workloads-using-amazon-cloudwatch/)
- [Monitor Bedrock agents with Datadog LLM Observability](https://aws.amazon.com/blogs/machine-learning/monitor-agents-built-on-amazon-bedrock-with-datadog-llm-observability/)
- [Monitoring Amazon Bedrock performance](https://docs.aws.amazon.com/bedrock/latest/userguide/monitoring.html)

---

## 5. Continuous Deployment / CI/CD for AI

### Core principles
CI/CD for AI agents is more complex than traditional software CI/CD because it must version and test **three distinct artifact types**: infrastructure/code (Lambda, CDK stacks), **prompts** (versioned YAML/text assets), and **model configurations** (agent instructions, knowledge base URIs, tool permissions). Without formal CI/CD, teams face model/infrastructure drift, unvalidated prompt updates causing silent behavior changes, and inability to reproduce incidents.

### Typical CI/CD pipeline stages for serverless AI

1. **Code and prompt commit** — Git-based version control for all Lambda functions, CDK stacks, and prompt files. Store prompts as versioned assets (e.g., `/prompts/v2/agent-support-en.yaml`).
2. **Build and lint** — Validate syntax, prompt format, and schema alignment using language linters and custom prompt validators.
3. **Unit tests and prompt regression** — Run golden prompt-response tests using frameworks like `promptfoo` alongside traditional unit tests (`pytest`). Regression failures on prompt golden tests must block promotion.
4. **IaC validation** — `cdk synth` + `cfn-lint` to validate CloudFormation templates before staging deployment.
5. **Integration tests** — Deploy to staging; invoke full agent workflows (S3 upload → Bedrock agent → KB retrieval) using mocked or real test datasets.
6. **Automated quality and security gates** — Run the full evaluation suite (Bedrock Model Evaluation) automatically; fail the pipeline if quality scores drop below threshold. Run static code analysis and adversarial tests.
7. **Human approval gate** — Require human-in-the-loop approval for production promotion, especially for prompt changes, model version upgrades, and guardrail configuration changes.
8. **Deploy to production** — Promote stacks, update Bedrock agent configurations, publish prompt versions using AWS CodeDeploy / CDK / AWS SAM CLI.
9. **Post-deployment smoke tests** — Use CloudWatch Synthetics canaries to validate production agent outputs and rollback readiness.
10. **Monitor and observe** — Auto-create dashboards, cost alerts, and token usage monitors via CloudWatch.

### Deployment strategies
- **Shadow testing**: deploy a new agent version alongside production, routing the same live traffic to both but only surfacing production responses to users. Collect quality metrics in shadow.
- **A/B testing**: route a small percentage of traffic to the new version using weighted routing (e.g., Lambda weighted aliases or API Gateway stage variables). Compare business-critical metrics.
- **Canary deployment**: release to a small user subset, monitor health metrics (latency, error rate, guardrail trigger rate), then gradually increase traffic. Roll back automatically if thresholds are breached.

### Agent and prompt versioning
- Treat prompts as **first-class versioned artifacts** in source control. Include prompt version in CloudTrail events for traceability.
- Use **Amazon Bedrock agent aliases and versions** — agents have explicit versioning and aliasing mechanisms that allow traffic shifting between versions.
- Deploy Bedrock agent configuration updates (tools, instructions, knowledge base URIs) **only via IaC templates** — never via console for production.
- Amazon Bedrock Agents — deploy updates only when prompt regression tests pass, tool permissions match IAM templates, and confidence thresholds are met.

### AgentCore integration with CI/CD
- Amazon Bedrock AgentCore extends CI/CD by managing **agent state, memory, and tool connectors** as part of the deployment lifecycle.
- Key integration points: **runtime registration and versioning**, memory snapshot promotion between environments, tools configuration management, and observability hook validation as a deployment gate.

### Evaluation gates
- Use **Amazon Bedrock Model Evaluation** to run automatic evaluation jobs against prompt datasets before any model version promotion. Score on accuracy, robustness, and semantic correctness.
- Use **Amazon Bedrock Agent Evaluation** (open-source solution) for end-to-end conversational agent evaluation at scale — covers dynamic, multi-turn conversation quality.
- Implement **confidence thresholds**: if model confidence or grounding score is below threshold, route to human review rather than blocking the pipeline outright.

**Key references:**
- [CI/CD and automation for serverless AI — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/agentic-ai-serverless/cicd-and-automation.html)
- [Advanced operations for gen AI in production — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/gen-ai-lifecycle-operational-excellence/prod-monitoring-advanced-operations.html)
- [Hardening through a GenAIOps framework — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/gen-ai-lifecycle-operational-excellence/preprod-hardening.html)
- [MLREL03-BP01 Enable CI/CD/CT automation — ML Lens](https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/mlrel03-bp01.html)
- [Evaluate conversational AI agents with Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/evaluate-conversational-ai-agents-with-amazon-bedrock/)

---

## 6. Disaster Recovery

### Scope of DR for AI workloads
AI workloads introduce artifact types that have no direct analog in traditional DR planning. A complete DR strategy must cover: **vector stores / embeddings indexes**, **knowledge base metadata and source documents**, **fine-tuned model artifacts**, **agent configurations**, **prompt registries**, **session/conversation history**, and **guardrail configurations**.

### DR strategies (Well-Architected Framework)
The four standard DR patterns apply to AI workloads with AI-specific considerations:

| Strategy | RTO / RPO | AI-specific notes |
|---|---|---|
| **Backup and Restore** | Hours | Low cost; suitable for KBs, S3 source documents, agent configs. Use AWS Backup. |
| **Pilot Light** | < 1 hour | Replicate core data; pre-deploy agent infrastructure in standby region. |
| **Warm Standby** | < 1 hour | Scaled-down full environment. Validate with regular DR drills. |
| **Multi-Site Active-Active** | < 5 minutes | Use CRIS for model inference layer; requires active-active vector store replication. |

### Vector store / knowledge base DR
- **Amazon OpenSearch Service**: enable **cross-cluster replication** (active-passive) to replicate user indexes, mappings, and metadata to a follower domain in a secondary region. Configure auto-follow to replicate all indexes matching naming patterns automatically.
- **Amazon S3 source documents**: enable **S3 Cross-Region Replication (CRR)** for all knowledge base source buckets. Use S3 Versioning to protect against accidental deletion.
- **Embeddings re-generation**: maintain the embedding pipeline as IaC so re-indexing from S3 replicas in the DR region is automated and testable. Document the RPO impact of re-indexing time.
- **Knowledge base metadata** (chunking configurations, data source definitions): deploy as CloudFormation/CDK stacks stored in CodeCommit/Git — recoverable in minutes.

### Model artifact DR
- **Fine-tuned model artifacts**: Amazon Bedrock stores custom model weights in the service. Use **AWS Backup** policies and cross-region copy rules to back up associated S3 training data and model outputs.
- **SageMaker model artifacts**: store in S3 with CRR. Register versions in **SageMaker Model Registry** — registry metadata should be backed up via AWS Backup or replicated through pipeline automation.
- Amazon Bedrock **foundation models** (on-demand, not fine-tuned) require no DR — they are AWS-managed and regionally available.

### Agent configuration DR
- All agent definitions, action group schemas, guardrail configurations, and inference profiles must be managed as **IaC (CDK/CloudFormation)**. The IaC repository (CodeCommit or GitHub) is the source of truth and is itself the DR artifact.
- Test DR runbooks regularly using **AWS Fault Injection Service (FIS)** to simulate regional failure scenarios.
- Use **Amazon Route 53 health checks and failover routing** to redirect agent API traffic to the standby region during regional impairment.
- Use **Amazon Application Recovery Controller (ARC)** to orchestrate regional failover detection and traffic shifting.

### RTO/RPO targets for AI services
- For on-demand Bedrock inference: RTO is near-zero using CRIS (automatic regional failover); RPO is N/A (stateless).
- For knowledge bases: RPO = time since last successful S3 sync + re-index time. Target RPO < 1 hour using continuous CRR and automated re-index pipelines.
- For fine-tuned models: RPO = time since last artifact backup. RTO = time to re-register model in target region.
- Run quarterly DR drills; document actual RTO/RPO measured vs. targets.

**Key references:**
- [OpenSearch cross-cluster replication](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/replication.html)
- [REL13-BP02 Recovery strategies — Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/2022-03-31/framework/rel_planning_for_recovery_disaster_recovery.html)
- [Resilience in Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/disaster-recovery-resiliency.html)
- [AWS Elastic Disaster Recovery](https://docs.aws.amazon.com/drs/latest/userguide/CloudEndure-Concepts.html)

---

## 7. Cost Management

### Token cost governance
- Implement a **token budget enforcement layer** (e.g., AWS Step Functions rate limiter workflow) that reads current TPM usage from CloudWatch, compares against predefined limits stored in DynamoDB, and enforces budgetary controls before invoking models.
- Monitor both **leading indicators** (token quota burn rate, request rate) and **trailing indicators** (Cost Explorer, CUR reports) for proactive cost management.
- Use **Amazon Bedrock application inference profiles** with **cost allocation tags** to attribute model usage costs to specific teams, applications, cost centers, and environments. Activate tags in the Billing and Cost Management console to make them visible in Cost Explorer and the Cost and Usage Report (CUR).

### Pricing model selection
Amazon Bedrock offers three pricing models — select based on usage profile:

| Pricing model | Best for | Notes |
|---|---|---|
| **On-Demand** | Variable/unpredictable workloads | Pay per token; no commitment; subject to account quotas |
| **Provisioned Throughput** | Consistent, predictable high-volume workloads | Reserved capacity; eliminates throttling risk; use for production SLAs |
| **Batch Inference** | Non-time-sensitive bulk processing | Up to 50% cost reduction vs. on-demand |

- Use **Intelligent Prompt Routing** (Bedrock native feature) to automatically route simpler queries to lower-cost models and complex queries to higher-capability models — reducing average cost per request without manual routing logic.
- Use **Prompt Caching** to eliminate re-processing of identical prompt prefixes (system prompts, static context) — reduces cost and latency for repeated patterns.

### Model selection and optimization
- Right-size model selection for the task. Use lighter models (e.g., Claude Haiku for high-volume, latency-sensitive tasks; Claude Sonnet/Opus for complex reasoning). In multi-agent architectures, use heavyweight models only for orchestrator agents.
- **Model Distillation** (Bedrock feature): distill larger models into smaller, cheaper models that retain most accuracy for specific use cases.
- **Fine-tuning**: fine-tuned smaller models can outperform larger general models at lower cost for domain-specific tasks.
- Optimize prompts for **conciseness** — unnecessary context increases token costs without improving quality.
- Design **small, focused agents** that hand off to each other rather than monolithic agents with large system prompts.

### Cost monitoring and allocation
- Use **AWS Cost Explorer** to visualize Bedrock costs by model, region, and tag dimensions.
- Set **AWS Budgets alerts** at project/team level for Bedrock spend.
- Integrate cost impact analysis into the **CI/CD pipeline** — human approval gates should surface estimated token cost impact of prompt or model changes before production promotion.
- Optimize **vector store costs**: manage data freshness and re-indexing frequency; remove stale documents from knowledge bases to avoid paying for unnecessary retrieval computation.

**Key references:**
- [Build a proactive AI cost management system — Part 1](https://aws.amazon.com/blogs/machine-learning/build-a-proactive-ai-cost-management-system-for-amazon-bedrock-part-1/)
- [Build a proactive AI cost management system — Part 2 (tagging)](https://aws.amazon.com/blogs/machine-learning/build-a-proactive-ai-cost-management-system-for-amazon-bedrock-part-2/)
- [Track, allocate, and manage generative AI cost with Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/track-allocate-and-manage-your-generative-ai-cost-and-usage-with-amazon-bedrock/)
- [Optimizing cost for foundation models with Amazon Bedrock](https://aws.amazon.com/blogs/aws-cloud-financial-management/optimizing-cost-for-using-foundational-models-with-amazon-bedrock/)
- [Effective cost optimization strategies for Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/effective-cost-optimization-strategies-for-amazon-bedrock/)

---

## 8. Model and Agent Governance

### Lifecycle management: prompts, agents, and models
Without formal lifecycle governance, enterprises face behavior drift, data leakage, undetected accuracy degradation, and lack of reproducibility. The AWS prescriptive guidance identifies three distinct lifecycle layers:

**Prompt governance:**
- Version-control all prompts as first-class artifacts in Git. Treat prompt changes with the same rigor as code changes.
- Use **prompt templates with variable injection** — reduces duplication, enables parameterized evaluation, and supports A/B testing of prompt variants.
- Establish a formal **prompt governance workflow** for prompts that affect regulated outputs (healthcare, legal, financial): creation → peer review → golden test validation → approval → deployment.
- Log all prompts, parameters, and model responses for retrospective review of errors, hallucinations, and security incidents.
- Maintain a registry of **golden test cases** per prompt; run regression automatically on every change.

**Agent governance:**
- Define explicit **responsibility boundaries** for each agent and its tools — agents should only invoke scoped tools based on least-privilege principles aligned with enterprise RBAC.
- Use **shadow mode** to observe new agents or prompts against production traffic before activating for users.
- Establish **confidence thresholds and fallback behavior** — route to human review, static rules, or simpler workflows when model confidence is low.
- For high-stakes use cases, deploy a **response validator Lambda** that inspects LLM responses against policy rules before surfacing to users.
- Use **model selection abstraction layers** — decouple business logic from specific model IDs to enable dynamic routing, fallback, and cost-performance tuning without application changes.

**Model versioning and tracking:**
- Track all model versions and provider update schedules. Know exactly which model version is in production — critical for reproducibility, evaluation, and cost impact analysis.
- Monitor model provider release notes; test new model versions in shadow mode before promoting.
- Document which evaluation results justified each model version promotion.

### Amazon SageMaker Model Registry and Model Cards
- Use **SageMaker Model Registry** as the authoritative store for all ML model versions — including custom/fine-tuned Bedrock models. Track approval stages (pending → approved → rejected → deployed).
- Use **cross-account Model Registry sharing via AWS RAM** to enable governed model discovery and consumption across organizational accounts.
- Create **SageMaker Model Cards** for each production model documenting: intended use, risk rating, training details, evaluation results, performance metrics, known limitations, and deprecation timeline. Model Cards create an immutable governance record.
- Leverage **SageMaker Role Manager** to define least-privilege IAM roles for ML personas (data scientist, MLOps engineer, model risk manager) — ensuring appropriate access at each lifecycle stage.
- Use **SageMaker Model Dashboard** for a unified view of all deployed models, integrating Model Monitor, Transform Jobs, Endpoints, ML Lineage Tracking, and CloudWatch data.

### Deprecation and rollback policies
- Define explicit **model deprecation timelines** when upgrading foundation model versions. Maintain the previous version in an alias for a defined rollback window (e.g., 30 days).
- Use **Amazon Bedrock agent aliases** to enable instant rollback to a previous agent version without redeployment.
- Document the **rollback procedure** in runbooks and test it quarterly.
- When a model provider deprecates a model version: run the full evaluation suite against the replacement model, promote via CI/CD pipeline with human approval, maintain the deprecated version in shadow mode during the transition.

### Governance framework alignment
- Apply the **AWS Well-Architected Responsible AI Lens** as a periodic governance review — structured assessment of fairness, transparency, accountability, safety, and controllability.
- Use the **AWS Well-Architected Tool** to run lensed workload reviews and generate improvement plans with prioritized recommendations.

**Key references:**
- [Prompt, agent, and model lifecycle management — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/agentic-ai-serverless/prompt-agent-and-model.html)
- [Model governance in SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/governance.html)
- [Unified Model Cards and Model Registry — SageMaker](https://aws.amazon.com/blogs/machine-learning/improve-governance-of-models-with-amazon-sagemaker-unified-model-cards-and-model-registry/)
- [Centralize model governance with SageMaker Registry and AWS RAM](https://aws.amazon.com/blogs/machine-learning/centralize-model-governance-with-sagemaker-model-registry-resource-access-manager-sharing/)
- [Bedrock Model Cards — Best Practices for AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/best-practices.html)

---

## 9. AWS Well-Architected Approach for AI/ML Workloads

### The three AI/ML Well-Architected Lenses (re:Invent 2025)
AWS released three complementary lenses at re:Invent 2025 that form a complete framework for AI/ML workload governance:

| Lens | Focus | Target audience |
|---|---|---|
| **Generative AI Lens** (updated Nov 2025) | Foundation models, LLM applications, agentic AI, RAG | Architects, MLOps engineers, builders |
| **Machine Learning Lens** (updated Nov 2025) | Traditional ML, SageMaker, training, inference, MLOps | Data scientists, ML engineers |
| **Responsible AI Lens** (new, re:Invent 2025) | Fairness, transparency, safety, accountability across AI systems | Risk managers, compliance, all AI teams |

Use all three lenses together — they are complementary, not duplicative. Run Well-Architected reviews using the **AWS Well-Architected Tool** (available at no charge in the console).

### Six pillars applied to generative AI

**Operational Excellence**
- Achieve consistent model output quality through continuous evaluation and monitoring.
- Maintain traceability across the full inference chain (prompt → retrieval → generation → response).
- Automate lifecycle management for prompts, agents, and models via CI/CD pipelines.
- Determine when model customization (fine-tuning, distillation) is warranted versus prompt engineering alone.
- Use the **GenAIOps framework** to operationalize generative AI: formal AI stack, automated pipelines, deep observability, multi-layered testing, and continuous feedback loops.

**Security**
- Protect Bedrock endpoints with VPC isolation, IAM least privilege, and SCPs.
- Mitigate harmful outputs and excessive agent agency through Guardrails and response validators.
- Audit all events via CloudTrail; enforce mandatory guardrail attachment via IAM policies.
- Secure prompts against injection; remediate model poisoning risks in fine-tuning pipelines.

**Reliability**
- Handle throughput requirements with CRIS and token-aware rate limiting.
- Implement observability across all agentic components.
- Handle failures gracefully with circuit breakers, fallbacks, and explicit error classification.
- Version all artifacts (prompts, agents, models, infrastructure) to enable rapid rollback.

**Performance Efficiency**
- Capture and continuously improve model performance through evaluation pipelines.
- Optimize vector store retrieval with proper chunking strategies, embedding model selection, and index configuration.
- Use prompt caching, intelligent routing, and model distillation to reduce latency and cost simultaneously.
- Optimize computation resources: prefer serverless (Lambda, AgentCore Runtime) for variable workloads.

**Cost Optimization**
- Select cost-optimized models for each task type.
- Balance on-demand vs. provisioned throughput based on workload predictability.
- Engineer prompts for conciseness; use batch inference for non-time-sensitive workloads.
- Optimize vector store ingestion frequency and storage to avoid unnecessary cost.
- Tag everything for cost attribution and use Cost Explorer + budgets for governance.

**Sustainability**
- Minimize computational resources for training, customization, hosting, and storage.
- Leverage serverless architectures (AWS Lambda, Amazon Bedrock serverless inference) to eliminate idle compute.
- Use model efficiency techniques: distillation, quantization, and fine-tuned smaller models in place of large general models where appropriate.
- Right-size embedding models — larger embeddings are not always better; evaluate quality vs. cost.

### Multi-account architecture recommendation
Following **AWS SRA for AI**:
- Dedicated **Generative AI OU** with Application and Generative AI accounts separated.
- Separate accounts per **SDLC environment** (dev / test / prod) and per risk profile.
- Centralized **Log Archive account** and **Security Tooling account** for cross-account audit log aggregation and security event monitoring.
- All AI workload guardrails, budgets, and SCPs enforced at the OU level — not left to individual teams.

**Key references:**
- [Generative AI Lens — Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lens.html)
- [Three Well-Architected Lenses at re:Invent 2025](https://aws.amazon.com/blogs/architecture/architecting-for-ai-excellence-aws-launches-three-well-architected-lenses-at-reinvent-2025/)
- [Announcing the updated Generative AI Lens](https://aws.amazon.com/blogs/architecture/announcing-the-updated-aws-well-architected-generative-ai-lens/)
- [AWS Well-Architected Tool](https://aws.amazon.com/well-architected-tool/resources/)
- [AWS SRA for AI — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture-generative-ai/gen-ai-sra.html)
- [GenAIOps framework — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/gen-ai-lifecycle-operational-excellence/preprod-hardening.html)

---

## Quick Reference: Key AWS Services per Domain

| Domain | Primary AWS Services |
|---|---|
| Reliability / Failover | Amazon Bedrock CRIS, AWS Step Functions (circuit breakers), Amazon Bedrock Flows, Amazon Route 53, ARC |
| Security | AWS IAM, AWS Organizations SCPs, AWS PrivateLink, AWS Secrets Manager, AWS KMS, Amazon Bedrock AgentCore Identity, Amazon GuardDuty |
| Responsible AI | Amazon Bedrock Guardrails, Amazon Bedrock Model Evaluation, Amazon SageMaker Clarify |
| Observability | Amazon CloudWatch, AWS X-Ray, AgentCore Observability, AWS CloudTrail, Amazon Bedrock invocation logging |
| CI/CD | AWS CodePipeline, AWS CodeBuild, AWS CodeDeploy, AWS CDK, AWS CloudFormation, CloudWatch Synthetics, Amazon Bedrock Model Evaluation, Bedrock Agent Evaluation |
| Disaster Recovery | AWS Backup, Amazon S3 CRR, OpenSearch cross-cluster replication, AWS Elastic Disaster Recovery, AWS FIS |
| Cost Management | AWS Cost Explorer, AWS Budgets, Amazon Bedrock application inference profiles, AWS Cost Allocation Tags, CUR |
| Model / Agent Governance | Amazon SageMaker Model Registry, SageMaker Model Cards, SageMaker Model Dashboard, SageMaker Role Manager, Amazon Bedrock agent versions/aliases |
| Well-Architected Reviews | AWS Well-Architected Tool, Generative AI Lens, Responsible AI Lens, Machine Learning Lens |
