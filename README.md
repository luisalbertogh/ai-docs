# 🤖 AI Documentation Hub

> Guides, summaries, and best practices on AI agent development, evaluation, observability, and platform engineering — with a focus on AWS.

---

## 📚 Contents

| Topic | Files |
| --- | --- |
| 🏗️ [AWS Services & Architecture](#aws-services--architecture) | Agentic AI, RAG, AgentCore |
| 🧰 [Frameworks & SDKs](#frameworks--sdks) | Strands, LangChain, LangGraph, LangSmith |
| 🔬 [Evaluation & Observability](#evaluation--observability) | RAGAS, TruLens, MLflow, AgentCore Evaluations |
| 🚀 [DevOps & Platform Engineering](#devops--platform-engineering) | CI/CD, security, reliability, governance |

---

## AWS Services & Architecture

### Agentic AI on AWS

End-to-end guide to building AI agents on AWS with Amazon Bedrock, AgentCore, and supporting services.

- 📄 [AWS Agentic AI](aws_agentic_ai.md) — Overview of Bedrock services (model catalog, fine-tuning, agents, AgentCore) and evaluation capabilities. Includes best practices and use cases.
- 📄 [AgentCore Deployment Howto](agent_core_howto.md) — Step-by-step journey from local development to production-ready deployment on Amazon Bedrock AgentCore Runtime.
- 📄 [AgentCore Evaluations Reference](agent_core_evaluations.md) — Technical deep-dive into AgentCore Evaluations (Preview): setup, online/on-demand modes, built-in evaluators, and results surfacing.

### RAG Applications

- 📄 [AWS RAG Application Development](aws_rag.md) — Architectural patterns, pipeline design, chunking strategies, vector stores, and embedding model selection for RAG on AWS.

---

## Frameworks & SDKs

### AWS-Native

- 📄 [Strands Agents SDK](strands.md) — AWS open-source, code-first framework for building autonomous agents. Covers core concepts, tool use, multi-agent patterns, and production deployment.

### LangChain Ecosystem

- 📄 [LangChain, LangGraph & LangSmith](langchain_et_al.md) — Comprehensive guide to the three core LangChain libraries:
  - **LangChain** — the LLM application framework (1,000+ integrations, RAG, tool calling)
  - **LangGraph** — stateful, graph-based agent orchestration with durable execution and human-in-the-loop
  - **LangSmith** — observability, evaluation, and prompt governance platform

---

## Evaluation & Observability

### Evaluation Frameworks

- 📄 [Agent Evaluation Practices](agent_evaluation.md) — LLM selection criteria, evaluation techniques (LLM-as-judge, programmatic, HITL, tracing) with sample metrics for each. Includes AWS-native evaluation services.
- 📄 [RAGAS](ragas.md) — Open-source evaluation framework specialised for RAG pipelines. Metrics: faithfulness, answer relevancy, context precision/recall, and more.
- 📄 [TruLens](trulens.md) — Open-source evaluation and tracing platform (by TruEra/Snowflake). Supports LangChain, LlamaIndex, and custom implementations.

### Experiment Tracking & Model Lifecycle

- 📄 [MLflow](mlflow.md) — Open-source ML lifecycle platform. Covers experiment tracking, model registry, projects, recipes, and MLflow 3.x GenAI tracing. Includes AWS/SageMaker integration guide.

---

## DevOps & Platform Engineering

- 📄 [DevOps Guide for AI on AWS](aws_devops_ai.md) — Practitioner guide for platform engineers managing AI agent workloads. Covers reliability, security, responsible AI, observability, CI/CD, disaster recovery, cost management, and model governance.

---

## 🔗 External Documentation

### ☁️ AWS

- [AWS Partner Documentation](https://partnercentral.awspartner.com/partnercentral2/s/guideslistview?keyword=bedrock) — Official AWS documentation for partners, searchable by topic.
- [AWS Agentic AI Partner Resources](https://partnercentral.awspartner.com/partnercentral2/s/article?category=Generative_AI_Center_of_Excellence&article=Amazon-Agentic-AI) — AWS Gen AI technical resources for partners.
- [AWS Bedrock Skill Builder](https://skillbuilder.aws/search?page=1&searchText=Amazon+Bedrock) — Online courses on Amazon Bedrock.
- [Agentic AI Partner Trainings](https://partnercentral.awspartner.com/partnercentral2/s/article?category=Training_for_you_and_your_team&article=AWS-Partner-Courses) — AWS Partner training catalogue for Agentic AI.
- [AWS Vector Databases Guide](https://docs.aws.amazon.com/vector-databases/) — Official guide to vector database options on AWS.

### 🔧 AWS Bedrock AgentCore

- [AgentCore Runtime Howtos](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-how-it-works.html) — How the AgentCore Runtime works.
- [AgentCore + Strands Starter Kit](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-get-started-toolkit.html) — Getting started toolkit combining AgentCore and Strands.

### 🧵 Strands Agents

- [Introducing Strands Agents](https://aws.amazon.com/blogs/opensource/introducing-strands-agents-an-open-source-ai-agents-sdk/) — AWS blog post announcing the Strands open-source SDK.
- [Strands Python Quickstart](https://strandsagents.com/docs/user-guide/quickstart/python/#_top) — Official Strands documentation quickstart.

### 📊 Model Evaluation

- [RAGAS Concepts](https://docs.ragas.io/en/stable/concepts/) — Official RAGAS documentation on evaluation concepts and metrics.

### 🎓 Trainings

- [Cognizant Role-Based Gen AI Trainings](https://cognizantonline.sharepoint.com/sites/Business/SitePages/Gen-AI-Hub-Training-Resources.aspx#role-based-learnings-on-gen-ai) — Role-based Gen AI learning paths, including GenAI on AWS.
