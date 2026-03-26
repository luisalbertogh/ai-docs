# DevOps and Platform Guide for AI Solutions on AWS

> A practitioner-focused guide for DevOps and Platform engineers responsible for building, operating, and governing AI agent and agentic AI solutions on AWS — covering reliability, security, compliance, observability, deployment, and governance.

---

## Guiding Framework: AWS Well-Architected for AI

AWS provides three complementary lenses for AI/ML workloads, all available in the Well-Architected Tool at no cost:

- **Generative AI Lens** — covers agentic AI architecture scenarios, resilience patterns, and operational best practices specific to LLM-based systems.
- **Machine Learning Lens** — addresses the broader ML lifecycle: data management, training, evaluation, deployment, and monitoring.
- **Responsible AI Lens** — provides a structured review framework for fairness, transparency, accountability, and safety.

The overarching recommendation is to follow the **AWS Security Reference Architecture (SRA) for AI**: a dedicated *Generative AI OU* (Organizational Unit) with accounts separated by SDLC environment and risk profile, centralised log archive and security tooling accounts, and explicit governance boundaries between experimentation, staging, and production workloads.

---

## 1. Reliability

### Core Failure Modes

The two primary failure modes in Bedrock-based agents are `ThrottlingException` (quota exhaustion) and `ServiceUnavailableException`. Design for both from the start.

### Recommended Practices

- **Exponential backoff with jitter** — implement token-aware rate limiters combined with exponential backoff on all model invocation calls.
- **Circuit breaker pattern** — use AWS Step Functions to implement circuit breakers around agent tool calls and model invocations. A tripped circuit falls back to a degraded response rather than propagating failures upstream.
- **Cross-Region Inference (CRIS)** — use Bedrock's cross-region inference profiles (US, EU, APAC, or Global) to automatically route requests to available regional capacity during peak utilisation or regional impairment. The Global inference profile spans all AWS commercial regions and provides near-zero failover RTO.
- **Prompt Flow fallback paths** — model explicit fallback actions in Bedrock Flows or Step Functions for each agent step. The Well-Architected Generative AI Lens (GENREL03-BP01) mandates error classification systems and defined fallback responses.
- **Idempotent tool design** — agent tool calls (Lambda functions, API calls) must be idempotent. Retried calls on transient failures must not produce duplicate side effects.
- **Timeout budgets** — set explicit timeouts at every layer: LLM call, tool execution, and overall session. Unbounded waits are a common reliability failure in multi-step agent workflows.

---

## 2. Security

### Account and Network Isolation

- Maintain a **dedicated Generative AI account** separate from application workloads, using AWS Organizations and SCPs to enforce boundaries.
- Route all Bedrock API traffic through **VPC Interface Endpoints** — never traverse the public internet for model invocations involving sensitive data.
- Apply **S3 Gateway Endpoints** for private artifact access (training data, model outputs, RAG source documents).

### Identity and Access

- Enforce **IAM least privilege** at every lifecycle phase: data ingestion, knowledge base indexing, model invocation, agent tool execution, and evaluation.
- Use **resource-based policies** on Bedrock resources (knowledge bases, agent aliases, guardrail IDs) to restrict access to specific IAM roles.
- Manage agent-to-service authentication through **AgentCore Identity** (built on Amazon Cognito with OAuth 2.0 token handling) — avoid embedding credentials in agent prompts or tool definitions.
- Mandate **Guardrail attachment via IAM conditions** — security teams can enforce that every model invocation includes a guardrail using IAM policy conditions, preventing unguarded calls in production.

### Prompt Injection and Input Validation

- Apply a multi-layer defence: input validation at the API boundary, secure prompt engineering (clear delimiters between system instructions and user input), and Bedrock Guardrails content moderation.
- Use **custom orchestration verifiers** as a layer between user input and the agent's reasoning loop to detect and reject instruction injection attempts.
- Log all agent inputs and outputs via CloudTrail data events (note: these must be **explicitly configured** for `AgentAlias`, `KnowledgeBase`, and `FlowAlias` resource types — they are off by default).

### Data Protection

- Encrypt all data at rest (S3, OpenSearch, DynamoDB) with **AWS KMS customer-managed keys (CMKs)**.
- Use **Amazon Macie** for sensitive data discovery in training datasets and RAG source corpora before ingestion.
- Never include PII, credentials, or confidential data in prompt templates stored in version control.

---

## 3. Responsible AI and Compliance

### Guardrails as a Control Plane

Amazon Bedrock Guardrails provides six policy types that together form the responsible AI enforcement layer:

| Policy Type | What It Controls |
| --- | --- |
| **Content filters** | Harmful categories: hate, insults, sexual content, violence — configurable severity thresholds. |
| **Denied topics** | Custom topic lists the agent must refuse to engage with. |
| **Word and PII filters** | Block specific words, phrases, or auto-detect and redact PII (names, SSNs, credit cards). |
| **Grounding checks** | Filters hallucinated responses in RAG — detects answers not grounded in retrieved context (achieves >75% hallucination reduction). |
| **Automated Reasoning checks** | Mathematically validates factual accuracy of responses against a defined rule set (up to 99% accuracy on structured domains). |
| **Prompt injection detection** | Identifies adversarial inputs attempting to override system instructions. |

Guardrails should be **attached to every production agent alias and knowledge base** — treat an unguarded model invocation in production as a security misconfiguration.

### Audit Logging and Traceability

- Enable **CloudTrail data events** for all Bedrock resource types (off by default).
- Enable **Bedrock Model Invocation Logging** to S3 or CloudWatch Logs — captures full inputs, outputs, token counts, and guardrail outcomes for every call.
- Use **Amazon EventBridge** to trigger automated responses to compliance-sensitive events (e.g., guardrail violation rate spike, model version change).

### Regulatory Compliance

- Amazon Bedrock is certified for **GDPR, HIPAA, SOC 1/2/3, FedRAMP High, and ISO 27001**. Confirm which certifications apply to your workload's data classification.
- For GDPR, use EU data residency (EU cross-region inference profile) or self-hosted deployments to keep prompts and completions within the EU.
- Retain invocation logs for the period required by your regulatory framework — configure S3 lifecycle policies accordingly.

---

## 4. Observability and Monitoring

### What to Instrument

Agentic workflows require observability at three levels: the model call, the agent step, and the session.

| Level | Key Signals |
| --- | --- |
| **Model call** | Latency (P50/P95/P99), input/output token counts, throttling rate, guardrail outcomes |
| **Agent step** | Tool selection accuracy, tool call latency, step count per session, error rate per node |
| **Session** | Goal completion rate, total session latency, total cost per session, user satisfaction (CSAT) |

### AWS Tooling

- **AgentCore Observability** — emits OpenTelemetry-compatible telemetry (session count, latency, token usage, error rates) directly into CloudWatch with no additional instrumentation for agents running on AgentCore Runtime.
- **CloudWatch Metrics** — key Bedrock-specific metrics to monitor: `InvocationThrottles`, `InvocationLatency`, `InputTokenCount`, `OutputTokenCount`. For agents: `InvocationCount`, `TotalTime`, `ModelLatency`.
- **CloudWatch Logs Insights** — query structured invocation logs to detect latency clusters, error patterns, and guardrail violation trends.
- **X-Ray** — distributed tracing across Lambda tool functions, Step Functions workflows, and Bedrock calls for end-to-end request attribution.
- **LangSmith / AgentCore Evaluations** — for qualitative signal: LLM-as-judge scores, hallucination rate from grounding checks, goal success rate.

### Alert Thresholds

- `InvocationThrottles` > 5% of requests — signals imminent quota exhaustion.
- `EstimatedTPMQuotaUsage` > 75% — proactively request quota increases before hitting limits.
- Guardrail `GroundingScore` degradation — hallucination rate increase in RAG outputs.
- Agent step count per session increases significantly — indicates reasoning loops or inefficient tool use.

---

## 5. Continuous Deployment and CI/CD for AI

### Three Artifact Types

AI agent pipelines have three distinct artifact types that require separate handling in CI/CD:

| Artifact | Storage | Deployment Mechanism |
| --- | --- | --- |
| **Application code** | Git (Lambda, containers, IaC) | Standard CI/CD pipeline |
| **Prompts and system instructions** | Git (YAML/JSON) | IaC-driven (CDK/CloudFormation) with evaluation gate |
| **Model configuration** | Git (model ID, inference params) | Versioned agent alias promotion |

### Evaluation Gates

Every change to a prompt, agent configuration, or model version must pass an **automated evaluation gate** before reaching production:

1. Run the candidate against a curated **Golden Test Set** (diverse, representative inputs with expected behaviours).
2. Score using LLM-as-judge evaluators (correctness, instruction following, safety) via Amazon Bedrock Evaluations or LangSmith.
3. Require human approval for changes that fall below a defined quality threshold or that affect safety-critical behaviours.
4. Only promote via **agent alias update** after gate passage — never hot-swap a model in a live alias without evaluation.

### Deployment Strategies

- **Shadow testing** — run the new agent version in parallel with production, logging both outputs without serving the new version to users. Compare results offline.
- **Canary deployment** — route a small percentage of live traffic (e.g., 5–10%) to the new agent alias. Monitor error rates, latency, and quality scores before rolling forward.
- **Blue-green via agent aliases** — maintain a `@champion` (production) and `@challenger` (new version) alias at all times. Promote by reassigning the alias pointer — instant rollback by reverting the alias.

### Prompt Versioning

- Store all prompt templates in Git as versioned YAML assets alongside the agent configuration.
- Test prompts against golden test cases using tools such as `promptfoo` or Bedrock Evaluations in the CI pipeline.
- Use **Bedrock Prompt Management** or **LangSmith Prompt Hub** as the authoritative, governed registry for prompts used in production.

---

## 6. Disaster Recovery

### Recovery Targets

| Component | Recommended RPO | Recommended RTO |
| --- | --- | --- |
| **Bedrock on-demand inference** | N/A (no data) | Near-zero (CRIS failover) |
| **Knowledge Base (vector store)** | < 1 hour | < 15 minutes |
| **Fine-tuned model artifacts** | 24 hours | < 1 hour (re-deploy from S3) |
| **Agent configuration (IaC)** | Real-time (Git) | < 30 minutes (redeploy) |
| **Conversation history (DynamoDB)** | < 5 minutes | < 5 minutes (DynamoDB Global Tables) |

### Recommended Patterns

- **Vector store DR** — use OpenSearch cross-cluster replication (active-passive, follower index in DR region) combined with S3 Cross-Region Replication for source documents. Automate re-indexing on failover.
- **Agent configuration DR** — treat IaC (CDK/CloudFormation) in Git as the DR artifact. The repository is the source of truth. Redeploy the entire agent stack from scratch in the DR region via pipeline.
- **Fine-tuned model artifacts** — back up with AWS Backup with cross-region copy rules. Store Safetensors or model weights in a replicated S3 bucket.
- **Bedrock Knowledge Bases** — enable continuous S3 CRR for source document buckets. Maintain a pre-warmed Knowledge Base in the DR region using the same data source configuration.
- **Regular DR testing** — use AWS Fault Injection Simulator (FIS) to simulate regional impairment and validate that agent workflows fail over as expected. Document and rehearse runbooks quarterly.

---

## 7. Cost Management

### Cost Attribution

- Tag all Bedrock resources (`KnowledgeBaseId`, `AgentId`, `GuardrailId`) and all supporting infrastructure (Lambda, OpenSearch, DynamoDB) with consistent tags for team, environment, and application.
- Use **Application Inference Profiles** with cost allocation tags to attribute Bedrock inference costs to individual teams or applications in Cost Explorer and Cost and Usage Reports (CUR).
- The **5:1 output token burn rate** (output tokens consume 5× more quota than input tokens) is a critical planning factor — account for this in budget forecasts.

### Optimisation Levers

| Lever | When to Apply |
| --- | --- |
| **Intelligent Prompt Routing** | Route simple queries to cheaper models (e.g., Haiku, Nova Micro) and complex queries to powerful models (Claude Opus, Nova Premier). |
| **Prompt Caching** | For prompts with large, repeated prefixes (long system prompts, RAG context). Eliminates re-processing costs. |
| **Batch Inference** | For non-real-time workloads (evaluation pipelines, document processing, overnight enrichment jobs). Provides ~50% cost reduction. |
| **Model Distillation** | Compress a large teacher model into a smaller domain-specific student model for high-volume, latency-sensitive inference. |
| **Provisioned Throughput** | For predictable, sustained high-volume workloads — commit to throughput in exchange for a lower per-token rate. |
| **Context window management** | Prune conversation history, compress intermediate agent steps, and limit retrieved chunks to reduce input token counts. |

### Budget Guardrails

- Set **AWS Budgets alerts** at 70%, 90%, and 100% of monthly Bedrock spend thresholds per team.
- Use Step Functions or Lambda to enforce per-session token budgets — reject or downgrade requests that would exceed an allocation.
- Monitor `OutputTokenCount` alongside latency — runaway agentic loops often manifest as both high latency and unexpectedly high output token counts.

---

## 8. Model and Agent Governance

### Versioning and Lifecycle

- **Agent aliases** are the deployment unit — each alias points to a specific agent version and model configuration. Never modify a live alias in place; always create a new version, evaluate it, and promote the alias.
- Maintain at least two active aliases per production agent: `@champion` (serving traffic) and `@challenger` (under evaluation or canary). This enables instant rollback.
- Document every model version change in a **SageMaker Model Card** — an immutable governance record capturing training data lineage, evaluation results, known limitations, and intended use.

### Organisational Governance

- Use **SageMaker Model Registry** with cross-account sharing via AWS RAM for governed model discovery across teams and accounts.
- Require evaluation evidence (automated scores + human review sign-off) as a mandatory artefact before any model is promoted to a production alias.
- Define and enforce **agent responsibility boundaries** — document what actions each agent is authorised to take, which tools it can call, and what data it can access. Store this as policy-as-code alongside the agent definition.
- Implement **confidence thresholds and response validators** — agents should decline or escalate when confidence is low, rather than returning low-quality responses silently.
- Apply **shadow mode** for new agents before live traffic exposure — run in parallel with an existing system, logging but not serving outputs, to validate behaviour without risk.

### Model Provider Updates

- Pin model versions explicitly in agent configurations — avoid using `latest` aliases from providers in production.
- Monitor provider release notes (Anthropic, Meta, Amazon) and plan model updates as versioned deployments with full evaluation cycles, not hotfixes.
- Maintain a rollback path (previous agent version + previous model ID) for at least 30 days after any model update.

---

## 9. Summary: Recommended Approach

The recommended approach follows a layered, defence-in-depth model where governance, security, and observability are embedded at every layer rather than bolted on after deployment.

| Layer | Approach |
| --- | --- |
| **Account structure** | Dedicated Generative AI OU with environment-separated accounts (dev/staging/prod), SCP-enforced boundaries. |
| **Networking** | VPC endpoints for all Bedrock API traffic; no public internet for agent data plane. |
| **Identity** | Least-privilege IAM per agent component; AgentCore Identity for agent-to-service auth. |
| **Safety** | Guardrails on every production invocation (mandatory via IAM condition), covering content, grounding, PII, and prompt injection. |
| **Observability** | OTel-native telemetry into CloudWatch; LLM-specific alerts on throttling, hallucination, and cost. |
| **Deployment** | Three-artifact CI/CD (code, prompts, model config) with evaluation gates and canary/alias-based promotion. |
| **Governance** | Model Cards + Registry for lineage; Git-as-source-of-truth for agent config; alias-based versioning for instant rollback. |
| **Cost** | Tag-based attribution; prompt routing and caching as first-line optimisations; batch inference for async workloads. |
| **Resilience** | CRIS for inference failover; CRR for knowledge base data; IaC-driven agent reconstruction for DR. |

---

## References

- [AWS Well-Architected — Generative AI Lens](https://docs.aws.amazon.com/wellarchitected/latest/generative-ai-lens/generative-ai-lens.html)
- [AWS Well-Architected — Machine Learning Lens](https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/machine-learning-lens.html)
- [AWS Security Reference Architecture — Generative AI](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture-generative-ai/introduction.html)
- [Designing Agentic Workflows on AWS — Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/agentic-ai-patterns/designing-agentic-workflows-on-aws.html)
- [Amazon Bedrock Guardrails Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)
- [Amazon Bedrock AgentCore Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html)
- [Amazon Bedrock Evaluations Documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/evaluation.html)
- [Amazon Bedrock Cross-Region Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles.html)
- [MLOps Best Practices for Serverless AI — AWS Blog](https://aws.amazon.com/blogs/machine-learning/mlops-best-practices-for-serverless-ai/)
- [Responsible AI Practices on AWS](https://aws.amazon.com/machine-learning/responsible-machine-learning/)
