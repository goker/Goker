# Agentic MLOps Platform — System Plan

## Architecting a Multi-Agent MLOps System with A2A and MCP Protocols

> Based on the layered protocol strategy described in [Architecting Agentic MLOps (InfoQ, Feb 2026)](https://www.infoq.com/articles/architecting-agentic-mlops-a2a-mcp/)

---

## Table of Contents

1. [Vision & Problem Statement](#1-vision--problem-statement)
2. [Core Principles](#2-core-principles)
3. [Protocol Foundation](#3-protocol-foundation)
4. [System Architecture](#4-system-architecture)
5. [Agent Catalog](#5-agent-catalog)
6. [MCP Tool Layer](#6-mcp-tool-layer)
7. [A2A Communication Layer](#7-a2a-communication-layer)
8. [Workflows & Orchestration](#8-workflows--orchestration)
9. [Repository Structure](#9-repository-structure)
10. [Technology Stack](#10-technology-stack)
11. [Security & Governance](#11-security--governance)
12. [Deployment Architecture](#12-deployment-architecture)
13. [Implementation Phases](#13-implementation-phases)
14. [Open Questions & Decisions](#14-open-questions--decisions)

---

## 1. Vision & Problem Statement

### The Problem

Traditional MLOps pipelines (Airflow DAGs, Kubeflow Pipelines, etc.) are **rigid, monolithic, and brittle**. Adding a new validation step or swapping a deployment target means rewriting pipeline code. Cross-team collaboration requires deep coupling. Error handling is hardcoded and doesn't adapt.

### The Vision

Build an **agentic MLOps platform** where the ML lifecycle — data preparation, training, validation, deployment, monitoring, and retraining — is orchestrated by a **team of autonomous, specialized AI agents** that communicate via open protocols:

- **A2A (Agent-to-Agent)** for horizontal agent-to-agent collaboration
- **MCP (Model Context Protocol)** for vertical agent-to-tool/data integration

The result: a system where new capabilities are added by deploying new agents (not rewriting pipelines), where agents discover and negotiate with each other, and where the orchestration logic adapts intelligently.

### Key Outcomes

- **Composability**: Add/remove/replace agents without touching orchestration code
- **Interoperability**: Agents built with different frameworks (CrewAI, LangGraph, custom) work together via A2A
- **Governed Tool Access**: All tool/data access is mediated through MCP servers with auth and audit
- **Adaptive Orchestration**: The orchestrator agent reasons about which agents to involve, not just follows a static DAG

---

## 2. Core Principles

| Principle | Description |
|-----------|-------------|
| **Decouple orchestration from execution** | The orchestrator decides *what* to do; specialized agents decide *how* |
| **Protocol over implementation** | Agents interact via A2A/MCP standards, not framework-specific APIs |
| **Agent as a service** | Each agent is an independent deployable unit with its own Agent Card |
| **MCP for all external access** | No agent directly calls databases, APIs, or file systems — always through MCP |
| **Fail gracefully, retry intelligently** | Agents report failure states; orchestrator adapts the plan |
| **Observe everything** | Every agent interaction, tool call, and decision is logged and traceable |

---

## 3. Protocol Foundation

### 3.1 MCP (Model Context Protocol) — Vertical Integration

MCP connects each agent to the tools and data it needs. Think of it as the "USB-C port" for agent capabilities.

```
┌──────────────┐
│    Agent      │
│  (LLM-based)  │
└──────┬───────┘
       │ MCP (JSON-RPC / stdio / SSE)
       │
┌──────▼───────┐
│  MCP Server   │ ← Exposes Tools, Resources, Prompts
│  (FastMCP)    │
└──────┬───────┘
       │
┌──────▼───────┐
│  External     │ ← Databases, APIs, S3, MLflow, K8s, etc.
│  Systems      │
└──────────────┘
```

**MCP Primitives Used:**
- **Tools**: Execute actions (train model, deploy, run validation, query metrics)
- **Resources**: Read-only data (model registry entries, dataset metadata, config files)
- **Prompts**: Reusable interaction templates (validation report format, deployment checklist)

### 3.2 A2A (Agent-to-Agent Protocol) — Horizontal Integration

A2A enables agents to discover, communicate, and delegate work to each other.

```
┌─────────────┐    A2A (JSON-RPC 2.0 / HTTPS)    ┌─────────────┐
│  Agent A     │◄────────────────────────────────►│  Agent B     │
│  (Client)    │   tasks/send, tasks/sendSubscribe │  (Server)    │
└─────────────┘                                    └─────────────┘
       │                                                  │
       │ Discovers via                                    │ Publishes
       ▼                                                  ▼
  /.well-known/agent.json                        /.well-known/agent.json
```

**A2A Concepts Used:**
- **Agent Card**: JSON metadata at `/.well-known/agent.json` — name, skills, endpoint, auth
- **Tasks**: Unit of work with lifecycle (submitted → working → completed/failed)
- **Messages**: Communication turns with role (user/agent) containing Parts
- **Artifacts**: Output produced by a task (model files, reports, metrics JSON)
- **Streaming**: SSE for long-running tasks (training progress, deployment status)

### 3.3 How They Layer Together

```
┌──────────────────────────────────────────────────────┐
│                  ORCHESTRATION LAYER                  │
│         Orchestrator Agent (decides what/who)         │
└───────────┬──────────────────────────┬───────────────┘
            │ A2A                      │ A2A
            ▼                          ▼
┌───────────────────┐      ┌───────────────────┐
│ Validation Agent   │      │ Deployment Agent   │
│ (A2A Server)       │      │ (A2A Server)       │
└─────────┬─────────┘      └─────────┬─────────┘
          │ MCP                      │ MCP
          ▼                          ▼
┌───────────────────┐      ┌───────────────────┐
│ MCP: MLflow        │      │ MCP: Kubernetes    │
│ MCP: Great Expect. │      │ MCP: Model Registry│
│ MCP: Data Quality  │      │ MCP: Feature Store │
└───────────────────┘      └───────────────────┘
```

---

## 4. System Architecture

### High-Level Architecture

```
                         ┌──────────────────────┐
                         │   Human / CI Trigger  │
                         └──────────┬───────────┘
                                    │ A2A task/send
                                    ▼
                         ┌──────────────────────┐
                         │  ORCHESTRATOR AGENT   │
                         │  ─────────────────    │
                         │  • Receives goals     │
                         │  • Discovers agents   │
                         │  • Plans execution    │
                         │  • Coordinates flow   │
                         │  • Handles failures   │
                         └──┬───┬───┬───┬───┬──┘
                            │   │   │   │   │
                  A2A       │   │   │   │   │       A2A
         ┌──────────────────┘   │   │   │   └──────────────────┐
         ▼                      ▼   │   ▼                      ▼
┌─────────────────┐  ┌──────────────┤  ┌─────────────────┐  ┌──────────────┐
│  DATA AGENT     │  │ TRAINING     │  │ VALIDATION      │  │ DEPLOYMENT   │
│  ───────────    │  │ AGENT        │  │ AGENT           │  │ AGENT        │
│  • Ingest       │  │ ──────       │  │ ──────────      │  │ ──────────   │
│  • Transform    │  │ • Train      │  │ • Model quality │  │ • Canary     │
│  • Validate     │  │ • Tune       │  │ • Data drift    │  │ • Blue/Green │
│  • Version      │  │ • Track      │  │ • Bias/fairness │  │ • Rollback   │
└────────┬────────┘  └──────┬───────┘  └────────┬────────┘  └──────┬───────┘
         │ MCP              │ MCP               │ MCP              │ MCP
         ▼                  ▼                   ▼                  ▼
┌─────────────────┐  ┌─────────────┐  ┌─────────────────┐  ┌──────────────┐
│ MCP Servers:    │  │ MCP Servers:│  │ MCP Servers:    │  │ MCP Servers: │
│ • S3/GCS        │  │ • MLflow    │  │ • Great Expect. │  │ • Kubernetes │
│ • Snowflake     │  │ • Ray       │  │ • Evidently     │  │ • Docker Reg │
│ • Delta Lake    │  │ • W&B       │  │ • Deepchecks    │  │ • Istio      │
│ • DVC           │  │ • SageMaker │  │ • Custom metrics│  │ • ArgoCD     │
└─────────────────┘  └─────────────┘  └─────────────────┘  └──────────────┘

                  ┌──────────────────────────────────┐
                  │        MONITORING AGENT           │
                  │  • Model performance tracking     │
                  │  • Data drift detection           │
                  │  • Alert → trigger retraining     │
                  └──────────────┬───────────────────┘
                                │ MCP
                                ▼
                  ┌──────────────────────────────────┐
                  │ MCP Servers: Prometheus, Grafana, │
                  │ CloudWatch, PagerDuty             │
                  └──────────────────────────────────┘
```

---

## 5. Agent Catalog

Each agent is an independent A2A server that publishes an Agent Card and exposes skills.

### 5.1 Orchestrator Agent

**Role**: Central coordinator. Receives high-level goals ("deploy model v2.3 to production"), discovers available agents, plans the execution sequence, and delegates work.

**Agent Card Skills**:
- `orchestrate-ml-pipeline` — End-to-end ML pipeline coordination
- `plan-deployment` — Plan and coordinate a model deployment
- `handle-retraining` — Coordinate retraining triggered by monitoring alerts

**Key Behaviors**:
- Fetches Agent Cards from a registry to discover available agents
- Reasons about which agents to involve based on the goal
- Manages task dependencies (validation must pass before deployment)
- Handles failures: retries, fallbacks, or escalation to human
- Maintains conversation context across multi-step workflows via A2A `contextId`

**Does NOT**: Directly access any external systems. All actions go through delegated agents.

### 5.2 Data Agent

**Role**: Manages the data lifecycle — ingestion, transformation, validation, versioning.

**Agent Card Skills**:
- `ingest-data` — Pull data from sources, apply transformations
- `validate-data` — Run data quality checks (schema, completeness, freshness)
- `version-dataset` — Create versioned snapshots of datasets

**MCP Tools Used**:
- `mcp-s3` / `mcp-gcs` — Read/write data to object storage
- `mcp-snowflake` / `mcp-bigquery` — Query data warehouses
- `mcp-dvc` — Data version control operations
- `mcp-great-expectations` — Data validation suite execution

### 5.3 Training Agent

**Role**: Executes and manages model training runs.

**Agent Card Skills**:
- `train-model` — Execute a training run with given config
- `hyperparameter-tune` — Run hyperparameter search
- `track-experiment` — Log experiment to tracking server

**MCP Tools Used**:
- `mcp-mlflow` — Experiment tracking, model logging, registry
- `mcp-ray` — Distributed training job submission
- `mcp-wandb` — Experiment tracking (alternative)
- `mcp-sagemaker` — Managed training jobs on AWS

**Streaming**: Reports training progress (loss, metrics) via A2A SSE streaming back to orchestrator.

### 5.4 Validation Agent

**Role**: Validates models before they proceed to deployment. Gatekeeping function.

**Agent Card Skills**:
- `validate-model-quality` — Run accuracy, precision, recall checks against thresholds
- `check-data-drift` — Compare training vs. serving data distributions
- `check-model-fairness` — Run bias and fairness checks
- `generate-validation-report` — Produce a structured validation report artifact

**MCP Tools Used**:
- `mcp-evidently` — Data drift and model performance monitoring
- `mcp-deepchecks` — Model validation suite
- `mcp-great-expectations` — Data quality validation
- `mcp-mlflow` — Retrieve model artifacts and metrics for comparison

**Output**: Returns an A2A Artifact containing the validation report (pass/fail + details).

### 5.5 Deployment Agent

**Role**: Deploys validated models to target environments.

**Agent Card Skills**:
- `deploy-canary` — Deploy model as canary with traffic splitting
- `deploy-blue-green` — Blue/green deployment swap
- `rollback-deployment` — Roll back to previous model version
- `promote-to-production` — Promote staging model to production

**MCP Tools Used**:
- `mcp-kubernetes` — K8s deployment, service, and ingress management
- `mcp-docker-registry` — Container image operations
- `mcp-istio` — Traffic routing and splitting
- `mcp-argocd` — GitOps deployment triggers

### 5.6 Monitoring Agent

**Role**: Continuously monitors deployed models and triggers retraining when needed.

**Agent Card Skills**:
- `monitor-model-performance` — Track prediction quality metrics
- `detect-drift` — Monitor for data/concept drift
- `trigger-retraining-alert` — Notify orchestrator that retraining is needed

**MCP Tools Used**:
- `mcp-prometheus` — Query metrics
- `mcp-grafana` — Dashboard management
- `mcp-cloudwatch` — AWS metrics
- `mcp-pagerduty` — Alert management

**Autonomous Behavior**: This agent runs on a schedule. When drift is detected beyond a threshold, it sends an A2A task to the Orchestrator to initiate retraining.

### 5.7 Cost Agent (Optional)

**Role**: Tracks and optimizes infrastructure costs for ML workloads.

**Agent Card Skills**:
- `estimate-training-cost` — Estimate cost of a training run
- `recommend-instance-type` — Suggest optimal compute for a workload
- `report-spend` — Generate cost reports

---

## 6. MCP Tool Layer

### 6.1 MCP Server Catalog

Each MCP server wraps one external system and exposes it as Tools/Resources.

| MCP Server | Wraps | Key Tools | Key Resources |
|------------|-------|-----------|---------------|
| `mcp-mlflow` | MLflow | `create_experiment`, `log_metrics`, `register_model`, `transition_model_stage` | `model://registry/{name}`, `experiment://runs/{id}` |
| `mcp-kubernetes` | K8s API | `apply_manifest`, `get_pod_status`, `scale_deployment`, `rollout_restart` | `cluster://deployments/{ns}/{name}` |
| `mcp-s3` | AWS S3 | `upload_object`, `download_object`, `list_objects` | `s3://{bucket}/{key}` |
| `mcp-prometheus` | Prometheus | `query_instant`, `query_range`, `get_alerts` | `metrics://{metric_name}` |
| `mcp-great-expectations` | Great Expectations | `run_checkpoint`, `get_validation_result` | `expectations://{suite_name}` |
| `mcp-evidently` | Evidently AI | `run_drift_report`, `run_model_performance_report` | `reports://{report_id}` |
| `mcp-docker-registry` | Docker Registry | `push_image`, `pull_image`, `list_tags` | `image://{repo}/{tag}` |
| `mcp-ray` | Ray | `submit_job`, `get_job_status`, `get_job_logs` | `jobs://{job_id}` |
| `mcp-feature-store` | Feast/Tecton | `get_features`, `materialize_features` | `features://{feature_view}` |
| `mcp-argocd` | ArgoCD | `sync_application`, `get_app_status` | `apps://{app_name}` |

### 6.2 MCP Server Design Pattern

Each MCP server follows a standard pattern:

```
mcp-servers/
  mcp-mlflow/
    server.py          ← FastMCP server definition
    tools.py           ← Tool implementations (@mcp.tool())
    resources.py       ← Resource implementations (@mcp.resource())
    config.py          ← Connection settings
    Dockerfile         ← Container packaging
    agent.json         ← Not needed (MCP servers aren't A2A agents)
```

### 6.3 Authentication & Access Control

- Each MCP server implements OAuth 2.1 token validation (via `mcp.server.auth`)
- Agents receive scoped tokens — a Training Agent cannot call deployment tools
- All MCP tool invocations are audit-logged

---

## 7. A2A Communication Layer

### 7.1 Agent Card Format

Each agent publishes at `/.well-known/agent.json`:

```json
{
  "name": "validation-agent",
  "version": "1.0.0",
  "description": "Validates ML models against quality, drift, and fairness criteria",
  "url": "https://validation-agent.internal:8080",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "validate-model-quality",
      "name": "Model Quality Validation",
      "description": "Run accuracy, precision, recall checks against configurable thresholds",
      "tags": ["mlops", "validation", "quality"]
    },
    {
      "id": "check-data-drift",
      "name": "Data Drift Detection",
      "description": "Compare training vs serving data distributions using statistical tests",
      "tags": ["mlops", "drift", "monitoring"]
    },
    {
      "id": "check-model-fairness",
      "name": "Model Fairness Check",
      "description": "Run bias and fairness evaluations across protected attributes",
      "tags": ["mlops", "fairness", "bias"]
    }
  ],
  "authentication": {
    "schemes": ["bearer"]
  },
  "defaultInputModes": ["application/json"],
  "defaultOutputModes": ["application/json"]
}
```

### 7.2 Task Lifecycle

```
  ┌───────────┐
  │ submitted  │ ← Orchestrator sends task/send
  └─────┬─────┘
        ▼
  ┌───────────┐
  │  working   │ ← Agent processes (streams progress via SSE)
  └─────┬─────┘
        │
   ┌────┴────┐
   ▼         ▼
┌──────┐  ┌──────┐
│ done │  │failed│ ← Agent returns result or error
└──────┘  └──────┘
```

### 7.3 Agent Registry / Discovery

Rather than hardcoding agent URLs, the system uses a lightweight agent registry:

- **Option A**: Static registry file (JSON/YAML) listing all agent endpoints — simple, GitOps-friendly
- **Option B**: DNS-based discovery (`validation-agent.agents.internal`)
- **Option C**: Dynamic registry service that agents register with on startup

Recommended for Phase 1: **Option A** (static config) with migration path to **Option C**.

---

## 8. Workflows & Orchestration

### 8.1 Workflow: End-to-End Model Deployment

**Trigger**: Human request or CI event — "Deploy model `fraud-detector` version `2.3` to production"

**Sequence**:

```
1. Orchestrator receives goal
2. Orchestrator fetches Agent Cards (discovers available agents)
3. Orchestrator plans: Data validation → Model validation → Canary deploy → Monitor

4. Orchestrator → Data Agent (A2A: task/send)
   "Validate the training dataset for fraud-detector v2.3"
   ├── Data Agent → mcp-great-expectations (MCP: run_checkpoint)
   ├── Data Agent → mcp-s3 (MCP: get dataset metadata)
   └── Data Agent returns: ✅ Data validation passed (A2A Artifact: report)

5. Orchestrator → Validation Agent (A2A: task/sendSubscribe for streaming)
   "Validate model fraud-detector v2.3 against production thresholds"
   ├── Validation Agent → mcp-mlflow (MCP: get model metrics)
   ├── Validation Agent → mcp-evidently (MCP: run drift report)
   ├── Validation Agent → mcp-deepchecks (MCP: run validation suite)
   ├── Streams progress: "Running accuracy checks..." "Running drift checks..."
   └── Validation Agent returns: ✅ All checks passed (A2A Artifact: report)

6. Orchestrator evaluates: Both passed → proceed to deployment

7. Orchestrator → Deployment Agent (A2A: task/sendSubscribe)
   "Deploy fraud-detector v2.3 as canary with 10% traffic"
   ├── Deployment Agent → mcp-docker-registry (MCP: verify image exists)
   ├── Deployment Agent → mcp-kubernetes (MCP: apply canary manifest)
   ├── Deployment Agent → mcp-istio (MCP: configure traffic split)
   ├── Streams: "Image verified" "Canary pod running" "Traffic split active"
   └── Deployment Agent returns: ✅ Canary deployed (A2A Artifact: deployment info)

8. Orchestrator → Monitoring Agent (A2A: task/send)
   "Watch fraud-detector canary for 30 minutes, alert if error rate > 1%"
   ├── Monitoring Agent → mcp-prometheus (MCP: query_range)
   └── Monitoring Agent returns: ✅ Canary healthy

9. Orchestrator → Deployment Agent (A2A: task/send)
   "Promote canary to 100% traffic"
   └── Deployment Agent promotes

10. Orchestrator returns final status + summary to caller
```

### 8.2 Workflow: Automated Retraining

**Trigger**: Monitoring Agent detects drift

```
1. Monitoring Agent → mcp-prometheus (MCP: query metrics)
   Detects: accuracy dropped below threshold

2. Monitoring Agent → Orchestrator (A2A: task/send)
   "Model fraud-detector showing accuracy degradation. Retraining recommended."
   Includes: drift report artifact

3. Orchestrator plans: Fresh data → Retrain → Validate → Deploy

4. Orchestrator → Data Agent: "Prepare latest 30-day training dataset"
5. Orchestrator → Training Agent: "Retrain fraud-detector with new dataset"
   ├── Training Agent → mcp-ray (MCP: submit distributed training job)
   ├── Training Agent → mcp-mlflow (MCP: log experiment, register model)
   ├── Streams training progress (loss curves, metrics)
   └── Returns: Model v2.4 registered

6. Orchestrator → Validation Agent: "Validate v2.4"
7. If passed → Orchestrator → Deployment Agent: "Canary deploy v2.4"
8. If failed → Orchestrator notifies human + logs failure reason
```

### 8.3 Failure Handling

| Failure Scenario | Orchestrator Response |
|-----------------|----------------------|
| Validation fails | Halt pipeline, log reason, notify human |
| Canary shows errors | Instruct Deployment Agent to rollback |
| Training job OOMs | Retry with Training Agent using larger instance recommendation from Cost Agent |
| Agent unreachable | Retry with backoff, then fail gracefully with human notification |
| MCP tool timeout | Agent retries internally, reports failure to orchestrator if persistent |

---

## 9. Repository Structure

Proposed repository layout:

```
agentic-mlops/
├── README.md
├── PLAN.md                          ← This document
├── LICENSE
├── pyproject.toml                   ← Root project config (monorepo)
├── docker-compose.yml               ← Local dev: spin up all agents + MCP servers
│
├── proto/                           ← Shared protocol definitions
│   ├── agent_card_schema.json       ← JSON Schema for Agent Cards
│   └── common_types.py              ← Shared Pydantic models (Task, Message, Artifact)
│
├── agents/                          ← A2A Agent implementations
│   ├── orchestrator/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   ├── agent.json               ← Agent Card
│   │   ├── src/
│   │   │   ├── __init__.py
│   │   │   ├── server.py            ← A2A server (JSON-RPC endpoint)
│   │   │   ├── planner.py           ← Goal → plan decomposition logic
│   │   │   ├── registry.py          ← Agent discovery (reads agent cards)
│   │   │   └── workflows.py         ← Predefined workflow templates
│   │   └── tests/
│   │
│   ├── data_agent/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   ├── agent.json
│   │   ├── src/
│   │   │   ├── server.py            ← A2A server
│   │   │   ├── skills.py            ← Skill implementations
│   │   │   └── mcp_clients.py       ← MCP client connections
│   │   └── tests/
│   │
│   ├── training_agent/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   ├── agent.json
│   │   ├── src/
│   │   │   ├── server.py
│   │   │   ├── skills.py
│   │   │   └── mcp_clients.py
│   │   └── tests/
│   │
│   ├── validation_agent/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   ├── agent.json
│   │   ├── src/
│   │   │   ├── server.py
│   │   │   ├── skills.py
│   │   │   └── mcp_clients.py
│   │   └── tests/
│   │
│   ├── deployment_agent/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   ├── agent.json
│   │   ├── src/
│   │   │   ├── server.py
│   │   │   ├── skills.py
│   │   │   └── mcp_clients.py
│   │   └── tests/
│   │
│   └── monitoring_agent/
│       ├── pyproject.toml
│       ├── Dockerfile
│       ├── agent.json
│       ├── src/
│       │   ├── server.py
│       │   ├── skills.py
│       │   └── mcp_clients.py
│       └── tests/
│
├── mcp_servers/                     ← MCP Server implementations
│   ├── mcp_mlflow/
│   │   ├── pyproject.toml
│   │   ├── Dockerfile
│   │   ├── src/
│   │   │   ├── server.py            ← FastMCP server
│   │   │   ├── tools.py             ← @mcp.tool() definitions
│   │   │   └── resources.py         ← @mcp.resource() definitions
│   │   └── tests/
│   │
│   ├── mcp_kubernetes/
│   │   └── ...
│   ├── mcp_s3/
│   │   └── ...
│   ├── mcp_prometheus/
│   │   └── ...
│   ├── mcp_great_expectations/
│   │   └── ...
│   ├── mcp_evidently/
│   │   └── ...
│   ├── mcp_docker_registry/
│   │   └── ...
│   └── mcp_ray/
│       └── ...
│
├── registry/                        ← Agent registry
│   ├── agents.yaml                  ← Static registry of agent endpoints
│   └── discovery.py                 ← Registry service (Phase 2)
│
├── config/                          ← Environment configs
│   ├── dev.yaml
│   ├── staging.yaml
│   └── production.yaml
│
├── deploy/                          ← Deployment manifests
│   ├── kubernetes/
│   │   ├── namespace.yaml
│   │   ├── agents/                  ← K8s manifests per agent
│   │   └── mcp-servers/             ← K8s manifests per MCP server
│   └── helm/
│       └── agentic-mlops/           ← Helm chart
│
├── docs/                            ← Documentation
│   ├── architecture.md
│   ├── adding-a-new-agent.md
│   ├── adding-a-new-mcp-server.md
│   └── workflows.md
│
└── tests/
    ├── integration/                 ← End-to-end workflow tests
    └── contracts/                   ← A2A/MCP contract tests
```

---

## 10. Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Language** | Python 3.12+ | ML ecosystem, MCP SDK, A2A SDK support |
| **A2A Framework** | `a2a-python` SDK | Official A2A Python SDK from Google/Linux Foundation |
| **MCP Framework** | `mcp` (FastMCP) | Official MCP Python SDK from Anthropic/Linux Foundation |
| **LLM (Agent brains)** | Claude / GPT-4 / Llama | Orchestrator + agents use LLMs for reasoning; pluggable |
| **Agent Framework** | LangGraph or custom | For internal agent logic (tool selection, reasoning chains) |
| **HTTP Server** | FastAPI / Starlette | A2A JSON-RPC endpoint hosting |
| **Experiment Tracking** | MLflow | Open-source, widely adopted |
| **Distributed Training** | Ray | Scalable distributed compute |
| **Data Validation** | Great Expectations | Comprehensive data quality |
| **Model Monitoring** | Evidently AI | Drift detection, model performance |
| **Container Runtime** | Docker | Packaging agents and MCP servers |
| **Orchestration** | Kubernetes | Production deployment |
| **Service Mesh** | Istio | Traffic management for canary deploys |
| **GitOps** | ArgoCD | Declarative deployments |
| **Observability** | Prometheus + Grafana | Metrics and dashboards |
| **CI/CD** | GitHub Actions | Pipeline automation |
| **Package Management** | `uv` | Fast Python dependency resolution |

---

## 11. Security & Governance

### 11.1 Authentication & Authorization

- **Agent-to-Agent**: A2A Agent Cards declare supported auth schemes (OAuth 2.0 Bearer tokens)
- **Agent-to-Tool**: MCP servers validate scoped OAuth tokens; agents receive least-privilege tokens
- **Human-to-System**: OAuth 2.0 / OIDC for dashboard and API access
- **Service Mesh mTLS**: All inter-agent traffic encrypted via Istio mTLS

### 11.2 Access Control Matrix

| Agent | MCP Servers Accessible | Rationale |
|-------|----------------------|-----------|
| Orchestrator | None (delegates everything) | Pure coordination, no direct tool access |
| Data Agent | S3, Snowflake, DVC, Great Expectations | Data operations only |
| Training Agent | MLflow, Ray, W&B, SageMaker | Training operations only |
| Validation Agent | MLflow, Evidently, Deepchecks, Great Expectations | Read-only model + data access |
| Deployment Agent | Kubernetes, Docker Registry, Istio, ArgoCD | Deployment operations only |
| Monitoring Agent | Prometheus, Grafana, CloudWatch, PagerDuty | Monitoring + alerting only |

### 11.3 Audit & Compliance

- Every A2A `task/send` logged with: timestamp, caller, callee, task ID, context ID
- Every MCP tool invocation logged with: agent ID, tool name, parameters, result status
- Immutable audit log stored separately from operational logs
- Model lineage: full trace from data version → training run → validation → deployment

---

## 12. Deployment Architecture

### 12.1 Local Development

```
docker-compose up
```

Spins up:
- All 6 agents as containers
- All MCP servers as containers
- MLflow server
- Prometheus + Grafana
- Mock Kubernetes (kind)

### 12.2 Production (Kubernetes)

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ orchestrator │  │ data-agent  │  │ train-agent │         │
│  │ (Deployment) │  │ (Deployment)│  │ (Deployment)│         │
│  │ replicas: 2  │  │ replicas: 2 │  │ replicas: 1 │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                  │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐         │
│  │ valid-agent │  │ deploy-agent│  │ monitor-agt │         │
│  │ (Deployment)│  │ (Deployment)│  │ (CronJob +  │         │
│  │ replicas: 2 │  │ replicas: 1 │  │  Deployment)│         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
│  ┌──────────────────────────────────────────────┐           │
│  │          MCP Servers (Deployments)            │           │
│  │  mlflow | k8s | s3 | prometheus | evidently  │           │
│  └──────────────────────────────────────────────┘           │
│                                                             │
│  ┌──────────────────────────────────────────────┐           │
│  │          Istio Service Mesh (mTLS)            │           │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

---

## 13. Implementation Phases

### Phase 1: Foundation (MVP)

**Goal**: Prove the architecture works end-to-end with 3 agents and 2 MCP servers.

- Implement Orchestrator Agent (basic, with static workflow)
- Implement Validation Agent (model quality checks only)
- Implement Deployment Agent (simple K8s deployment)
- Implement `mcp-mlflow` MCP server
- Implement `mcp-kubernetes` MCP server
- A2A communication between orchestrator ↔ validation ↔ deployment
- Static agent registry (YAML file)
- Docker Compose local dev environment
- Basic integration test: "validate and deploy a model"

### Phase 2: Core Agents

**Goal**: Add remaining agents and MCP servers for a complete ML lifecycle.

- Implement Data Agent + `mcp-s3`, `mcp-great-expectations`
- Implement Training Agent + `mcp-ray`
- Implement Monitoring Agent + `mcp-prometheus`
- Add SSE streaming for long-running tasks (training progress)
- Implement full retraining workflow
- Add authentication (OAuth 2.0 bearer tokens)
- Kubernetes deployment manifests

### Phase 3: Intelligence & Resilience

**Goal**: Make the orchestrator smarter and the system more resilient.

- Orchestrator uses LLM reasoning for dynamic planning (not just static workflows)
- Agent health checks and circuit breakers
- Retry logic with exponential backoff
- Cost Agent for infrastructure optimization
- Comprehensive observability (distributed tracing, dashboards)
- Helm chart for production deployment

### Phase 4: Scale & Polish

**Goal**: Production-readiness.

- Dynamic agent registry with auto-discovery
- Multi-cluster support
- A/B testing workflow
- Human-in-the-loop approval gates
- Comprehensive documentation and onboarding guides
- Performance benchmarks and load testing
- Security audit

---

## 14. Open Questions & Decisions

| # | Question | Options | Recommendation |
|---|----------|---------|----------------|
| 1 | **LLM for agent reasoning** | Claude API / OpenAI / Local (Llama) / Pluggable | Pluggable with Claude as default |
| 2 | **Agent framework** | LangGraph / CrewAI / AutoGen / Custom | LangGraph (good graph-based orchestration, flexible) or Custom (less abstraction, more control) |
| 3 | **A2A transport** | HTTP + SSE / gRPC (A2A v0.3+) | HTTP + SSE for Phase 1, gRPC option for Phase 3 |
| 4 | **Agent registry** | Static YAML / DNS / Dynamic service | Static YAML for Phase 1, dynamic service for Phase 3 |
| 5 | **Monorepo vs polyrepo** | Single repo / Separate repos per agent | Monorepo for Phase 1-2, evaluate split at Phase 4 |
| 6 | **Cloud provider** | AWS / GCP / Azure / Cloud-agnostic | Cloud-agnostic with AWS as reference implementation |
| 7 | **Model serving** | Seldon / KServe / BentoML / Custom | KServe (K8s native, good ecosystem) |
| 8 | **Feature store** | Feast / Tecton / Custom | Feast (open-source, fits the stack) |
| 9 | **Orchestrator intelligence** | Static workflows only / LLM-powered planning | Static workflows Phase 1, LLM planning Phase 3 |

---

## References

- [Architecting Agentic MLOps: A Layered Protocol Strategy with A2A and MCP (InfoQ)](https://www.infoq.com/articles/architecting-agentic-mlops-a2a-mcp/)
- [A2A Protocol Specification](https://a2a-protocol.org/latest/specification/)
- [A2A GitHub Repository](https://github.com/a2aproject/A2A)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MCP Specification](https://modelcontextprotocol.github.io/python-sdk/)
- [Unlocking Multi-Agent AI with MCP and A2A (Ardor)](https://ardor.cloud/blog/mcp-vs-a2a-unlocking-multi-agent-ai)
- [Orchestrating Multi-Agent Workflows with MCP & A2A (Iguazio)](https://www.iguazio.com/blog/orchestrating-multi-agent-workflows-with-mcp-a2a/)
- [The Autonomous MLOps Engineer (ActiveWizards)](https://activewizards.com/blog/the-autonomous-mlops-engineer-automating-the-ml-lifecycle)
