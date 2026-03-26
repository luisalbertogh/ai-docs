# Agent and LLM Evaluation Practices

Evaluating LLMs and AI agents requires a multi-layered approach that moves beyond simple "vibe checks" to quantitative metrics and systematic techniques.

## LLM Evaluation Considerations

Choosing the right LLM for a use case requires balancing several competing criteria. No single model excels across all dimensions — the right choice depends on the workload's priorities.

### Performance and Quality

| Criterion | What to assess |
| --- | --- |
| **Task accuracy** | How well does the model perform on your specific domain (coding, reasoning, summarization, multilingual)? Use benchmark scores (MMLU, HumanEval, MATH) as a starting point, but always validate on your own data. |
| **Instruction following** | Does the model reliably follow complex or structured prompts, output formats (JSON, XML), and system instructions? |
| **Reasoning capability** | Can the model handle multi-step reasoning, chain-of-thought tasks, and tool use correctly? |
| **Context window** | What is the usable context length? Longer windows matter for RAG, document analysis, and long conversations. |
| **Hallucination rate** | How often does the model fabricate facts? Critical for knowledge-intensive or high-stakes applications. |
| **Multimodal support** | Does the use case require image, audio, or document understanding beyond plain text? |

### Latency

| Criterion | What to assess |
| --- | --- |
| **Time to first token (TTFT)** | Perceived responsiveness in streaming UIs — lower TTFT feels more interactive. |
| **Tokens per second (TPS)** | Overall throughput for completing a response. Matters most for long-form generation. |
| **P95 / P99 latency** | Tail latency under load — the worst-case experience users will encounter in production. |
| **Cold start behavior** | Does the model/endpoint require warm-up time, or is it always hot? Relevant for serverless deployments. |

### Cost

| Criterion | What to assess |
| --- | --- |
| **Input / output token pricing** | Most providers charge separately per input and output token — output is usually more expensive. |
| **Cost per task** | Estimate total token consumption for a representative task to compare models on a per-call basis. |
| **Context length cost** | Large context windows amplify input token costs; confirm the effective price for your average prompt size. |
| **Batch vs. real-time pricing** | Some providers offer discounted batch inference for asynchronous workloads (e.g., evaluation pipelines). |
| **Rate limits and burst cost** | Understand how quota limits translate into queuing latency or hard failures at scale. |

### Reliability and Availability

| Criterion | What to assess |
| --- | --- |
| **SLA / uptime guarantees** | Does the provider offer a contractual uptime SLA (e.g., 99.9%)? What is the historical reliability? |
| **Failover and redundancy** | Can you route to a fallback model or region if the primary endpoint is degraded? |
| **Rate limit headroom** | Are the default rate limits sufficient for your peak traffic, and is quota increase self-service or gated? |
| **Version stability** | Does the provider pin model versions, or can behavior change without notice on a rolling deployment? |

### Security and Compliance

| Criterion | What to assess |
| --- | --- |
| **Data residency** | Does inference happen within required geographic boundaries (e.g., EU data sovereignty)? |
| **Data retention policy** | Does the provider store your prompts and completions for training or logging? Is zero-retention available? |
| **Compliance certifications** | SOC 2, ISO 27001, HIPAA BAA, GDPR — required certifications depend on your industry and data classification. |
| **Private deployment option** | Can the model run in a VPC, on-premises, or via a dedicated deployment to avoid shared infrastructure? |
| **Prompt injection resilience** | How robust is the model against adversarial inputs designed to override system instructions? |

### Ecosystem and Operability

| Criterion | What to assess |
| --- | --- |
| **API compatibility** | Does the provider support the OpenAI-compatible API format, simplifying integration and model swapping? |
| **SDK and tooling support** | Are there well-maintained SDKs, LangChain/LlamaIndex integrations, and observability hooks? |
| **Fine-tuning availability** | Can the model be fine-tuned on domain-specific data, and at what cost and complexity? |
| **Model card and transparency** | Is the training data, evaluation methodology, and known limitations documented? |
| **Vendor lock-in risk** | How easy is it to swap to a different provider if pricing, quality, or availability changes? |

---

## Key Metrics

### 1. Foundation Model & RAG Metrics

- **Correctness:** Measures how accurately the model's output aligns with ground truth or factual data.
- **Completeness:** Evaluates if the response addresses all parts of the user's query.
- **Faithfulness (Groundedness):** Specifically for RAG, ensures the answer is derived solely from the provided context without hallucinations.
- **Relevance:** Assesses how well the response satisfies the user's intent.
- **Toxicity and Safety:** Monitors for harmful content, bias, or violations of configured guardrails.

### 2. Agent-Specific Metrics

- **Goal Completion Rate:** The percentage of sessions where the agent successfully resolved the user's request.
- **Tool Use Accuracy:** Evaluates if the agent selected the correct tool and provided valid input parameters.
- **Trajectory Efficiency:** Measures the number of steps or "turns" taken to reach a solution; fewer steps often indicate better reasoning.
- **Intent Detection Accuracy:** How accurately the agent identifies what the user is trying to achieve.
- **Latency per Step:** Tracking the time taken for each link in the agent's chain of thought.

## Evaluation Techniques

### 1. LLM-as-a-Judge

Using a high-capability model (like Anthropic Claude 3.5 Sonnet) as an automated "judge" to score the outputs of another model. This allows for scalable qualitative assessment using natural language rubrics.

**Sample metrics:**

| Metric | Description |
| --- | --- |
| **Correctness** | Is the answer factually accurate relative to ground truth or context? |
| **Completeness** | Does the response address all parts of the user's query? |
| **Faithfulness** | Is the answer grounded in the source material without hallucination? |
| **Helpfulness** | Does the response usefully address the user's underlying intent? |
| **Logical coherence** | Is the reasoning internally consistent and well-structured? |
| **Harmfulness** | Does the output contain harmful, offensive, or dangerous content? |
| **Tone / Brand alignment** | Does the response match the expected voice and style? (custom rubric) |

### 2. Programmatic Evaluation

Using deterministic scripts to compare model outputs against a "Golden Set" of ground truth answers. This is ideal for classification tasks or data extraction where there is a clear right answer.

**Sample metrics:**

| Metric | Description |
| --- | --- |
| **Exact match** | Binary check — does the output exactly equal the expected answer? |
| **F1 score** | Token-level overlap between prediction and ground truth (useful for extractive QA). |
| **BERT Score** | Semantic similarity using contextual embeddings — catches paraphrase matches. |
| **ROUGE-L** | Longest common subsequence overlap; common for summarization tasks. |
| **Accuracy** | Proportion of correct classifications across a labeled test set. |
| **Robustness** | Consistency of output under paraphrased or perturbed inputs. |
| **Toxicity score** | Programmatic content safety check (e.g., using a moderation API or classifier). |

### 3. Human-in-the-Loop (HITL)

Subjective evaluation where human reviewers (internal teams or managed workers) score responses based on nuanced criteria like brand voice, empathy, or complex domain expertise.

**Sample metrics:**

| Metric | Description |
| --- | --- |
| **Likert-scale rating** | Reviewers score each response on a 1–5 scale per criterion (e.g., clarity, helpfulness). |
| **Pairwise preference** | Reviewer selects the better of two responses (A/B comparison). |
| **Pass / Fail** | Binary judgment on whether a response meets a defined quality threshold. |
| **Inter-rater agreement (Cohen's κ)** | Measures consistency across multiple human reviewers to validate scoring reliability. |
| **Annotation throughput** | Number of samples reviewed per hour — tracks evaluation pipeline efficiency. |

### 4. Online vs. On-demand Evaluation

- **Online:** Continuous monitoring of live production traffic using sampling to detect performance drift.
- **On-demand:** Targeted evaluation of specific traces or batches of data during the development or debugging phase.

**Sample metrics:**

| Metric | Description |
| --- | --- |
| **Goal completion rate** | Percentage of sessions where the agent successfully resolved the user's request. |
| **Latency (P50 / P95 / P99)** | Response time distribution across live traffic — tracks SLA compliance. |
| **Error rate** | Proportion of failed or refused responses in production traffic. |
| **User satisfaction (CSAT)** | Post-session thumbs-up/down or star rating collected from end users. |
| **Hallucination rate (sampled)** | Proportion of sampled responses flagged as ungrounded by a judge model or guardrail. |
| **Token cost per session** | Average token consumption per interaction — tracks cost efficiency over time. |

### 5. Instrumentation and Tracing

Using standards like **OpenTelemetry** and **OpenInference** to capture the "trace" of an agent's execution. This reveals the internal reasoning, tool calls, and data retrievals, allowing for granular component-level evaluation.

**Sample metrics:**

| Metric | Description |
| --- | --- |
| **Tool use accuracy** | Did the agent select the correct tool with valid parameters at each step? |
| **Trajectory length** | Number of reasoning steps or tool calls taken to reach a final answer. |
| **Retrieval relevance** | Are the retrieved context chunks actually relevant to the query (scored per chunk)? |
| **Step latency** | Time taken for each individual span in the trace (LLM call, tool call, retrieval). |
| **Span error rate** | Proportion of spans that resulted in an error or exception. |
| **Context utilization** | How much of the retrieved context was actually used in the final response? |

---

## AWS-Native Evaluation Services and Features

AWS provides built-in evaluation capabilities through **Amazon Bedrock Evaluations**, a fully managed service for assessing foundation models (FMs) and RAG applications without requiring custom infrastructure. The following summarizes the available evaluation types, metrics, and complementary AWS services.

### Amazon Bedrock Evaluations

Amazon Bedrock Evaluations supports three primary evaluation modes for models and two for RAG pipelines:

#### Model Evaluation

| Mode | Description |
| --- | --- |
| **LLM-as-a-Judge (LLMaaJ)** | Uses a selected judge model to automatically score another model's outputs. Supports custom judge prompts, flexible scoring, and dynamic content injection. Can evaluate Bedrock-hosted models or any external model via Bring Your Own Inference (BYOI). |
| **Programmatic (Automatic)** | Deterministic metrics computed against ground truth: BERT Score, F1, exact match, accuracy, robustness, and toxicity. Uses built-in or custom datasets. |
| **Human-Based** | Routes prompt-response pairs to human reviewers. Can use your internal workforce or an AWS-managed team of skilled evaluators for subjective criteria such as brand voice, style, or domain expertise. |

**Built-in LLM-as-a-Judge metrics (Model Evaluation):**

- Correctness, Completeness, Helpfulness, Logical coherence
- Responsible AI: Harmfulness, Answer refusal, Stereotyping

#### RAG Evaluation (Generally Available — March 2025)

Supports both **Amazon Bedrock Knowledge Bases** and custom RAG pipelines (BYOI). One knowledge base or RAG system is evaluated per job.

| Evaluation Scope | Metrics |
| --- | --- |
| **Retrieval only** | Context relevance, Context coverage |
| **Retrieve & Generate (end-to-end)** | Correctness, Completeness, Faithfulness (hallucination detection), Helpfulness, Logical coherence, Harmfulness, Answer refusal, Stereotyping, **Citation precision**, **Citation coverage** |

> Citation precision and citation coverage are newer metrics (GA March 2025) that assess how accurately the model grounds its response in specific retrieved passages.

#### Key Capabilities

- **BYOI (Bring Your Own Inference):** Evaluate models or RAG systems running anywhere — Bedrock, other cloud providers, or on-premises — by uploading your own input/output pairs.
- **Custom metrics:** Define custom evaluation rubrics using the LLMaaJ framework with your own judge prompts and scoring systems.
- **Comparison across jobs:** The built-in compare feature lets you view results of multiple evaluation jobs side by side to benchmark prompts, models, chunking strategies, or retrieval settings.
- **Guardrails integration:** Amazon Bedrock Guardrails can be incorporated directly into RAG evaluation jobs for testing safety controls alongside quality metrics.
- **Dataset format:** JSONL datasets; up to 1,000 prompts per evaluation job.

### Amazon Bedrock Knowledge Bases

Beyond evaluation jobs, Knowledge Bases provides operational levers that directly affect RAG quality and should be iterated on using evaluation feedback:

- **Chunking strategies** (fixed-size, semantic, hierarchical) — tune for context relevance.
- **Embedding model selection** — affects retrieval precision.
- **Retrieval optimization parameters** — number of chunks, overlap, metadata filtering.
- **Supported vector stores:** Amazon OpenSearch Serverless, Amazon Aurora PostgreSQL, Amazon Neptune Analytics, MongoDB Atlas, Pinecone, Redis Enterprise Cloud.

### Amazon Bedrock Guardrails

Guardrails provide a complementary safety evaluation layer that can be applied at inference time and measured in evaluation jobs:

- Content filters, denial topics, prompt injection detection, grounding checks
- PII detection and redaction
- Hallucination / grounding assessment as a guardrail policy

### Amazon CloudWatch — LLM Observability

Amazon CloudWatch collects runtime metrics from Bedrock services for continuous production monitoring:

| Service | Key CloudWatch Metrics |
| --- | --- |
| **Foundation models** | `Invocations`, `InvocationLatency`, `TimeToFirstToken`, `InputTokenCount`, `OutputTokenCount`, `InvocationThrottles`, `InvocationServerErrors` |
| **Bedrock Agents** (namespace: `AWS/Bedrock/Agents`) | `InvocationCount`, `TotalTime`, `TTFT` (time-to-first-token), `ModelLatency`, `ModelInvocationCount`, `InputTokenCount`, `OutputTokenCount` |
| **Guardrails** | Invocations, latency, text units consumed, findings by policy type (`GuardrailPolicyType`, `FindingType`) |
| **Model Invocation Logs** | Delivery success/failure metrics for S3 and CloudWatch Logs targets |

CloudWatch alarms can be configured to trigger on latency spikes, error rate thresholds, or token quota consumption, enabling proactive detection of model performance drift in production.
