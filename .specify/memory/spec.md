# Platform Specification: SaleScore ML Platform

**Version**: 2.0.0
**Created**: 2026-05-13
**Status**: Draft

---

## What We Are Building

A **self-deployable MLOps platform** for small and medium sales businesses.
A client points it at their data sources, selects models from the portfolio, and gets monthly churn/LTV/lead scores in S3.
The platform handles everything: feature engineering, training, model registry, batch scoring, infrastructure.

**Client onboarding flow**:
1. Clone repo
2. Fill in `client_config.yaml` (S3 paths, model selection, schedule)
3. `cdk deploy --all --context env=prod`
4. Wait for first monthly scores

---

## User Scenarios & Testing

### User Story 1 — Monthly Batch Scoring (Priority: P1)

On the 2nd of each month, EventBridge triggers the scoring pipeline for each active model.
Glue reads raw client data, joins and engineers features, writes to S3.
Batch job loads the `Production`-stage model from MLflow, scores all customers, writes predictions to S3.

**Acceptance Scenarios**:

1. **Given** raw Parquet files exist in S3 and a `Production` model is registered in MLflow,
   **When** EventBridge triggers the scoring state machine on the 2nd of the month,
   **Then** predictions Parquet lands at `s3://{output_bucket}/predictions/{model_name}/run_date={YYYY-MM}/` within 30 minutes.

2. **Given** the pipeline is re-run for the same month,
   **Then** output is overwritten with identical content (idempotent).

3. **Given** no `Production` model exists in MLflow for a given model,
   **Then** the state machine fails with a clear error — no silent scoring with a wrong model.

---

### User Story 2 — Model Training & Registration (Priority: P1)

A data scientist runs the training pipeline in `dev`.
The training job reads features from S3, trains the model, logs metrics to MLflow, and registers the model as `Staging`.
After review, the model is promoted to `Production` — either manually via MLflow UI or via a promotion script.

**Acceptance Scenarios**:

1. **Given** feature data exists in S3 (dev prefix),
   **When** training job runs,
   **Then** model is registered in MLflow with metrics (accuracy, AUC, F1), feature importance, and artifact path.

2. **Given** a `Staging` model in MLflow,
   **When** `promote_model.py --model churn --version 3` is run,
   **Then** model transitions to `Production` stage and previous `Production` is archived.

---

### User Story 3 — Feature Engineering (Priority: P1)

Glue PySpark job reads raw tables, joins them, engineers features, writes feature Parquet.
Feature job is model-specific (config-driven) but runs on shared Glue infrastructure.

**Acceptance Scenarios**:

1. **Given** `client_aggregates` and `client_transactions` in S3,
   **When** Glue feature job for churn model runs,
   **Then** feature Parquet is written to `s3://{features_bucket}/features/churn/run_date={YYYY-MM}/` with the expected schema.

2. **Given** a customer exists in `client_aggregates` but has no rows in `client_transactions`,
   **Then** customer is included with transaction features as 0 (left join semantics).

---

### User Story 4 — New Model Onboarding (Priority: P2)

Adding a new model to the portfolio requires:
- A `model_config.yaml` file
- A training script
- A Glue feature engineering script
No infrastructure changes required.

**Acceptance Scenarios**:

1. **Given** `configs/lead_scoring/model_config.yaml` and `training/lead_scoring/train.py` exist,
   **When** the training pipeline is triggered for `lead_scoring`,
   **Then** it runs end-to-end using shared infrastructure (Glue, Batch, MLflow, Step Functions).

---

### User Story 5 — Client Deployment (Priority: P2)

A new client deploys the platform to their AWS account with minimal config.

**Acceptance Scenarios**:

1. **Given** a new AWS account with CDK bootstrapped,
   **When** client fills `client_config.yaml` and runs `cdk deploy --all --context env=prod`,
   **Then** full infrastructure is provisioned: S3 buckets, Glue jobs, Batch queues, MLflow server, Step Functions, EventBridge rules.

2. **Given** the platform is deployed,
   **When** client uploads their data to the configured S3 input path,
   **Then** the first scheduled run executes without any additional configuration.

---

### Edge Cases

- What if MLflow server is unreachable during scoring? → Batch job retries 3x, then fails the state machine.
- What if input data schema changes? → Glue job fails schema validation and halts — no partial output.
- What if a model degrades in prod? → MLflow metrics are logged per run; model can be rolled back by promoting previous version.
- What if `cdk deploy` fails mid-stack? → CDK rollback restores previous state; no partial infrastructure.

---

## Requirements

### Functional Requirements

**Platform**
- **FR-001**: Platform MUST support multiple simultaneous models from the portfolio, each with independent schedule and config.
- **FR-002**: Adding a new model MUST require only a config YAML + training script — no infrastructure code changes.
- **FR-003**: All models MUST be versioned and tracked in MLflow Model Registry.
- **FR-004**: Scoring jobs MUST load models by MLflow model name + stage (`Production`) — never by S3 path.
- **FR-005**: Platform MUST support three environments: `prototype`, `dev`, `prod`, parameterized via CDK context.

**Feature Engineering**
- **FR-006**: Feature engineering MUST run as AWS Glue PySpark job.
- **FR-007**: Glue job MUST join `client_aggregates` + `client_transactions` on `customer_id` (left join).
- **FR-008**: Feature schema MUST be validated before writing output — job fails on schema mismatch.

**Training**
- **FR-009**: Training MUST run as containerized job on AWS Batch.
- **FR-010**: Training job MUST log metrics, parameters, and model artifact to MLflow.
- **FR-011**: Trained model MUST be registered in MLflow as `Staging` — never directly as `Production`.

**Scoring**
- **FR-012**: Scoring MUST run as containerized job on AWS Batch, triggered monthly by Step Functions.
- **FR-013**: Output MUST be Parquet: all feature columns + `prediction` column + `prediction_probability` column.
- **FR-014**: Output MUST be written to `s3://{output_bucket}/predictions/{model_name}/run_date={YYYY-MM}/`.
- **FR-015**: Pipeline MUST be idempotent — re-run overwrites output for the same `run_date`.

**IaC**
- **FR-016**: ALL infrastructure MUST be defined in AWS CDK v2 (Python).
- **FR-017**: CDK stacks MUST be parameterized by environment (`prototype`/`dev`/`prod`).
- **FR-018**: Client deployment MUST require only `client_config.yaml` + `cdk deploy --all`.

**Observability**
- **FR-019**: Every pipeline stage MUST emit structured JSON logs with: stage, model_name, model_version, run_date, input_rows, output_rows, duration_seconds, status.
- **FR-020**: Failed stages MUST halt the Step Functions state machine and send an alert (SNS).

---

### Data Schema

#### Input: `client_aggregates` (Parquet)
| Column | Type | Description |
|---|---|---|
| `customer_id` | string | Unique customer identifier |
| `snapshot_date` | date | Date of the aggregate snapshot |
| `age` | int | Customer age |
| `tenure_months` | int | Months since first contract |
| `contract_type` | string | `monthly` / `annual` / `two_year` |
| `monthly_charges` | float | Current monthly charge |
| `total_charges` | float | Cumulative charges to date |
| `num_products` | int | Number of active products |
| `support_tickets_6m` | int | Support tickets raised in last 6 months |
| `payment_method` | string | `card` / `transfer` / `autopay` |
| `has_churned` | int | Ground truth label (nullable — absent in scoring) |

#### Input: `client_transactions` (Parquet)
| Column | Type | Description |
|---|---|---|
| `customer_id` | string | |
| `transaction_date` | date | |
| `transaction_amount` | float | |
| `transaction_type` | string | `payment` / `refund` / `adjustment` |
| `channel` | string | `online` / `app` / `branch` |

#### Glue Output: `features` (Parquet)
| Column | Type | Description |
|---|---|---|
| `customer_id` | string | |
| `age` | int | |
| `tenure_months` | int | |
| `contract_type_encoded` | int | Label-encoded |
| `monthly_charges` | float | |
| `total_charges` | float | |
| `num_products` | int | |
| `support_tickets_6m` | int | |
| `payment_method_encoded` | int | Label-encoded |
| `tx_count_3m` | int | Transactions in last 3 months |
| `tx_count_6m` | int | Transactions in last 6 months |
| `avg_tx_amount` | float | Average transaction value |
| `days_since_last_tx` | int | Days to snapshot_date |
| `total_spend_6m` | float | Sum of payments last 6 months |
| `refund_rate` | float | refunds / total transactions |

#### Scoring Output: `predictions` (Parquet)
All `features` columns + `prediction` (int, 0/1) + `prediction_probability` (float)

---

### Model Portfolio (v1)

| Model | Type | Target Column | Primary Metric |
|---|---|---|---|
| `churn` | Classification | `has_churned` | AUC-ROC |
| `ltv` | Regression | `lifetime_value` | RMSE |
| `lead_scoring` | Classification | `converted` | F1 |
| `sales_forecast` | Regression | `sales_next_month` | MAE |
| `payment_default` | Classification | `defaulted` | Precision@K |

---

### `model_config.yaml` Structure (per model)

```yaml
model_name: churn
model_type: classification          # classification | regression | clustering
algorithm: xgboost                  # xgboost | random_forest | logistic_regression
schedule: "0 6 2 * *"              # cron: 2nd of every month at 06:00 UTC
input_sources:
  aggregates: s3://{raw_bucket}/client_aggregates/
  transactions: s3://{raw_bucket}/client_transactions/
features_output: s3://{features_bucket}/features/churn/
predictions_output: s3://{output_bucket}/predictions/churn/
target_column: has_churned
hyperparameters:
  n_estimators: 300
  max_depth: 6
  learning_rate: 0.05
mlflow:
  experiment_name: churn_v1
  registered_model_name: churn_classifier
```

---

## Success Criteria

- **SC-001**: Full end-to-end pipeline (feature eng → training → scoring) runs in under 45 minutes for 1M rows.
- **SC-002**: `cdk deploy --all` provisions a complete environment from scratch in a new AWS account.
- **SC-003**: Adding a new model to the portfolio requires zero infrastructure code changes.
- **SC-004**: All pipeline stages are idempotent — re-running produces identical output.
- **SC-005**: MLflow UI shows full lineage: experiment → run → registered model → deployment.

---

## Assumptions

- Client has an AWS account with CDK bootstrapped.
- MLflow is self-hosted within the client's AWS account (not managed service).
- Training is offline batch — no real-time inference in v1.
- Model training data (with labels) is provided by the client alongside raw scoring data.
- v1 scope: churn model only; platform designed to support full portfolio without re-engineering.
- Environments are separate AWS accounts or separate CDK context prefixes within one account.
