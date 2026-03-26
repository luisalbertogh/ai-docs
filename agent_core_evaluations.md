# Amazon Bedrock AgentCore Evaluations — Technical Reference

> **Status:** Preview (not available in all regions).
> Available in: US East (N. Virginia), US West (Oregon), Asia Pacific (Sydney), Europe (Frankfurt).

---

## 1. What AgentCore Evaluations Is

### Purpose

Amazon Bedrock AgentCore Evaluations provides **automated, data-driven assessment tools** for measuring how well an AI agent performs specific tasks, handles edge cases, and maintains consistency across inputs and contexts. It is designed for **both pre-deployment quality gates and continuous production monitoring**.

Key capabilities:
- Measure end-to-end task completion (goal attainment)
- Measure accuracy of tool invocations
- Evaluate custom, domain-specific quality metrics
- Works for agents hosted in AgentCore Runtime **and** agents hosted anywhere else

### How It Differs from Standard Bedrock Model Evaluation

| Dimension | Standard Bedrock Model Evaluation | AgentCore Evaluations |
|---|---|---|
| Target | Foundation models (text generation quality) | AI agents (multi-step, tool-using workflows) |
| Data input | Static datasets / prompt-response pairs | Live production traces, OpenTelemetry spans |
| Evaluation mode | Batch / one-off | Continuous online + targeted on-demand |
| Unit of evaluation | Single prompt-response | Session → Trace → Tool call (hierarchical) |
| Instrumentation required | No | Yes (ADOT / OpenTelemetry / OpenInference) |
| Judge mechanism | Metrics + LLM-as-judge | LLM-as-judge only (reference-free) |
| Integration | Console / API against a model | Agent frameworks: Strands, LangGraph |

---

## 2. Setup and Prerequisites

### 2.1 Required Permissions and Infrastructure

1. **AWS Account** with IAM permissions for AgentCore Evaluations.
2. **Amazon Bedrock** access with model invocation permissions (required for custom evaluators).
3. **Amazon CloudWatch Transaction Search** must be enabled.
4. **AWS Distro for OpenTelemetry (ADOT)** SDK instrumenting your agent — add `aws-opentelemetry-distro` to `requirements.txt`.
5. **Python 3.10+**.

### 2.2 IAM — User/Role Policy

Attach the AWS-managed `BedrockAgentCoreFullAccess` policy, or use a scoped custom policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock-agentcore:CreateEvaluator",
                "bedrock-agentcore:GetEvaluator",
                "bedrock-agentcore:ListEvaluators",
                "bedrock-agentcore:UpdateEvaluator",
                "bedrock-agentcore:DeleteEvaluator",
                "bedrock-agentcore:CreateOnlineEvaluationConfig",
                "bedrock-agentcore:GetOnlineEvaluationConfig",
                "bedrock-agentcore:ListOnlineEvaluationConfigs",
                "bedrock-agentcore:UpdateOnlineEvaluationConfig",
                "bedrock-agentcore:DeleteOnlineEvaluationConfig",
                "bedrock-agentcore:Evaluate"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": ["iam:PassRole"],
            "Resource": "arn:aws:iam::*:role/AgentCoreEvaluationRole*",
            "Condition": {
                "StringEquals": {
                    "iam:PassedToService": "bedrock-agentcore.amazonaws.com"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:Converse",
                "bedrock:InvokeModelWithResponseStream",
                "bedrock:ConverseStream"
            ],
            "Resource": [
                "arn:aws:bedrock:*::foundation-model/*",
                "arn:aws:bedrock:*:*:inference-profile/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:DescribeIndexPolicies",
                "logs:PutIndexPolicy",
                "logs:CreateLogGroup"
            ],
            "Resource": "*"
        }
    ]
}
```

### 2.3 Service Execution Role

AgentCore Evaluations requires a dedicated IAM execution role to access resources on your behalf. Its trust policy must allow `bedrock-agentcore.amazonaws.com` to assume it, and its permissions policy must grant:

- `logs:DescribeLogGroups`, `logs:GetQueryResults`, `logs:StartQuery` — read traces from CloudWatch
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` — write results to `/aws/bedrock-agentcore/evaluations/*`
- `logs:DescribeIndexPolicies`, `logs:PutIndexPolicy` — configure log indexing on `aws/spans`
- `bedrock:InvokeModel`, `bedrock:Converse` — invoke judge models (custom evaluators)

**Trust policy:**
```json
{
    "Statement": [{
        "Effect": "Allow",
        "Principal": { "Service": "bedrock-agentcore.amazonaws.com" },
        "Action": "sts:AssumeRole",
        "Condition": {
            "StringEquals": {
                "aws:SourceAccount": "{{accountId}}",
                "aws:ResourceAccount": "{{accountId}}"
            },
            "ArnLike": {
                "aws:SourceArn": [
                    "arn:aws:bedrock-agentcore:{{region}}:{{accountId}}:evaluator/*",
                    "arn:aws:bedrock-agentcore:{{region}}:{{accountId}}:online-evaluation-config/*"
                ]
            }
        }
    }]
}
```

> **Shortcut:** The AgentCore console or Starter Toolkit can auto-create this role.

### 2.4 Service Limits

| Limit | Value |
|---|---|
| Evaluation configurations per region | 1,000 |
| Simultaneously active configurations | 100 |
| Evaluators per online evaluation config | 10 |
| CloudWatch log groups per data source | 5 |
| Filters per config | 5 |
| Token throughput (large regions) | 1M input+output tokens/minute/account |
| Results returned per `Evaluate` API call | 10 (last 10 by default) |

---

## 3. Online Evaluation

### 3.1 How It Works

Online evaluation **continuously monitors a deployed agent's live production traffic**. The service:

1. Samples a configured percentage of incoming sessions.
2. Filters sessions based on optional rules.
3. Reads OpenTelemetry spans and events from CloudWatch log groups.
4. Converts them to a unified internal format.
5. Sends them through the selected evaluators (LLM-as-judge).
6. Writes scored results to a dedicated CloudWatch log group and emits CloudWatch metrics.

The configuration is managed via the `CreateOnlineEvaluationConfig` API and is **asynchronous and persistent** — it runs continuously until disabled or deleted.

### 3.2 Data Sources

Two options when creating a configuration:

- **AgentCore Runtime endpoint** — specify the agent ID and endpoint; log groups are auto-resolved.
- **CloudWatch log groups** — specify up to 5 log group names plus the `service.name` used in `OTEL_RESOURCE_ATTRIBUTES` (format: `<agent-runtime-name>.<endpoint-name>` for Runtime-hosted agents).

### 3.3 Sampling Rate

Configured via `samplingConfig.samplingPercentage` (a float, 0.01 to 100).

- Console default: **10%**
- CLI/SDK: you set it explicitly (e.g., `1.0` = 1%, `80.0` = 80%)

### 3.4 Execution Status

- `ENABLED` — job actively processes incoming traces and scores them.
- `DISABLED` — configuration exists but is paused; no traces processed.

Set at creation with `enableOnCreate: true/false`, or toggle later via `UpdateOnlineEvaluationConfig`.

### 3.5 Evaluator Protection

When a configuration is `ENABLED`, all referenced custom evaluators are **locked**: they cannot be modified or deleted while the job is running. To modify a locked evaluator, clone it.

### 3.6 Code Samples

**AgentCore CLI:**
```bash
AGENT_ID="YOUR_AGENT_ID"
CONFIG_NAME="YOUR_CONFIG_NAME"

agentcore eval online create \
  --name $CONFIG_NAME \
  --agent-id $AGENT_ID \
  --sampling-rate 1.0 \
  --evaluator "Builtin.GoalSuccessRate" \
  --evaluator "Builtin.Helpfulness" \
  --description "Online Evaluation Config"
```

**AgentCore Starter Toolkit (Python SDK):**
```python
from bedrock_agentcore_starter_toolkit import Evaluation

eval_client = Evaluation()

config = eval_client.create_online_config(
    config_name="MY_CONFIG_NAME",          # underscores, not hyphens
    agent_id="agent_myagent-ABC123xyz",
    sampling_rate=1.0,                     # 1.0 = 1% of interactions
    evaluator_list=["Builtin.GoalSuccessRate", "Builtin.Helpfulness"],
    config_description="Online Evaluation Config",
    auto_create_execution_role=True,       # auto-creates IAM role
    enable_on_create=True
)

config_id = config['onlineEvaluationConfigId']
```

**AWS SDK (boto3):**
```python
import boto3

client = boto3.client('bedrock-agentcore-control')

response = client.create_online_evaluation_config(
    onlineEvaluationConfigName="my_agent_eval_config",
    description="Continuous evaluation of healthcare agent",
    rule={
        "samplingConfig": {"samplingPercentage": 80.0}
    },
    dataSourceConfig={
        "cloudWatchLogs": {
            "logGroupNames": ["/aws/agentcore/my-agent-traces"],
            "serviceNames": ["my_agent.DEFAULT"]
        }
    },
    evaluators=[
        {"evaluatorId": "Builtin.Helpfulness"},
        {"evaluatorId": "Builtin.GoalSuccessRate"}
    ],
    evaluationExecutionRoleArn="arn:aws:iam::123456789012:role/AgentCoreEvaluationRole",
    enableOnCreate=True
)
```

**AWS CLI:**
```bash
aws bedrock-agentcore-control create-online-evaluation-config \
    --online-evaluation-config-name "my_agent_eval_config" \
    --description "Continuous evaluation" \
    --rule '{"samplingConfig": {"samplingPercentage": 80.0}}' \
    --data-source-config '{"cloudWatchLogs": {"logGroupNames": ["/aws/agentcore/traces"], "serviceNames": ["my_agent.DEFAULT"]}}' \
    --evaluators '[{"evaluatorId": "Builtin.Helpfulness"}]' \
    --evaluation-execution-role-arn "arn:aws:iam::123456789012:role/AgentCoreEvaluationRole" \
    --enable-on-create
```

### 3.7 Session Idle Timeout

When creating via console, you can set a **session idle timeout** (1–60 minutes, default 15 min) which controls when a session is considered complete.

---

## 4. On-Demand Evaluation

### 4.1 How It Works

On-demand evaluation performs **targeted, one-shot assessments of specific agent interactions**. Unlike online evaluation, it does not run continuously — you trigger it explicitly by calling the `Evaluate` API and specifying which spans or traces to score.

Use cases:
- Testing a new custom evaluator before deploying it in online mode
- Investigating a specific customer complaint or interaction
- Validating a bug fix on a known failing session
- Analyzing historical data

### 4.2 Triggering On-Demand Evaluation

The flow is:

1. **Invoke your agent** to generate traces (using `invoke_agent_runtime` or any framework).
2. **Wait ~2 minutes** for traces to propagate to CloudWatch.
3. **Download spans** from two CloudWatch log groups:
   - `aws/spans` — span metadata (attributes, scope, timestamps)
   - `/aws/bedrock-agentcore/runtimes/<agent_id>-<endpoint>` — span events (inputs/outputs)
4. **Call `Evaluate` API** with the combined span list and one or more evaluator IDs.

### 4.3 Downloading Spans from CloudWatch

```python
import boto3
import json
from datetime import datetime, timedelta

region = "us-east-1"
agent_id = "YOUR_AGENT_ID"
session_id = "YOUR_SESSION_ID"

def query_logs(log_group_name, query_string):
    client = boto3.client('logs', region_name=region)
    start_time = datetime.now() - timedelta(minutes=60)
    end_time = datetime.now()

    query_id = client.start_query(
        logGroupName=log_group_name,
        startTime=int(start_time.timestamp()),
        endTime=int(end_time.timestamp()),
        queryString=query_string
    )['queryId']

    import time
    while (result := client.get_query_results(queryId=query_id))['status'] not in ['Complete', 'Failed']:
        time.sleep(1)
    return result['results']

def query_session_logs(log_group_name, session_id):
    query = f"""fields @timestamp, @message
    | filter ispresent(scope.name) and ispresent(attributes.session.id)
    | filter attributes.session.id = "{session_id}"
    | sort @timestamp asc"""
    return query_logs(log_group_name, query)

def extract_messages_as_json(query_results):
    return [json.loads(f['value']) for row in query_results
            for f in row if f['field'] == '@message'
            and f['value'].strip().startswith('{')]

# Download from both log groups
runtime_logs = query_session_logs(
    f"/aws/bedrock-agentcore/runtimes/{agent_id}-DEFAULT", session_id)
aws_span_logs = query_session_logs("aws/spans", session_id)

session_span_logs = (
    extract_messages_as_json(aws_span_logs) +
    extract_messages_as_json(runtime_logs)
)
```

### 4.4 Calling the Evaluate API

```python
client = boto3.client('bedrock-agentcore', region_name=region)

# Evaluate full session (trace-level)
response = client.evaluate(
    evaluatorId="Builtin.Helpfulness",
    evaluationInput={"sessionSpans": session_span_logs}
)
print(response["evaluationResults"])
```

**Targeting specific traces or spans:**

```python
# Trace-level evaluator — target specific traces
response = client.evaluate(
    evaluatorId="Builtin.Helpfulness",
    evaluationInput={"sessionSpans": session_span_logs},
    evaluationTarget={"traceIds": ["trace-id-1", "trace-id-2"]}
)

# Tool-level evaluator — target specific spans
response = client.evaluate(
    evaluatorId="Builtin.ToolSelectionAccuracy",
    evaluationInput={"sessionSpans": session_span_logs},
    evaluationTarget={"spanIds": ["span-id-1", "span-id-2"]}
)
```

**Starter Toolkit (simplest path):**
```python
from bedrock_agentcore_starter_toolkit import Evaluation

eval_client = Evaluation()
results = eval_client.run(
    agent_id="YOUR_AGENT_ID",
    session_id="YOUR_SESSION_ID",
    evaluators=["Builtin.Helpfulness", "Builtin.GoalSuccessRate"]
)

successful = results.get_successful_results()
for result in successful:
    print(f"Evaluator: {result.evaluator_name}")
    print(f"Score:     {result.value:.2f}")
    print(f"Label:     {result.label}")
    print(f"Explanation: {result.explanation}")
```

**CLI (starter toolkit):**
```bash
agentcore eval run \
  --agent-id $AGENT_ID \
  --session-id $SESSION_ID \
  --evaluator "Builtin.Helpfulness" \
  --evaluator "Builtin.GoalSuccessRate"
```

### 4.5 Result Structure

```json
{
    "evaluationResults": [
        {
            "evaluatorArn": "arn:aws:bedrock-agentcore:::evaluator/Builtin.Helpfulness",
            "evaluatorId": "Builtin.Helpfulness",
            "evaluatorName": "Builtin.Helpfulness",
            "explanation": "...step-by-step reasoning from the judge LLM...",
            "spanContext": {
                "sessionId": "...",
                "traceId": "..."
            },
            "value": 0.85,
            "label": "Generally Yes"
        }
    ]
}
```

- **Session-level evaluators:** `spanContext` contains only `sessionId`.
- **Trace-level evaluators:** `spanContext` contains `sessionId` + `traceId`.
- **Tool-level evaluators:** `spanContext` contains `sessionId` + `traceId` + `spanId`.
- Partial failures return both successful and failed results; failed entries include an error code and message.

---

## 5. OpenTelemetry / OpenInference Instrumentation

### 5.1 Terminology

| Term | Description |
|---|---|
| **Instrumentation Library** | Records telemetry (traces, spans, tool calls, model invocations). AgentCore supports **OpenTelemetry** and **OpenInference**. |
| **Instrumentation Agent** | Auto-captures telemetry without code changes. AgentCore supports **ADOT (AWS Distro for OpenTelemetry)**. |
| **Span** | Metadata about a single operation: timestamps, scope, attributes (including `session.id`). Stored in `aws/spans` log group. |
| **Event** | Payload associated with a span: inputs/outputs from models and tools. Stored in the agent's runtime log group. |
| **Session** | Logical grouping of related traces from one user/workflow. Identified by `session.id` attribute. |
| **Trace** | Complete record of a single agent execution. Contains one or more spans. |
| **Tool Call** | A span representing invocation of an external function or API. |

### 5.2 Supported Frameworks and Libraries

| Framework | Instrumentation Library |
|---|---|
| Strands Agents | Built-in (`strands.telemetry.tracer` scope) |
| LangGraph | `opentelemetry-instrumentation-langchain` |
| LangGraph | `openinference-instrumentation-langchain` |

### 5.3 Span Scopes

AgentCore Evaluations recognizes spans only from these scopes:

- `strands.telemetry.tracer`
- `opentelemetry.instrumentation.langchain`
- `openinference.instrumentation.langchain`

Spans from other scopes are ignored.

### 5.4 Required Span Structure

```json
{
    "spanId": "string",           // required
    "traceId": "string",          // required
    "parentSpanId": "string",
    "name": "string",             // required
    "scope": { "name": "string" }, // required — must match a supported scope
    "startTimeUnixNano": "epoch",  // required
    "endTimeUnixNano":   "epoch",  // required
    "attributes": {
        "session.id": "string"    // required
    },
    "resource": {
        "attributes": {
            "service.name": "agentcore_evaluation_demo.DEFAULT"
        }
    }
}
```

For `invoke_agent` spans (Strands), the key attribute is `"gen_ai.operation.name": "invoke_agent"`.

### 5.5 Event Structure

Events are associated with spans via `spanId` + `traceId`. The `body` field carries inputs and outputs:

```json
{
    "spanId": "string",
    "traceId": "string",
    "scope": { "name": "string" },
    "body": {
        "output": { "messages": [{ "content": "...", "role": "assistant" }] },
        "input":  { "messages": [{ "content": "...", "role": "user" }] }
    },
    "attributes": {
        "event.name": "string",
        "session.id": "string"
    }
}
```

> Both spans (`aws/spans`) and their corresponding events (runtime log group) are required for evaluation. If a span has a supported scope but no matching event, a `ValidationException` is thrown.

### 5.6 Service Name Configuration

For agents outside AgentCore Runtime, set:
```bash
export OTEL_RESOURCE_ATTRIBUTES="service.name=my_agent.DEFAULT"
export OTEL_EXPORTER_OTLP_LOGS_HEADERS="..."  # log destination for events
```

---

## 6. Judge Models and Metrics

### 6.1 LLM-as-Judge Mechanism

All evaluators use **reference-free LLM-as-judge**: no ground-truth labels are required. The judge LLM receives a structured prompt containing the agent's trace context (conversation history, tool calls, outputs) and produces a JSON response with:
- `reasoning` — step-by-step justification (≤250 words)
- `score` — a categorical label or numerical value from the rating scale

### 6.2 Built-in Evaluators (13 total)

Built-in evaluators use predefined models, prompts, and rating scales that cannot be modified. They use **cross-region inference (CRIS)** to automatically route to the optimal region.

| Evaluator ID | Level | What It Measures | Score Type |
|---|---|---|---|
| `Builtin.GoalSuccessRate` | Session | Did the agent complete all user goals across the full conversation? | Yes / No |
| `Builtin.Coherence` | Trace | Logical consistency and internal consistency of the response | Not At All / Not Generally / Neutral/Mixed / Generally Yes / Completely Yes |
| `Builtin.Conciseness` | Trace | Did the response use minimal words without unnecessary elaboration? | Perfectly Concise / Partially Concise / Not Concise |
| `Builtin.ContextRelevance` | Trace | How relevant is the response to the retrieved context? | (categorical) |
| `Builtin.Correctness` | Trace | Factual accuracy of the response | (categorical) |
| `Builtin.Faithfulness` | Trace | Does the response stay faithful to retrieved/provided context? | (categorical) |
| `Builtin.Harmfulness` | Trace | Does the response contain harmful content? | (categorical) |
| `Builtin.Helpfulness` | Trace | How helpful is the response to the user? | (categorical) |
| `Builtin.InstructionFollowing` | Trace | Did the agent follow its system instructions? | (categorical) |
| `Builtin.Refusal` | Trace | Did the agent appropriately refuse out-of-scope requests? | (categorical) |
| `Builtin.ResponseRelevance` | Trace | Is the response relevant to the user query? | (categorical) |
| `Builtin.Stereotyping` | Trace | Does the response contain stereotyping bias? | (categorical) |
| `Builtin.ToolParameterAccuracy` | Tool | Were the tool parameters faithful to the user request? | (categorical) |
| `Builtin.ToolSelectionAccuracy` | Tool | Did the agent select the correct tool for the task? | (categorical) |

ARN format for built-in evaluators:
```
arn:aws:bedrock-agentcore:::evaluator/Builtin.Helpfulness
```

### 6.3 Custom Evaluators

Custom evaluators let you choose:
- **Judge model** — any Amazon Bedrock foundation model (specified by model ARN)
- **Evaluation instructions** — a prompt template with placeholders (`{context}`, `{assistant_turn}`, `{tool_turn}`, `{available_tools}`)
- **Rating scale** — numerical (with labels and definitions) or categorical
- **Evaluation level** — `SESSION`, `TRACE`, or `TOOL_CALL`

ARN format:
```
arn:aws:bedrock-agentcore:<region>:<account>:evaluator/<evaluator-id>
```

**Example custom evaluator config (`custom_evaluator_config.json`):**
```json
{
    "llmAsAJudge": {
        "modelConfig": {
            "bedrockEvaluatorModelConfig": {
                "modelId": "global.anthropic.claude-sonnet-4-5-20250929-v1:0",
                "inferenceConfig": {
                    "maxTokens": 500,
                    "temperature": 1.0
                }
            }
        },
        "instructions": "You are evaluating the quality of the Assistant's response...\n\nContext: {context}\nCandidate Response: {assistant_turn}",
        "ratingScale": {
            "numerical": [
                { "value": 1.0,  "label": "Very Good", "definition": "Completely accurate." },
                { "value": 0.75, "label": "Good",      "definition": "Mostly accurate." },
                { "value": 0.50, "label": "OK",        "definition": "Partially correct." },
                { "value": 0.25, "label": "Poor",      "definition": "Significant errors." },
                { "value": 0.0,  "label": "Very Poor", "definition": "Completely incorrect." }
            ]
        }
    }
}
```

**Create via AWS SDK:**
```python
import boto3, json

client = boto3.client('bedrock-agentcore-control')

with open('custom_evaluator_config.json') as f:
    evaluator_config = json.load(f)

response = client.create_evaluator(
    evaluatorName="my_custom_evaluator",
    level="TRACE",               # SESSION | TRACE | TOOL_CALL
    evaluatorConfig=evaluator_config
)
```

**Create via CLI:**
```bash
aws bedrock-agentcore-control create-evaluator \
    --evaluator-name 'my_custom_evaluator' \
    --level TRACE \
    --evaluator-config file://custom_evaluator_config.json
```

### 6.4 Cross-Region Inference (CRIS)

Built-in evaluators automatically use CRIS to optimize model availability and compute. Your data stays in the originating region, but inference may run in another region (transmitted encrypted). To avoid CRIS, use a custom evaluator and specify a single-region model ID.

---

## 7. How Results Are Surfaced

### 7.1 CloudWatch Log Groups

Online evaluation results are written to:
```
/aws/bedrock-agentcore/evaluations/results/<online-evaluation-config-id>
```

Results follow **OpenTelemetry semantic conventions for GenAI evaluation result events** and are stored in JSON format. Each result is parented to the original `spanId` and references the original `traceId` and `sessionId`.

### 7.2 CloudWatch Metrics

Evaluation scores are emitted as **CloudWatch metrics** under namespace `Bedrock-AgentCore/Evaluations`. You can filter by evaluator type and evaluation label.

**To view in console:**
1. Open CloudWatch → Metrics → All Metrics
2. Browse → `Bedrock-AgentCore/Evaluations`
3. Select dimension combinations (evaluator type, label, etc.)

### 7.3 CloudWatch GenAI Observability Console

1. Open CloudWatch
2. Navigate: GenAI Observability → Bedrock AgentCore
3. Under **Agents**, select the agent and endpoint
4. Open the **Evaluations** tab

This provides:
- Aggregated score dashboards
- Quality trend tracking over time
- Drill-down into low-scoring sessions
- Full interaction flow from input to output

### 7.4 AgentCore Console

1. Open Amazon Bedrock AgentCore console
2. Left navigation → **Evaluation**
3. Select an evaluation configuration to view its detail page (includes the CloudWatch log group link)

---

## 8. Summary of Key APIs

| API | Client | Purpose |
|---|---|---|
| `create_online_evaluation_config` | `bedrock-agentcore-control` | Create continuous monitoring config |
| `get_online_evaluation_config` | `bedrock-agentcore-control` | Retrieve config details |
| `list_online_evaluation_configs` | `bedrock-agentcore-control` | List all configs |
| `update_online_evaluation_config` | `bedrock-agentcore-control` | Enable/disable or modify config |
| `delete_online_evaluation_config` | `bedrock-agentcore-control` | Remove config |
| `create_evaluator` | `bedrock-agentcore-control` | Create a custom evaluator |
| `get_evaluator` | `bedrock-agentcore-control` | Retrieve evaluator definition |
| `list_evaluators` | `bedrock-agentcore-control` | List custom evaluators |
| `update_evaluator` | `bedrock-agentcore-control` | Modify an unlocked custom evaluator |
| `delete_evaluator` | `bedrock-agentcore-control` | Delete an unlocked custom evaluator |
| `evaluate` | `bedrock-agentcore` (data plane) | Trigger on-demand evaluation |

---

## 9. Reference Links

- Overview: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations.html
- How it works: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/how-it-works-evaluations.html
- Evaluation types: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations-types.html
- Terminology: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations-terminology.html
- Built-in evaluators (prompt templates): https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/prompt-templates-builtin.html
- Custom evaluators: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/custom-evaluators.html
- Online evaluation: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/online-evaluations.html
- Create online evaluation: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/create-online-evaluations.html
- Results and output: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/results-and-output.html
- On-demand evaluation: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/on-demand-evaluations.html
- Getting started (on-demand): https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/getting-started-on-demand.html
- Understanding input spans: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/understanding-input-spans.html
- Prerequisites: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations-prerequisites.html
- Cross-region inference: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/evaluations-cross-region-inference.html
- Sample code: https://github.com/awslabs/amazon-bedrock-agentcore-samples/tree/main/01-tutorials/07-AgentCore-evaluations
- What's new announcement: https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-bedrock-agentcore-policy-evaluations-preview/
