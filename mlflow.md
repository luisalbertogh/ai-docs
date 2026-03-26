# MLflow: Comprehensive Guide

> MLflow is the **open-source AI engineering platform** for agents, LLMs, and ML models. It enables teams of all sizes to debug, evaluate, monitor, and optimize production-quality AI applications while controlling costs and managing access to models and data.

---

## 1. What Is MLflow?

MLflow is an open-source platform originally created by Databricks in 2018 and now maintained by the open-source community. Its core purpose is to manage the **end-to-end machine learning lifecycle**: from experimentation and reproducibility, to deployment and monitoring in production.

MLflow addresses four fundamental pain points in ML development:

- **Tracking**: It is hard to keep track of experiments — which parameters, code version, and data produced a given model.
- **Reproducibility**: It is hard to reproduce results, especially across teams and environments.
- **Deployment**: There is no standard way to package and move a model from development to production.
- **Model management**: There is no central store to manage, version, and audit models throughout their lifecycle.

MLflow 3.0 (current major version as of 2026) extends the platform into generative AI and LLM observability, introducing **tracing**, **prompt registries**, and **LoggedModel** entities for agents and GenAI applications.

---

## 2. Main Features

### 2.1 MLflow Tracking

**Purpose**: Records and queries experiments — parameters, metrics, tags, and output artifacts for every training run.

**Key concepts**:

- **Experiments**: A logical grouping of runs (e.g., one experiment per project or dataset).
- **Runs**: A single execution of training code. Each run records:
  - **Parameters** (`mlflow.log_param`): Hyperparameters and configuration values.
  - **Metrics** (`mlflow.log_metric`): Numeric measurements such as accuracy, loss, RMSE — optionally step-indexed for epoch-by-epoch tracking.
  - **Artifacts**: Output files such as model weights, plots, data files, and evaluation reports.
  - **Tags**: Arbitrary key-value metadata (e.g., model type, dataset version).
  - **Source code**: Git commit hash and source file.
- **Autologging**: One-line integration (`mlflow.autolog()`) that automatically captures parameters and metrics for popular frameworks (scikit-learn, TensorFlow, PyTorch, XGBoost, LightGBM, etc.).
- **Tracking UI**: A web-based interface for browsing, filtering, and comparing runs visually.
- **Tracking Server**: Can run locally, on a remote server, or as a fully managed service (e.g., on Amazon SageMaker).

---

### 2.2 MLflow Projects

**Purpose**: Packages ML code in a reusable and reproducible format, so any developer or automated system can run it reliably.

**Key concepts**:

- Projects are defined by an `MLproject` file at the root of a directory or Git repository.
- Specify the **entry points** (scripts to run), their **parameters**, and the **environment** (conda, Docker, or virtualenv).
- Supports running projects locally or on remote infrastructure (Databricks, Kubernetes).
- Enables chaining of projects into multi-step workflows.

```yaml
# MLproject example
name: my_project
conda_env: conda.yaml
entry_points:
  main:
    parameters:
      alpha: {type: float, default: 0.5}
      l1_ratio: {type: float, default: 0.1}
    command: "python train.py --alpha {alpha} --l1_ratio {l1_ratio}"
```

```bash
# Run a project from a Git repo
mlflow run https://github.com/mlflow/mlflow-example -P alpha=0.4
```

---

### 2.3 MLflow Models

**Purpose**: A standard format for packaging machine learning models so they can be used by a variety of downstream tools and inference platforms.

**Key concepts**:

- A model is saved as a directory with a standard `MLmodel` file describing its **flavors**: the frameworks it can be loaded with (e.g., `python_function`, `sklearn`, `tensorflow`, `pytorch`, `onnx`).
- The **`python_function` (pyfunc) flavor** is the universal interface: any model saved with MLflow can be loaded with `mlflow.pyfunc.load_model()` and called with `.predict()`, regardless of the original framework.
- Models store their **dependencies** (conda environment or `requirements.txt`) to ensure consistent inference environments.
- **Model Signatures**: Optional schema definitions for input/output data, enabling runtime validation.
- **Model Evaluation**: Automated evaluation tools (`mlflow.evaluate()`) integrated with tracking for comparing model quality across runs.

---

### 2.4 MLflow Model Registry

**Purpose**: A centralized hub for managing, versioning, and governing the full lifecycle of ML models — from experimentation through staging to production.

**Key concepts**:

- **Registered Models**: Named entries in the registry, each containing one or more **model versions**.
- **Model Versions**: Every time a new model is registered under the same name, it receives a new version number. Each version is linked back to the MLflow run that produced it, enabling full reproducibility.
- **Aliases**: Named pointers to specific versions (e.g., `@champion`, `@challenger`, `@staging`). These replace the older deprecated stage transitions (Staging / Production / Archived).
- **Tags**: Key-value metadata on models and versions for searchability and workflow status (e.g., `validation_status=approved`).
- **Model Lineage**: The registry tracks the source run, enabling audit trails from deployed model back to training data and code.
- **UI and REST API**: Models can be browsed, compared, and promoted through the MLflow UI or programmatically via `MlflowClient`.

---

### 2.5 MLflow Recipes (formerly Pipelines)

**Purpose**: Provides opinionated, production-ready templates for common ML workflows to enforce best practices and accelerate development.

**Key concepts**:

- Recipes (previously called Pipelines in MLflow < 2.x) are structured, multi-step workflows with a defined directory layout.
- Built-in **recipe templates** for:
  - **Regression**: Data ingestion → splitting → transformation → training → evaluation → registration.
  - **Classification**: Similar structure for classification tasks.
- Each step is individually executable and cacheable — only re-runs steps where inputs have changed (similar to a Makefile).
- Recipes enforce MLOps best practices: data validation, model evaluation thresholds, and registry promotion gates.
- Integrated with MLflow Tracking: every recipe run is automatically logged as an MLflow experiment run.

```bash
# Run the full recipe
mlflow recipes run --profile local

# Run a single step
mlflow recipes run --step train --profile local
```

---

### 2.6 MLflow Tracing (MLflow 3.x — GenAI & Agents)

**Purpose**: End-to-end observability for LLM applications, agents, and multi-step AI pipelines.

**Key concepts**:

- Records **inputs, outputs, and metadata at every step** of an AI application's execution.
- Enables debugging of complex agentic workflows (tool calls, RAG retrieval steps, LLM calls).
- Auto-instrumentation with one-line integrations for LangChain, LlamaIndex, OpenAI, Anthropic, and more.
- Traces are searchable and filterable in the MLflow UI by status, tags, user, environment, or execution time.
- **Prompt Registry**: Version, track, and reuse prompt templates across an organization.

---

## 3. Practical Use Cases with Python Code

### Use Case 1: Experiment Tracking

Train a Random Forest classifier across multiple hyperparameter configurations and track all results in MLflow for comparison.

```python
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score, f1_score

# Load data
X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Point to your tracking server (local or remote)
mlflow.set_tracking_uri("http://localhost:5000")  # or an S3-backed SageMaker ARN
mlflow.set_experiment("iris-classification")

# Hyperparameter grid to explore
param_grid = [
    {"n_estimators": 50,  "max_depth": 3},
    {"n_estimators": 100, "max_depth": 5},
    {"n_estimators": 200, "max_depth": None},
]

for params in param_grid:
    with mlflow.start_run(run_name=f"rf-n{params['n_estimators']}-d{params['max_depth']}"):
        # Log hyperparameters
        mlflow.log_params(params)
        mlflow.set_tag("framework", "scikit-learn")
        mlflow.set_tag("dataset", "iris")

        # Train model
        model = RandomForestClassifier(**params, random_state=42)
        model.fit(X_train, y_train)

        # Evaluate
        y_pred = model.predict(X_test)
        accuracy = accuracy_score(y_test, y_pred)
        f1 = f1_score(y_test, y_pred, average="weighted")

        # Log metrics
        mlflow.log_metric("accuracy", accuracy)
        mlflow.log_metric("f1_score", f1)

        # Log model with input/output signature
        signature = mlflow.models.infer_signature(X_train, model.predict(X_train))
        mlflow.sklearn.log_model(
            model,
            artifact_path="model",
            signature=signature,
            registered_model_name="iris-random-forest"  # auto-registers in the Model Registry
        )

        print(f"Run complete | n_estimators={params['n_estimators']} | accuracy={accuracy:.4f}")
```

After running this script, open the MLflow UI (`mlflow ui`) and navigate to the `iris-classification` experiment to compare all runs visually by accuracy, F1, and parameters.

---

### Use Case 2: Model Registry — Version Management and Alias Promotion

After tracking experiments, promote the best model to production using the Model Registry.

```python
import mlflow
from mlflow.tracking import MlflowClient

mlflow.set_tracking_uri("http://localhost:5000")
client = MlflowClient()

MODEL_NAME = "iris-random-forest"

# --- Step 1: Find the best run by accuracy ---
experiment = client.get_experiment_by_name("iris-classification")
runs = client.search_runs(
    experiment_ids=[experiment.experiment_id],
    order_by=["metrics.accuracy DESC"],
    max_results=1
)
best_run = runs[0]
best_run_id = best_run.info.run_id
print(f"Best run: {best_run_id} | accuracy={best_run.data.metrics['accuracy']:.4f}")

# --- Step 2: Register the best model (if not already registered via log_model) ---
model_uri = f"runs:/{best_run_id}/model"
model_version = mlflow.register_model(model_uri=model_uri, name=MODEL_NAME)
print(f"Registered version: {model_version.version}")

# --- Step 3: Add metadata tags ---
client.set_model_version_tag(
    name=MODEL_NAME,
    version=model_version.version,
    key="validation_status",
    value="approved"
)
client.set_model_version_tag(
    name=MODEL_NAME,
    version=model_version.version,
    key="approved_by",
    value="data-science-team"
)

# --- Step 4: Promote to production using an alias ---
# Set @champion alias to this version (replaces deprecated stage transitions)
client.set_registered_model_alias(
    name=MODEL_NAME,
    alias="champion",
    version=model_version.version
)
print(f"Version {model_version.version} promoted to @champion alias")

# --- Step 5: Load the production model for inference ---
champion_model = mlflow.pyfunc.load_model(f"models:/{MODEL_NAME}@champion")
import numpy as np
sample = np.array([[5.1, 3.5, 1.4, 0.2]])
prediction = champion_model.predict(sample)
print(f"Prediction for sample: {prediction}")

# --- Step 6: List all registered versions ---
versions = client.search_model_versions(f"name='{MODEL_NAME}'")
for v in versions:
    print(f"  Version {v.version} | status={v.status} | run_id={v.run_id}")
```

---

## 4. Perks (Advantages) and Drawbacks

### Advantages

| Advantage | Description |
| --------- | ----------- |
| **Open-source and free** | No licensing costs. Full source code access allows customization and self-hosting. |
| **Framework agnostic** | Works with any ML library: scikit-learn, TensorFlow, PyTorch, XGBoost, LightGBM, Spark MLlib, Hugging Face, and more. Also supports Python, R, and Java. |
| **Language agnostic logging** | REST API allows logging from any language or environment. |
| **Reproducibility** | Every run logs the code version, data, parameters, and environment — making experiments fully reproducible. |
| **Autologging** | One-line `mlflow.autolog()` captures all relevant training information automatically for supported frameworks. |
| **Intuitive UI** | Built-in web UI for browsing experiments, comparing runs on charts, and managing model versions — no additional tooling required. |
| **Cloud integration** | Native integrations with AWS SageMaker, Azure ML, and Google Cloud. Models can be deployed to managed endpoints with minimal code. |
| **Collaboration** | Centralized tracking server enables team-wide experiment sharing, comparison, and governance. |
| **GenAI & LLM support** | MLflow 3.x adds tracing, prompt versioning, and agent observability — extending the platform beyond classical ML. |
| **Active community** | Large open-source community, frequent releases, and strong ecosystem integrations. |

### Drawbacks

| Drawback | Description |
| -------- | ----------- |
| **Learning curve** | Understanding all components (tracking, projects, models, registry, recipes) and integrating them into existing workflows takes significant time. |
| **Complex initial setup** | Self-hosting requires configuring the tracking server, backend metadata database (PostgreSQL/MySQL), S3 artifact store, and IAM/access control. |
| **No native pipeline orchestration** | MLflow does not natively support DAG-based workflows or complex multi-branch pipelines. For that, integration with Airflow, Prefect, or Kubeflow Pipelines is needed. |
| **No parallel model execution** | Only linear pipelines are supported natively — parallel experiment branches require external orchestration. |
| **Limited enterprise governance** | Role-based access control, audit logging, and advanced security features require additional setup or a managed version (Databricks or SageMaker). |
| **Overhead for small projects** | For a solo developer with simple experiments, the full MLflow stack may add unnecessary complexity. |
| **Storage management** | Artifact stores (S3, GCS, Azure Blob) need to be managed separately — costs and data lifecycle policies are the user's responsibility. |
| **UI performance** | The MLflow UI can become slow when there are very large numbers of runs or large artifact files. |

---

## 5. MLflow on AWS

AWS provides a **fully managed MLflow** offering through Amazon SageMaker AI, eliminating the need to provision and maintain your own tracking server infrastructure.

### 5.1 Architecture Overview

A managed MLflow deployment on AWS consists of:

| Component | AWS Service |
| --------- | ----------- |
| Tracking Server (compute) | Managed by SageMaker AI service account |
| Backend metadata store | Managed by SageMaker (run IDs, params, metrics, timestamps) |
| Artifact store | **Amazon S3** bucket in your AWS account |
| Authentication | **AWS IAM** + SageMaker RBAC |
| Audit logging | **AWS CloudTrail** (control plane + data plane API calls) |
| Event automation | **Amazon EventBridge** |

### 5.2 MLflow Apps vs. MLflow Tracking Servers

As of late 2025, SageMaker offers two managed MLflow modes:

| Feature | MLflow Apps (new) | MLflow Tracking Servers (legacy) |
| ------- | ----------------- | -------------------------------- |
| MLflow version | 3.x (latest) | 2.x |
| Startup time | Fast (serverless) | Slower (provisioned) |
| Scaling | Automatic serverless | Manual sizing (S/M/L) |
| Cross-account sharing | Yes | Limited |
| Cost when idle | Scales to zero | Incurs fixed cost |

**MLflow Apps** (announced December 2025) are the recommended approach for new deployments.

### 5.3 Tracking Server Sizing (Provisioned Mode)

| Size | Sustained TPS | Burst TPS | Recommended Team Size |
| ---- | ------------- | --------- | --------------------- |
| Small | 25 | 50 | Up to 25 users |
| Medium | 50 | 100 | Up to 50 users |
| Large | 100 | 200 | Up to 100 users |

### 5.4 Setting Up Managed MLflow on SageMaker

```python
import boto3

# Create a managed MLflow Tracking Server
sm_client = boto3.client("sagemaker", region_name="us-east-1")

response = sm_client.create_mlflow_tracking_server(
    TrackingServerName="my-mlflow-server",
    ArtifactStoreUri="s3://my-bucket/mlflow-artifacts",  # Your S3 bucket
    TrackingServerSize="Small",
    MlflowVersion="2.16",
    RoleArn="arn:aws:iam::123456789012:role/SageMakerMLflowRole"
)

tracking_server_arn = response["TrackingServerArn"]
print(f"Tracking Server ARN: {tracking_server_arn}")
```

```python
import mlflow

# Connect your MLflow client to the SageMaker-managed server
mlflow.set_tracking_uri(tracking_server_arn)

# Use MLflow exactly as you would locally
with mlflow.start_run(experiment_id="1"):
    mlflow.log_param("learning_rate", 0.01)
    mlflow.log_metric("loss", 0.42)
```

### 5.5 Deploying MLflow Models to SageMaker Endpoints

Use the `ModelBuilder` class from the SageMaker Python SDK to deploy a registered MLflow model to a real-time inference endpoint.

```python
from sagemaker.serve import ModelBuilder, SchemaBuilder, Mode
import numpy as np

# Define sample input/output for schema inference
sample_input = np.array([[5.1, 3.5, 1.4, 0.2]])
sample_output = np.array([0])

schema = SchemaBuilder(
    sample_input=sample_input,
    sample_output=sample_output
)

# Build deployable model from MLflow Model Registry
model_builder = ModelBuilder(
    mode=Mode.SAGEMAKER_ENDPOINT,
    schema_builder=schema,
    role_arn="arn:aws:iam::123456789012:role/SageMakerExecutionRole",
    model_metadata={
        # Registry path: "models:/<model-name>/<version>"
        "MLFLOW_MODEL_PATH": "models:/iris-random-forest/1",
        # ARN of the MLflow Tracking Server where the model is registered
        "MLFLOW_TRACKING_ARN": "arn:aws:sagemaker:us-east-1:123456789012:mlflow-tracking-server/my-mlflow-server"
    }
)

# Build and deploy to a SageMaker real-time endpoint
model = model_builder.build()
predictor = model.deploy(
    initial_instance_count=1,
    instance_type="ml.c6i.xlarge"
)

# Invoke the endpoint
result = predictor.predict(sample_input)
print(f"Prediction: {result}")

# Clean up
predictor.delete_endpoint()
```

### 5.6 S3 as Artifact Store

MLflow on AWS uses Amazon S3 for all artifact storage (model weights, images, datasets, evaluation reports). The S3 bucket lives in your own AWS account.

```python
import mlflow

# Log artifacts directly to S3
with mlflow.start_run():
    # Log a local file — MLflow uploads it to S3 automatically
    mlflow.log_artifact("model_report.html")

    # Log an entire directory
    mlflow.log_artifacts("./evaluation_outputs/", artifact_path="evaluation")

    # Reference a model stored in S3 (without a tracking server)
    mlflow.sklearn.log_model(
        model,
        artifact_path="model",
        artifact_uri="s3://my-bucket/mlflow-artifacts/models/"
    )
```

### 5.7 Integration with SageMaker Model Registry

MLflow on SageMaker provides **bidirectional synchronization** with SageMaker Model Registry:

- Models registered in MLflow Model Registry are **automatically synchronized** to SageMaker Model Registry.
- From SageMaker Model Registry, models can be deployed to SageMaker inference endpoints with a few clicks.
- This allows teams to use MLflow for experimentation and model management, and SageMaker for governed production deployment.

### 5.8 Additional AWS Integrations

| AWS Service | Integration |
| ----------- | ----------- |
| **Amazon S3** | Artifact store for all MLflow runs (model files, plots, data) |
| **Amazon SageMaker Pipelines** | Orchestrate multi-step ML workflows that log to MLflow |
| **Amazon SageMaker JumpStart** | Fine-tune foundation models and visualize training in MLflow UI |
| **AWS IAM** | Role-based access control for MLflow tracking servers |
| **AWS CloudTrail** | Audit logging for all MLflow API calls (create runs, log metrics, register models, etc.) |
| **Amazon EventBridge** | Event-driven automation triggered by MLflow lifecycle events (model registration, stage transitions) |
| **Amazon SageMaker Studio** | Integrated IDE with MLflow UI embedded; create and manage servers from the Studio interface |

### 5.9 Required IAM Permissions

The SageMaker execution role needs permissions to:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject",
    "s3:DeleteObject",
    "s3:ListBucket",
    "sagemaker:CreateMlflowTrackingServer",
    "sagemaker:DescribeMlflowTrackingServer",
    "sagemaker:UpdateMlflowTrackingServer",
    "sagemaker:DeleteMlflowTrackingServer",
    "sagemaker:CreateArtifact",
    "sagemaker:ListArtifacts",
    "sagemaker:AddAssociation",
    "sagemaker:DescribeMLflowTrackingServer"
  ],
  "Resource": "*"
}
```

---

## 6. Reference Links

### Official MLflow Documentation

- [MLflow Official Website](https://mlflow.org/)
- [MLflow Documentation — Experiment Tracking](https://mlflow.org/docs/latest/ml/tracking/)
- [MLflow Documentation — Model Registry](https://mlflow.org/docs/latest/ml/model-registry/)
- [MLflow Documentation — Model Registry Workflows](https://mlflow.org/docs/latest/ml/model-registry/workflow/)
- [MLflow GitHub Repository](https://github.com/mlflow/mlflow)
- [MLflow Releases](https://mlflow.org/releases)
- [MLflow Python API — mlflow.sagemaker](https://mlflow.org/docs/latest/python_api/mlflow.sagemaker.html)

### AWS Documentation

- [Managed MLflow on Amazon SageMaker AI](https://docs.aws.amazon.com/sagemaker/latest/dg/mlflow.html)
- [Deploy MLflow Models with ModelBuilder](https://docs.aws.amazon.com/sagemaker/latest/dg/mlflow-track-experiments-model-deployment.html)
- [Track Experiments Using MLflow — SageMaker Unified Studio](https://docs.aws.amazon.com/sagemaker-unified-studio/latest/userguide/use-mlflow-experiments.html)
- [Amazon SageMaker AI Experiments Overview](https://aws.amazon.com/sagemaker-ai/experiments/)
- [SageMaker Studio MLflow Integration — GitHub Samples](https://github.com/aws-samples/sagemaker-studio-mlflow-integration)

### AWS Blog Posts

- [Manage ML and GenAI Experiments Using Amazon SageMaker with MLflow (GA announcement)](https://aws.amazon.com/blogs/aws/manage-ml-and-generative-ai-experiments-using-amazon-sagemaker-with-mlflow/)
- [Accelerate AI Development Using Amazon SageMaker AI with Serverless MLflow](https://aws.amazon.com/blogs/aws/accelerate-ai-development-using-amazon-sagemaker-ai-with-serverless-mlflow/)
- [Accelerating GenAI Development with Fully Managed MLflow 3.0 on SageMaker](https://aws.amazon.com/blogs/machine-learning/accelerating-generative-ai-development-with-fully-managed-mlflow-3-0-on-amazon-sagemaker-ai/)
- [Managing Your ML Lifecycle with MLflow and Amazon SageMaker](https://aws.amazon.com/blogs/machine-learning/managing-your-machine-learning-lifecycle-with-mlflow-and-amazon-sagemaker/)

### Community and Tutorials

- [Complete MLflow Guide: Experiment Tracking to Model Registry (2026)](https://www.youngju.dev/blog/ai-platform/2026-03-03-mlflow-experiment-tracking-guide.en)
- [MLflow Model Registry: Workflows, Benefits & Challenges — lakeFS](https://lakefs.io/blog/mlflow-model-registry/)
- [MLflow: A Comprehensive Guide — K21 Academy](https://k21academy.com/mlops/mlflow-the-complete-guide-k21academy/)
- [Getting Started with MLflow for ML Lifecycle](https://oneuptime.com/blog/post/2026-01-26-mlflow-ml-lifecycle/view)
- [MLflow Reviews and Stack — StackShare](https://stackshare.io/mlflow)
