# LangChain Ecosystem: LangChain, LangGraph, and LangSmith

> This document covers the three core libraries of the LangChain ecosystem: **LangChain** (the agent framework), **LangGraph** (the stateful orchestration engine), and **LangSmith** (the observability and evaluation platform). Together they form a full stack for building, orchestrating, debugging, and monitoring production AI agents.

---

## 1. LangChain

### 1.1 Purpose

LangChain is an open-source framework for building applications powered by large language models. Its core problem: integrating LLMs with tools, data sources, memory, and external services used to require significant custom wiring. LangChain provides a standardised, composable abstraction layer that lets developers connect any model to any tool with minimal boilerplate, and swap providers without rewriting application logic.

Since version 1.0 (released 2025), LangChain is built on top of the LangGraph runtime, meaning even simple agents benefit from durable execution, persistence, and human-in-the-loop support out of the box.

---

### 1.2 Main Features

| Feature | Description |
| --- | --- |
| **Standard Model Interface** | Unified API across 100+ LLM providers (OpenAI, Anthropic, Bedrock, Gemini, etc.) so the provider can be swapped without changing application code. |
| **Tool / Function Calling** | First-class support for binding tools and functions to LLMs, including automatic schema generation and result routing. |
| **Prompt Templates** | Composable, versioned prompt templates with variable injection, few-shot support, and message formatting for chat models. |
| **Memory Abstractions** | Short-term conversation buffers and long-term persistent memory that survives across agent sessions. |
| **Retrieval / RAG** | Built-in document loaders, text splitters, embedding wrappers, and vector store retrievers for retrieval-augmented generation pipelines. |
| **Prebuilt Agents** | Ready-to-use agent architectures (ReAct, tool-calling, structured output) instantiable with `create_agent` in under 10 lines of code. |
| **Middleware Hooks** | Composable hooks that intercept agent steps to add human approval, compress context, redact PII, or enforce guardrails without touching core logic. |
| **LangGraph Runtime** | Agents run on LangGraph's durable engine, gaining checkpointing, state persistence, rewind, and streaming for free. |
| **LangSmith Integration** | Native tracing and observability wired in by default; every chain and agent run is automatically traced when a LangSmith API key is present. |
| **1,000+ Integrations** | Community and official integrations with vector stores, document loaders, APIs, cloud services, and databases. |
| **Python and JavaScript SDKs** | Fully parallel implementations in both languages with shared conceptual guides. |

---

### 1.3 Key Considerations Before Adopting

- **Abstraction overhead**: The framework adds layers on top of provider SDKs. For simple, single-call use cases this overhead is unnecessary and teams are often better served by the native SDK directly.
- **Ecosystem churn**: LangChain has a history of rapid API changes. Upgrading minor versions has broken production code. Pinning dependency versions and thorough integration tests are essential.
- **Learning curve breadth**: The ecosystem (LangChain + LangGraph + LangSmith) is large. Teams must invest time to understand which layer solves which problem.
- **Debugging complexity**: The composability that makes chains flexible also makes stack traces harder to read. LangSmith is effectively required in production.
- **Not a substitute for prompt engineering**: LangChain wraps models but does not make them smarter. Poor prompts and weak model selection still produce poor results.
- **Vendor alignment**: LangChain is maintained by LangChain AI Inc., a VC-backed company. The open-source core (MIT licensed) is stable, but the surrounding managed services (LangSmith, LangGraph Platform) carry commercial risk.
- **Framework lock-in**: Applications built heavily around LangChain abstractions are harder to migrate to bare-metal SDKs or other frameworks (e.g., LlamaIndex, Haystack). The interfaces are opinionated.

---

### 1.4 Suitable Use Cases

- **Conversational assistants** that need to maintain context across many turns, use tools (web search, calculators, databases), and be swappable between model providers.
- **RAG applications** combining document ingestion, chunking, embedding, and retrieval into a unified pipeline over enterprise knowledge bases.
- **Rapid prototyping** of LLM-backed features where the goal is a working demo quickly, before potentially replacing LangChain with a leaner stack.
- **Multi-tool agents** that dynamically select from a catalogue of tools (APIs, code execution, form filling) based on user intent.
- **Structured data extraction** pipelines that parse unstructured documents (PDFs, emails, HTML) into typed schemas.
- **Multi-provider applications** where different tasks route to different models (e.g., fast cheap model for classification, powerful model for generation).

---

### 1.5 Out-of-the-Box AWS Integrations

All AWS integrations are consolidated in the dedicated **`langchain-aws`** package (`pip install langchain-aws`), which replaces the older AWS components that lived in `langchain-community`.

| AWS Service | Integration Type | What It Enables |
| --- | --- | --- |
| **Amazon Bedrock** | LLM / Chat Model | Access to foundation models from Anthropic, Meta, Mistral, AI21, Cohere, Amazon, and Stability AI via a single Bedrock API. |
| **Amazon Bedrock Converse** | Chat Model | Unified conversational interface for all Bedrock models with consistent message formatting. |
| **Amazon Bedrock Knowledge Bases** | Retriever | Managed RAG retriever backed by Bedrock's native vector store and ingestion pipeline. |
| **Amazon Bedrock Agents** | Runnable / Agent | Integrates with native Bedrock Agents for orchestration, allowing LangChain chains to call or be called by Bedrock-managed agents. |
| **Amazon Bedrock AgentCore** | Agent Runtime | Managed execution environment for agents with built-in observability, session management, and memory. |
| **Amazon Bedrock AgentCore Browser** | Tool | Enables agents to navigate and automate web pages inside a sandboxed browser. |
| **Amazon Bedrock AgentCore Code Interpreter** | Tool | Executes Python, JavaScript, and TypeScript in isolated MicroVM sandboxes for deep-research agents. |
| **Amazon SageMaker Endpoints** | LLM / Chat Model | Hosts custom or fine-tuned models on SageMaker and exposes them through the standard LangChain model interface. |
| **Amazon SageMaker Experiments** | Tracking | ML experiment tracking and run comparison integrated with SageMaker's experiment management system. |
| **Amazon S3** | Document Loader / Blob Store | Loads files and directories from S3 buckets as LangChain documents; also stores trace artifacts in self-hosted LangSmith deployments. |
| **Amazon S3 Vectors** | Vector Store | Native S3-based vector storage for embedding-based retrieval (preview service). |
| **Amazon Kendra** | Retriever | Intelligent enterprise search over documents using Kendra's NLP-powered index. |
| **Amazon OpenSearch Service** | Vector Store | Distributed search and analytics engine with vector similarity search for RAG. |
| **Amazon DocumentDB** | Vector Store | MongoDB-compatible managed database with vector search capability. |
| **Amazon MemoryDB** | Vector Store / Checkpointer | Redis-compatible in-memory database used for both vector retrieval and LangGraph state checkpointing. |
| **Amazon DynamoDB** | Chat History / Checkpointer | Persistent conversation history and LangGraph agent state checkpointing in DynamoDB tables. |
| **AWS Lambda** | Tool | Invokes Lambda functions as agent tools, enabling serverless execution of arbitrary business logic. |
| **Amazon API Gateway** | Tool | Wraps RESTful and WebSocket APIs managed by API Gateway as callable agent tools. |
| **Amazon Textract** | Document Loader | Extracts text, tables, and forms from scanned PDFs and images via Textract's ML OCR. |
| **Amazon Athena** | Tool / Retriever | Queries data in S3 using serverless SQL via Athena, usable as an agent data tool. |
| **AWS Glue Data Catalog** | Metadata | Provides schema and metadata context to agents querying Glue-managed data assets. |
| **Amazon Neptune** | Graph Store | Graph database supporting Cypher and SPARQL for knowledge graph-backed retrieval. |
| **Amazon Comprehend** | NLP Tool | Content moderation and entity recognition using Comprehend's managed NLP models. |
| **Amazon ElastiCache (Valkey/Redis)** | Cache / Checkpointer | High-performance caching and LangGraph state persistence via Valkey or Redis protocol. |

---

---

## 2. LangGraph

### 2.1 Purpose

LangGraph is the low-level orchestration engine that powers LangChain's agent runtime. It solves a specific problem: standard LLM chains are stateless and linear, but real-world agents need to loop, branch, revisit previous steps, persist state across long sessions, handle failures gracefully, and coordinate multiple specialised sub-agents. LangGraph models these workflows as directed graphs (nodes = computation steps, edges = transitions), giving developers precise, explicit control over execution flow rather than relying on implicit agent reasoning.

LangGraph is open-source (MIT licensed), available in Python and JavaScript, and is the recommended foundation for any agent that needs to run reliably in production.

---

### 2.2 Main Features

| Feature | Description |
| --- | --- |
| **Graph-based Workflow Model** | Agents are defined as directed graphs where nodes are Python functions or sub-graphs and edges are conditional or unconditional transitions, inspired by Pregel and Apache Beam. |
| **Persistent State Management** | Every graph execution maintains a typed state object that is automatically persisted at each step (checkpoint), enabling long-running workflows that survive process restarts. |
| **Durable Execution** | Built-in checkpointing means agents can resume from the last successful node after failures, network timeouts, or planned pauses without re-running completed work. |
| **Human-in-the-Loop** | Workflows can pause at any node to request human review, approval, or input, then resume from that exact point once the human responds. |
| **Streaming** | Token-by-token and step-by-step streaming is native, giving real-time visibility into agent reasoning and intermediate outputs. |
| **Multi-Agent Orchestration** | Supports supervisor, hierarchical, and peer-to-peer multi-agent architectures where specialised agents hand off tasks, share state, or coordinate through a router. |
| **Memory Integration** | Both short-term working memory (within a single run) and long-term cross-session memory (via external stores like DynamoDB or MemoryDB) are first-class concepts. |
| **Provider Agnostic** | Compatible with any LLM that supports tool/function calling; no lock-in to a specific model provider. |
| **LangSmith Integration** | Every graph execution is automatically traced in LangSmith, with full visibility into node transitions, state diffs, and intermediate LLM calls. |
| **LangGraph Platform** | Managed deployment infrastructure (generally available) for hosting, scaling, and operating long-running LangGraph agents in production without managing Kubernetes yourself. |
| **LangGraph Studio** | Visual IDE for inspecting, debugging, and replaying agent graph executions with branching and retry support. |
| **Prebuilt Architectures** | Common patterns (ReAct, tool-calling loops, supervisor multi-agent) available as prebuilt graph templates to reduce boilerplate. |

---

### 2.3 Key Considerations Before Adopting

- **Steep learning curve**: The graph/node/edge mental model, combined with typed state machines and Pregel-style execution semantics, requires significant ramp-up time — especially for developers without distributed systems backgrounds.
- **Over-engineering risk for simple tasks**: For linear chains or single-call interactions, LangGraph adds complexity with no benefit. It is designed for genuinely complex, branching, long-running workflows.
- **Operational overhead (self-managed)**: Without LangGraph Platform, teams must build their own infrastructure for checkpointing, scaling, retries, and monitoring. This requires external systems (databases, queues, Kubernetes).
- **Debugging complexity**: Multi-agent graph executions with many nodes can produce extremely complex traces. Debugging requires LangSmith or equivalent tooling; logs alone are insufficient.
- **Ecosystem churn**: As part of the LangChain ecosystem, LangGraph has undergone significant API changes. The v1.0 release (2025) stabilised the surface but teams should budget for ongoing updates.
- **Scaling limitations**: Very high parallelism scenarios and distributed execution across many workers are not where LangGraph currently excels. Designed primarily for orchestrating sequences of LLM calls, not for massive concurrent fan-out.
- **Glue code burden**: Connecting LangGraph graphs to external data pipelines, APIs, and vector stores often requires significant custom integration code.
- **Platform lock-in (managed tier)**: LangGraph Platform (the managed deployment service) is a proprietary, paid product from LangChain AI Inc. Organisations using it for hosting become dependent on the vendor for deployment infrastructure.

---

### 2.4 Suitable Use Cases

- **Complex research agents** that plan, search, synthesise, and iterate over multiple steps before producing a final answer (e.g., deep research, report generation).
- **Customer service automation** where agents handle multi-turn conversations, look up account data, escalate to humans when needed, and resume after the human interaction is complete.
- **Code generation and review pipelines** where agents write code, run tests, analyse failures, and iterate in a loop until tests pass.
- **Document processing workflows** that route different document types to specialised sub-agents (e.g., invoice extractor, contract analyser, form filler) and aggregate results.
- **Long-running background tasks** such as overnight data enrichment jobs, compliance checking across large document sets, or scheduled monitoring agents.
- **Multi-agent coordination systems** where a supervisor agent delegates tasks to domain-specialist agents and synthesises their outputs.
- **Any workflow requiring human-in-the-loop approval** — expense approvals, content moderation, medical record review — where the agent must pause, wait, and resume.

---

### 2.5 Out-of-the-Box AWS Integrations

LangGraph's AWS integrations are delivered through the **`langchain-aws`** package and work at the orchestration layer:

| AWS Service | Integration Type | What It Enables |
| --- | --- | --- |
| **Amazon Bedrock** | LLM Backend | Any Bedrock foundation model (Anthropic Claude, Meta Llama, Amazon Nova, etc.) can serve as the reasoning engine for LangGraph nodes. |
| **Amazon Bedrock AgentCore** | Agent Runtime | Managed execution environment for LangGraph agents, providing built-in observability, session management, and scalable infrastructure without self-managed Kubernetes. |
| **Amazon Bedrock AgentCore Memory** | Long-term Memory | Managed cross-session memory store backed by Bedrock, enabling agents to recall past interactions without building a custom memory layer. |
| **Amazon Bedrock Session Management** | State / Checkpointing | Bedrock-managed session state for LangGraph checkpoints, replacing the need for a self-managed DynamoDB or Redis checkpointer. |
| **Amazon DynamoDB** | Checkpointer | Community checkpointer for persisting LangGraph state in DynamoDB tables, suitable for self-hosted deployments on AWS. |
| **Amazon MemoryDB / ElastiCache Valkey** | Checkpointer | In-memory checkpointing for high-throughput LangGraph workflows where low-latency state reads and writes are required. |
| **Amazon EKS** | Deployment Platform | LangGraph Platform (self-hosted enterprise tier) is tested and deployable on Amazon EKS, with Terraform modules provided for EKS, RDS, ElastiCache, and S3 provisioning. |
| **Amazon Bedrock Knowledge Bases** | Retriever Node | A LangGraph node can query a Bedrock Knowledge Base as a retrieval step within a larger agent graph. |
| **AWS Lambda** | Tool Node | LangGraph agents call Lambda functions as tool nodes, enabling serverless side-effects and integrations with existing AWS infrastructure. |

**Reference**: The AWS blog post [Build multi-agent systems with LangGraph and Amazon Bedrock](https://aws.amazon.com/blogs/machine-learning/build-multi-agent-systems-with-langgraph-and-amazon-bedrock/) provides end-to-end examples of multi-agent LangGraph architectures running on Bedrock.

---

---

## 3. LangSmith

### 3.1 Purpose

LangSmith is the observability, evaluation, and deployment platform for AI agents and LLM applications. The core problem it solves: LLM application behaviour is non-deterministic and hard to introspect — models decide every output at runtime based on context, making traditional software debugging and monitoring techniques insufficient. LangSmith provides full trace capture of every step an agent takes, structured evaluation frameworks to measure quality, and deployment tooling to move from experiment to production safely.

LangSmith is **framework-agnostic** — it works with LangChain and LangGraph natively, but also with OpenAI SDK, Anthropic SDK, LlamaIndex, Vercel AI SDK, and custom implementations via its tracing API and OpenTelemetry support.

---

### 3.2 Main Features

| Feature | Description |
| --- | --- |
| **Automatic Tracing** | Captures complete execution traces of every LLM call, tool invocation, and agent step with zero-instrumentation when using LangChain/LangGraph. |
| **OpenTelemetry Support** | Full end-to-end OpenTelemetry integration allows standardised tracing across heterogeneous stacks and existing observability infrastructure. |
| **Trace UI and API** | Rich web interface and programmatic API for filtering, searching, comparing, sharing, and exporting traces. |
| **Dashboards and Alerts** | Real-time monitoring dashboards for cost, latency, error rate, and quality metrics, with configurable alerts for threshold violations. |
| **Insights Agent (Polly)** | AI-powered analysis assistant that automatically clusters traces to surface usage patterns, recurring failure modes, and anomalies. |
| **Dataset Management** | Build and version datasets from production traces, curated examples, or synthetic data for offline evaluation and regression testing. |
| **LLM-as-Judge Evaluators** | Automated evaluation using LLMs to score agent outputs against user-defined quality criteria (correctness, helpfulness, tone, safety, etc.). |
| **Human Annotation Queues** | Route traces to domain experts or QA reviewers for manual labelling and scoring, with structured annotation interfaces. |
| **Pairwise Comparisons** | Side-by-side evaluation of two agent versions or prompt variants against the same inputs to identify regressions or improvements. |
| **Multi-turn Evaluation** | Captures and evaluates the full trajectory of a multi-step agent run, scoring intermediate decisions not just final outputs. |
| **Prompt Hub** | Centralised registry for versioning, sharing, and deploying prompts across the organisation, with diff views between versions. |
| **Experiment Tracking** | Runs offline experiments against datasets with full metric tracking, comparable to MLflow experiments but specialised for LLM evaluation. |
| **LangGraph Platform Integration** | LangSmith serves as the monitoring and management layer for agents deployed via LangGraph Platform, providing unified observability. |
| **Fleet (No-Code Agents)** | No-code agent builder for non-technical users to create, configure, and run agents within guardrails defined by engineering teams. |

---

### 3.3 Key Considerations Before Adopting

**Cost and Pricing:**

- Pricing has two independent components: per-seat charges and per-trace charges.
- **Developer plan**: Free, 1 seat, 5,000 traces/month.
- **Plus plan**: $39/seat/month (up to 10 seats). Base traces at $2.50 per 1,000 (14-day retention); extended traces at $5.00 per 1,000 (400-day retention).
- **Enterprise**: Custom pricing, supports self-hosted deployment on Kubernetes.
- Trace costs escalate rapidly with high-volume production workloads. An application generating 5 million traces/month incurs roughly $12,500/month in trace overages alone.
- Extended data retention (400 days) costs approximately 2x the base rate — significant for compliance-heavy organisations that need long audit trails.
- Per-seat billing starts to compound for mid-size teams; a team of 20 with moderate trace volume can exceed $3,000/month.

**Ecosystem Coupling:**

- LangSmith is tightly coupled to the LangChain/LangGraph ecosystem. Zero-instrumentation tracing only works natively with LangChain components. Other frameworks (LlamaIndex, custom code) require manual SDK instrumentation or OpenTelemetry setup.
- Migrating away from LangSmith means replacing dataset management, evaluation pipelines, and monitoring dashboards simultaneously — a significant switching cost.

**Self-Hosted Complexity:**

- Self-hosted deployment (Enterprise tier) requires: Amazon EKS cluster (v1.24+, 16 vCPU / 64 GB RAM minimum), external PostgreSQL 15+ (Amazon RDS recommended), external Redis 7+ (Amazon ElastiCache recommended), external ClickHouse 24+ for analytics, and Amazon S3 for artifact storage. This is a substantial infrastructure commitment.
- LangChain provides Terraform modules for AWS to simplify EKS, RDS, ElastiCache, and S3 provisioning, but operational burden remains significant.

**Data Residency and Compliance:**

- SaaS deployment offers US or EU data residency. Other regions require self-hosted Enterprise deployment.
- The platform is SOC 2 Type II, GDPR, and HIPAA compliant. LangChain commits to not training on customer data.

**Vendor Risk:**

- LangSmith is a commercial product from LangChain AI Inc. (VC-backed). The pricing model and feature roadmap are controlled by the vendor. There is no community-maintained alternative within the LangChain ecosystem.
- Open-source alternatives (Langfuse, Phoenix/Arize, Weave/W&B) offer similar tracing and evaluation capabilities with different trade-offs on features and ecosystem integration.

---

### 3.4 Suitable Use Cases

- **Production agent monitoring** where teams need real-time visibility into agent cost, latency, failure rates, and quality degradation across thousands of concurrent users.
- **Evaluation-driven development** workflows where every change to a prompt, model, or agent architecture is validated against a curated dataset before deployment.
- **Debugging complex agent failures** in multi-step, multi-tool, or multi-agent runs where step-by-step trace replay is the only practical way to identify the root cause.
- **Human review and annotation** pipelines where domain experts review agent outputs and their feedback feeds back into evaluation datasets and fine-tuning.
- **A/B testing of agent versions** side-by-side on real production traffic before fully rolling out a new model or prompt.
- **Compliance and auditability** requirements where every decision an agent made and why must be retained and accessible for regulatory inspection.
- **Cross-team collaboration** on agent development where product managers, data scientists, and engineers all need to inspect, annotate, and discuss agent behaviour without writing code.
- **Prompt versioning and governance** at scale, centralising all prompts used across multiple agent applications in a single governed registry.

---

### 3.5 Out-of-the-Box AWS Integrations

| AWS Service | Integration Type | What It Enables |
| --- | --- | --- |
| **Amazon EKS** | Self-Hosted Deployment | LangSmith Enterprise runs on customer-managed EKS clusters (v1.24+), tested and supported by LangChain with Terraform provisioning modules. |
| **Amazon RDS (PostgreSQL)** | Metadata Storage | LangSmith uses PostgreSQL (Amazon RDS recommended) as the primary relational store for trace metadata, datasets, and evaluation results. |
| **Amazon ElastiCache (Redis)** | Cache / Queue | Used internally by self-hosted LangSmith for caching and background job queues. |
| **Amazon S3** | Artifact Storage | Stores trace artifacts, telemetry data, and large payload blobs for self-hosted deployments. |
| **Amazon Bedrock** (via tracing) | Observed LLM Backend | LangSmith captures and traces all LLM calls made to Amazon Bedrock models, including latency, token usage, cost, and inputs/outputs, with optional request proxying through LangSmith. |
| **Amazon SageMaker** (via tracing) | Observed LLM Backend | LangSmith can trace LLM inference calls routed through SageMaker endpoints, including custom-hosted models. |
| **AWS Marketplace** | Procurement | LangSmith Enterprise is available on AWS Marketplace, enabling procurement through existing AWS spend commitments and consolidated billing. |

**Self-Hosted Infrastructure Summary**: A full self-hosted LangSmith deployment on AWS requires EKS + RDS + ElastiCache + ClickHouse (on EBS) + S3. LangChain provides Terraform modules that provision all these resources, reducing setup time but not eliminating the operational overhead.

---

---

## 4. Ecosystem Summary and Relationship Map

```text
┌─────────────────────────────────────────────────────────────┐
│                        LangSmith                            │
│         (Observability · Evaluation · Prompt Registry)      │
│         Traces every run from LangChain and LangGraph       │
└─────────────────────────┬───────────────────────────────────┘
                          │ traces
          ┌───────────────┴───────────────┐
          │                               │
┌─────────▼──────────┐         ┌──────────▼─────────┐
│     LangChain      │         │      LangGraph       │
│  (Agent Framework) │ built on │ (Orchestration Engine│
│  High-level APIs   ├─────────►  Stateful Graphs     │
│  Provider wrappers │         │  Durable Execution   │
└────────────────────┘         └──────────────────────┘
          │                               │
          └───────────────┬───────────────┘
                          │ integrates with
          ┌───────────────▼───────────────┐
          │         langchain-aws         │
          │  Bedrock · SageMaker · S3     │
          │  Kendra · OpenSearch · Lambda │
          │  DynamoDB · MemoryDB · EKS    │
          └───────────────────────────────┘
```

| Library | Open Source | Managed Service | Primary Cost Driver |
| --- | --- | --- | --- |
| LangChain | Yes (MIT) | No | Engineering time |
| LangGraph | Yes (MIT) | Yes (LangGraph Platform) | Platform subscription + compute |
| LangSmith | No (proprietary) | Yes (SaaS / self-hosted) | Seats + trace volume |

---

## 5. Sources

- [LangChain Overview - Official Docs](https://docs.langchain.com/oss/python/langchain/overview)
- [LangChain: Open Source AI Agent Framework](https://www.langchain.com/langchain)
- [LangChain and LangGraph Agent Frameworks Reach v1.0](https://blog.langchain.com/langchain-langgraph-1dot0/)
- [AWS Integrations - LangChain Docs](https://docs.langchain.com/oss/python/integrations/providers/aws)
- [langchain-aws GitHub Repository](https://github.com/langchain-ai/langchain-aws)
- [LangGraph Overview - Official Docs](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangGraph: Agent Orchestration Framework](https://www.langchain.com/langgraph)
- [LangGraph Platform GA Announcement](https://blog.langchain.com/langgraph-platform-ga/)
- [Build multi-agent systems with LangGraph and Amazon Bedrock (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/build-multi-agent-systems-with-langgraph-and-amazon-bedrock/)
- [LangGraph Alternatives 2026 - ema.ai](https://www.ema.ai/additional-blogs/addition-blogs/langgraph-alternatives-to-consider)
- [LangGraph Limitations - Latenode Community](https://community.latenode.com/t/current-limitations-of-langchain-and-langgraph-frameworks-in-2025/30994)
- [LangSmith: AI Agent & LLM Observability Platform](https://www.langchain.com/langsmith-platform)
- [LangSmith Observability Docs](https://docs.langchain.com/langsmith/observability)
- [LangSmith Pricing FAQ](https://docs.langchain.com/langsmith/pricing-faq)
- [LangSmith Plans and Pricing](https://www.langchain.com/pricing)
- [Self-hosted LangSmith on AWS - Docs](https://docs.langchain.com/langsmith/aws-self-hosted)
- [LangSmith on AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-vmzygmggk4gms)
- [End-to-End OpenTelemetry Support in LangSmith](https://blog.langchain.com/end-to-end-opentelemetry-langsmith/)
- [The True Cost of LangSmith - MetaCTO](https://www.metacto.com/blogs/the-true-cost-of-langsmith-a-comprehensive-pricing-integration-guide)
- [LangSmith Pricing 2026 - MarginDash](https://margindash.com/langsmith-pricing)
