# AWS Strands Agents SDK Summary

The **Strands Agents SDK** is an open-source, code-first framework designed by AWS to build production-ready, autonomous AI agents. It prioritizes model-driven reasoning, where the LLM (Foundation Model) orchestrates tasks and tool-use with minimal boilerplate.

## Core Concepts

The Strands Agents SDK is built on several fundamental concepts that enable a model-driven approach to AI development. Instead of rigid state machines, Strands relies on the LLM's inherent reasoning to manage the agent loop.

### Agent Loop and Coordination

The **Agent Loop** is the engine of the Strands SDK. It coordinates three core components: the **Model Provider**, the **System Prompt**, and the **Toolbelt**. The model acts as the orchestrator, reasoning through tasks and deciding which tools to call or when to synthesize a final response. This allows agents to handle unexpected scenarios and adapt dynamically to information discovered during execution.

![Agent loop](./pics/strands_loop.png)

### Prompts and State Management

**System Prompts** establish the agent's persona, goals, and operational boundaries. They guide the model's reasoning process without requiring hardcoded logic. **Session State Management** allows agents to maintain context across multi-turn conversations. Strands supports **Durable Sessions**, which enable the automatic persistence and restoration of conversation history and internal state, ensuring continuity even across disparate interactions or system restarts.

### Hooks and Interceptors

**Hooks** provide a way to inject custom logic at specific points in the agent's lifecycle (e.g., pre-processing a user query or post-processing a tool's output). This is essential for implementing custom logging, security checks, or data transformation without modifying the core agent logic.

## Model Providers

Strands is designed to be model-agnostic, supporting a wide range of foundation models through diverse providers.

- **Highlights:** Flexibility to choose the best model for a specific task (e.g., a fast model for routing and a large model for reasoning).
- **Purpose:** Decouples the agent's logic from the specific API of a model provider, allowing for easy model switching or multi-model architectures.
- **Available Options:**
  - **Amazon Bedrock:** Native integration for models like Claude, Nova, Llama, and Titan.
  - **Amazon SageMaker:** Support for custom-hosted models on SageMaker endpoints.
  - **Third-Party APIs:** Integration with OpenAI, Anthropic, Mistral, and Cohere.
  - **Local Models:** Capabilities to run agents against local model servers like Ollama for development and privacy.

## Multi-Agent Patterns

Strands excels at complex orchestration by allowing multiple agents to collaborate using different structural patterns.

- **Highlights:** Enables the decomposition of complex problems into smaller, manageable tasks handled by specialist agents.
- **Purpose:** To scale agentic systems beyond the capabilities of a single LLM and to implement governed workflows.
- **Available Patterns:**
  - **Graph:** A deterministic, directed workflow where execution follows predefined paths and conditional logic.
  - **Swarm:** A self-organizing pattern where agents autonomously decide when to hand off tasks to one another.
  - **Workflow:** A structured sequence of tasks with explicit transitions, often used for linear business processes.
  - **Agents-as-Tools:** A hierarchical pattern where an orchestrator agent treats other specialist agents as if they were simple tools.

## Tools and Extensions

Tools extend an agent's capabilities by allowing it to interact with external systems and data.

- **Highlights:** Seamless integration of Python functions, specialized SDKs, and remote services.
- **Purpose:** To provide agents with "hands" to perform actions like querying databases, searching the web, or calling internal APIs.
- **Available Options:**
  - **Local Python Tools:** Functions decorated with the `@tool` attribute for immediate execution.
  - **MCP Tools:** Integration with **Model Context Protocol** servers to access a vast ecosystem of community and enterprise tools.
  - **Community Tools:** Pre-built connectors for popular services like Google Search, GitHub, and Slack.

## Deployment

Strands agents are designed to be portable and production-ready across various AWS environments.

- **Highlights:** Choice between fully managed runtimes or custom infrastructure depending on control and latency requirements.
- **Purpose:** To transition agents from local prototypes to secure, scalable, and monitored enterprise applications.

### Deployment Patterns

Strands supports six deployment options ranging from fully managed serverless to bare-metal control. Built-in integration guides are available for each.

| Deployment Option | Best For | Pros | Cons |
| --- | --- | --- | --- |
| **Amazon Bedrock AgentCore** | Scalable, managed agent runtimes | Purpose-built for AI agents; serverless with no infra management; built-in session isolation and observability; auto-scales | AWS lock-in; less flexibility for custom configurations; potentially higher cost at sustained high throughput |
| **AWS Lambda** | Short-lived, event-driven, or batch workloads | True serverless pay-per-use; minimal infra overhead; great for async/batch triggers | 15-minute execution timeout limits long-running agents; cold start latency; limited memory; streaming requires workarounds |
| **AWS Fargate** | Interactive apps requiring real-time streaming | Containerized and portable; native streaming support; handles high concurrency; no server management | Higher baseline cost than Lambda; requires container expertise; slower startup than Lambda |
| **AWS App Runner** | Interactive apps with simplified ops | Containerized with streaming support; automated deployment, scaling, and load balancing; simple setup from a container image or source repo | Less control over scaling behavior than ECS/EKS; fewer customization options for advanced networking |
| **Amazon EKS** | Complex, high-concurrency workloads | Full Kubernetes ecosystem; streaming support; maximum scalability; rich operational tooling | High operational complexity; steep learning curve; significant infra management overhead |
| **Amazon EC2** | High-volume or specialized infrastructure | Maximum control and flexibility; supports GPU/specialized hardware; no cold starts; persistent connections | Manual scaling and management; always-on cost even at idle; highest operational burden |

### Choosing a Pattern

- **Start with Bedrock AgentCore** if you want the least operational overhead and a runtime purpose-built for AI agents.
- **Use Lambda** for lightweight agents triggered by events (e.g., S3 uploads, API Gateway) where execution time is short.
- **Use Fargate or App Runner** when your agent needs to stream responses in real time or sustain concurrent sessions without Kubernetes complexity.
- **Use EKS** if you are already operating a Kubernetes platform and need fine-grained control over scheduling, networking, or scaling policies.
- **Use EC2** only when you require specialized hardware (e.g., GPU inference), maximum throughput control, or have constraints that prevent containerized or managed options.

### Production Considerations

When moving from prototype to production, the following practices apply regardless of the deployment target:

- **Explicit tool lists** — always declare only the tools the agent needs; avoid dynamic loading in production.
- **Conversation management** — use `SlidingWindowConversationManager` to prevent context window overflow under sustained load.
- **Streaming** — leverage `stream_async()` for lower perceived latency in interactive deployments.
- **Observability** — integrate OpenTelemetry tracing and route metrics to CloudWatch or a compatible backend.
- **Security** — validate inputs, sanitize outputs, and apply Amazon Bedrock Guardrails or AgentCore Policy (Cedar) to enforce runtime boundaries.

## Observability, Evaluation, and Safety

Building trusted agents requires deep visibility and rigorous safety controls.

- **Highlights:** Standardized telemetry and policy-based governance.
- **Purpose:** To monitor agent performance, evaluate output quality, and ensure compliance with business rules.
- **Available Features:**
  - **Observability:** Native OpenTelemetry integration for tracing "chains of thought" and tool interactions.
  - **Evaluation:** Built-in "LLM-as-a-Judge" metrics for automated quality assessments (Helpfulness, Correctness, etc.).
  - **Safety & Security:** Integration with **Amazon Bedrock Guardrails** and **AgentCore Policy (Cedar)** to enforce runtime boundaries.

### Evaluation in Strands

Strands ships a dedicated evaluation package (`strands-agents-evals`) that provides a structured, LLM-as-a-judge framework to assess agent quality at multiple levels of granularity.

**Core concepts:**

- **Cases and Experiments** — test inputs are defined as `Case` objects and grouped into `Experiment` runs; each experiment executes one or more evaluators against the agent's responses and traces.
- **Evaluation levels** — evaluators operate at three scopes: `OUTPUT_LEVEL` (single response), `TRACE_LEVEL` (turn-by-turn tool and model calls), and `SESSION_LEVEL` (full conversation goal achievement).
- **Structured results** — every evaluator returns a typed `EvaluationOutput` with a numeric score, a pass/fail boolean, and a reasoning explanation.

**Built-in evaluators:**

| Evaluator | Level | What it measures |
| --- | --- | --- |
| `OutputEvaluator` | Output | Subjective response quality via a custom rubric (safety, tone, relevance, completeness) |
| `HelpfulnessEvaluator` | Trace | Whether the response effectively addressed the user's needs |
| `FaithfulnessEvaluator` | Trace | Hallucination detection — whether claims are grounded in the conversation history |
| `ToolSelectionAccuracyEvaluator` | Trace | Whether the correct tools were selected at the right points |
| `ToolParameterAccuracyEvaluator` | Trace | Whether tool arguments were derived from context rather than fabricated |
| `TrajectoryEvaluator` | Session | Quality of the agent's multi-step reasoning and action sequence |
| `InteractionsEvaluator` | Session | Message flow and dependencies in multi-agent scenarios |
| `GoalSuccessRateEvaluator` | Session | End-to-end success — whether the user's overall goal was achieved |

**Key capabilities:**

- **Remote evaluation** — traces from a deployed AgentCore Runtime agent can be fetched via `CloudWatchProvider` or `LangfuseProvider` and evaluated post-hoc, without re-running the agent.
- **ExperimentGenerator** — an LLM-powered utility that automatically generates test cases and evaluation rubrics from a description of the agent's purpose.
- **Simulators** — generate synthetic multi-turn conversations to stress-test agent behavior before going to production.
- **Custom evaluators** — extend the base `Evaluator` class to implement domain-specific scoring logic.
- **OpenTelemetry integration** — evaluation feeds directly off the same OTel traces used for observability; the `StrandsInMemorySessionMapper` converts spans into typed session objects (`AgentInvocationSpan`, `InferenceSpan`, `ToolExecutionSpan`) that evaluators consume.

## Getting Started from Scratch (Python)

### 1. Installation

Install the necessary packages using pip:

```bash
pip install strands-agents bedrock-agentcore
```

### 2. Simple Implementation Example

Here is a basic example of an agent that uses a calculator tool:

```python
from strands.agents import Agent
from strands.tools import tool

# Define a tool
@tool
def calculator(expression: str) -> str:
    """Evaluates a mathematical expression."""
    try:
        # Note: eval() is used for simplicity; use a safer evaluation in production.
        return str(eval(expression))
    except Exception as e:
        return f"Error: {e}"

# Initialize the agent
agent = Agent(
    model_id="anthropic.claude-3-5-sonnet-20240620-v1:0",
    tools=[calculator],
    system_prompt="You are a helpful assistant with calculation abilities."
)

# Invoke the agent
response = agent.invoke("What is 1234 * 5678?")
print(response.content)
```

## AWS Bedrock AgentCore Integrations

AWS Bedrock AgentCore provides a set of managed services that integrate natively with Strands Agents to build, deploy, and operate production-grade AI agents:

- **[Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agents-tools-runtime.html)** — Use this to host and run your Strands agent in production without managing infrastructure; the runtime handles invocation, session isolation, and scaling automatically.
- **[Memory](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory.html)** — Use this so your agent can recall what happened in past conversations and accumulate user-specific knowledge over time, going beyond the context window of a single session.
- **[Gateway](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)** — Use this to give your agent access to external APIs and services as MCP tools, without writing custom integration code or handling authentication per tool.
- **[Identity](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity.html)** — Use this so your agent can act on behalf of a user or assume an IAM role to call AWS services and third-party APIs securely, with credentials managed outside of agent code.
- **[Code Interpreter](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/code-interpreter-tool.html)** — Use this when your agent needs to generate and run code as part of its reasoning, such as data analysis, computation, or file processing, in a safe isolated environment.
- **[Browser](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/browser-tool.html)** — Use this when your agent must interact with web-based applications or extract information from sites that require navigation, login, or form submission.
- **[Observability](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html)** — Use this to trace every step your agent takes during execution, diagnose failures, and monitor latency and tool usage in production.
- **[Evaluations](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations.html)** — Use this to measure and compare your agent's response quality and tool-use accuracy against defined test cases before or after deploying changes.
- **[Policy](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/policy.html)** — Use this to enforce boundaries on what your agent is allowed to do, restricting which tools it can invoke or which data it can access based on explicit rules.

### Developing for AWS Bedrock AgentCore

To deploy a Strands agent to the **Amazon Bedrock AgentCore Runtime**, you typically follow these steps:

### Step-by-Step Process

1. **Define Agent Logic:** Use the Strands SDK to define your agent, tools, and system prompt (as shown above).
2. **Expose Endpoints:** Wrap your agent in a web server (e.g., FastAPI) that exposes the required AgentCore Runtime endpoints:
    - `POST /invocations`: Where the agent receives and processes messages.
    - `GET /ping`: For health checks.
3. **Configure Memory:** Transition from local state to persistent memory by integrating `agentcore-memory`.
4. **Containerize:** Build an **ARM64** Docker image containing your application.
5. **Deploy:** Use the `agentcore deploy` CLI command or AWS SDK to push the image to ECR and create a Runtime Agent.

### Example Runtime Wrapper (Brief)

```python
from fastapi import FastAPI
from strands.agents import Agent

app = FastAPI()
agent = Agent(model_id="...")

@app.post("/invocations")
async def invoke(request: dict):
    user_message = request.get("input")
    response = agent.invoke(user_message)
    return {"output": response.content}

@app.get("/ping")
async def ping():
    return {"status": "ok"}
```

## References

- [Strands Agents SDK Documentation](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/strands-sdk-memory.html)
- [Building AI Agents on AWS Serverless (AWS Blog)](https://aws.amazon.com/blogs/compute/effectively-building-ai-agents-on-aws-serverless/)
- [Strands Agents SDK: Deep Dive into Architectures (AWS Blog)](https://aws.amazon.com/blogs/machine-learning/strands-agents-sdk-a-technical-deep-dive-into-agent-architectures-and-observability/)
