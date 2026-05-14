# SaleScore ML Platform — Constitution

## Core Principles

### I. Platform-First, Model-Second
This is a reusable ML platform, not a collection of pipelines.
All infrastructure, tooling, and orchestration is shared across models.
A new model requires only a config file and a training script — not new infrastructure.

### II. Model Configurability via YAML (NON-NEGOTIABLE)
Every model is described by a single `model_config.yaml`:
model name, feature list, target column, algorithm family, hyperparameters, schedule, input sources.
No business logic is hardcoded — it lives in config.
Adding a new model to the portfolio = adding a config file + training script.

### III. Three-Environment Promotion Flow
**prototype** → experimentation, notebooks, MLflow tracking, no production data
**dev** → scripts, integration tests, CI, staging data, MLflow model registration
**prod** → scheduled batch scoring, registered+approved models only, real client data

Promotion between environments requires explicit approval (MLflow model stage transition: Staging → Production).
No model runs in prod unless it passed dev validation and was promoted via MLflow.

### IV. MLflow as Single Source of Truth for Models
All experiments tracked in MLflow (prototype env).
All models registered in MLflow Model Registry.
Batch scoring jobs load models by name + stage (`Production`) — never by file path.
Model versioning, lineage, and metrics live in MLflow, not in filenames or S3 keys.

### V. Separation of Concerns
Feature engineering (Glue PySpark) is independent from model training (AWS Batch / SageMaker).
Training is independent from inference/scoring.
Each stage has its own Step Functions state machine and can be triggered independently.

### VI. IaC — Everything in CDK Python (NON-NEGOTIABLE)
All AWS resources are defined in AWS CDK (Python).
CDK stacks are parameterized by environment (`prototype` / `dev` / `prod`).
No manual console changes. `cdk diff` required before every `cdk deploy`.
Stacks: `StorageStack`, `GlueStack`, `TrainingStack`, `MLflowStack`, `ScoringStack`, `OrchestrationStack`.

### VII. Minimal Client Configuration
A new client deploys the platform by providing one config file:
S3 source paths, model selection, run schedule, environment tag.
All infrastructure spins up from `cdk deploy` + that config — zero manual steps.

### VIII. Observability
Every pipeline stage logs structured JSON: stage, model_name, model_version, run_date,
input_rows, output_rows, duration_seconds, status, error (if any).
No silent failures. Failed stages halt the state machine and emit an alert.

## AWS Architecture

- **Storage**: S3 (raw → features → predictions, per environment prefix)
- **Feature Engineering**: AWS Glue PySpark Jobs (one job per model or shared by model family)
- **Training**: AWS Batch (containerized PySpark/sklearn/xgboost training jobs)
- **Model Registry**: Amazon SageMaker Managed MLflow (fully managed tracking server, S3 artifact store)
- **Scoring**: AWS Batch (batch scoring on configurable schedule, loads model from MLflow by name+stage)
- **Orchestration**: AWS Step Functions (separate state machines: feature_eng, training, scoring)
- **Scheduling**: Amazon EventBridge Scheduler (one rule per model, schedule defined in model_config.yaml — weekly/monthly/quarterly)
- **IaC**: AWS CDK v2 (Python)
- **Containers**: Amazon ECR (one image per model family: classification, regression, clustering)

## Model Portfolio (v1 scope — SMB Sales)

All models share the same platform infrastructure, differ only in config + training script.
- Churn prediction (classification) ← first to implement
- Customer Lifetime Value (regression)
- Lead scoring (classification)
- Sales forecasting (regression)
- Payment default risk (classification)

## Development Workflow

prototype: notebook experiments → MLflow tracking → register model as "Staging"
dev: scripted training → integration tests → promote model to "Production" in MLflow
prod: EventBridge triggers monthly scoring → Step Functions → Glue → Batch → S3 output

## Governance

Constitution supersedes all other practices.
All implementation decisions are evaluated against these principles.
Amendments require updating this document + PR review.

**Version**: 2.0.0 | **Ratified**: 2026-05-13 | **Last Amended**: 2026-05-13
