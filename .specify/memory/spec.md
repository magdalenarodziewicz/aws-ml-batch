# Feature Specification: Monthly Customer Churn Batch ML Pipeline

**Feature Branch**: `001-churn-classification-pipeline`

**Created**: 2026-05-13

**Status**: Draft

**Input**: Monthly batch ML pipeline — customer churn classification on AWS

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Monthly Churn Predictions Run (Priority: P1)

A data analyst triggers (or the scheduler triggers) the monthly pipeline.
Raw data lands in S3. Glue joins the tables. The classification model scores every customer.
Output Parquet lands in S3 with original features + `churn_predicted` column.

**Why this priority**: Core business value — without this, nothing else matters.

**Independent Test**: Can be tested end-to-end with mocked Parquet files locally;
delivers a complete prediction file as output.

**Acceptance Scenarios**:

1. **Given** `client_aggregates` and `client_transactions` Parquet files exist in S3 input bucket,
   **When** the pipeline runs for month `2026-05`,
   **Then** a Parquet file is written to `s3://output-bucket/predictions/run_date=2026-05/churn_predictions.parquet`
   containing all original columns + `churn_predicted` (0 or 1).

2. **Given** the pipeline is re-run for the same month,
   **When** output already exists,
   **Then** the file is overwritten with identical content (idempotent).

3. **Given** input data contains 10 000 rows,
   **When** the pipeline completes,
   **Then** output contains exactly 10 000 rows (no row loss).

---

### User Story 2 — Data Preparation via Glue (Priority: P1)

Glue job reads two raw Parquet tables, joins them on `customer_id`,
applies feature engineering, and writes a single feature-ready Parquet file to S3.

**Why this priority**: No features = no model. Glue prep is a hard dependency for scoring.

**Independent Test**: Glue job can be run standalone with mocked inputs;
output schema is verifiable without running the ML model.

**Acceptance Scenarios**:

1. **Given** `client_aggregates` and `client_transactions` in S3,
   **When** Glue job runs,
   **Then** output Parquet has the expected feature schema (see Requirements) with no nulls in key columns.

2. **Given** a customer appears in `client_aggregates` but not in `client_transactions`,
   **When** Glue job runs,
   **Then** that customer is included with transaction features filled as 0 (left join semantics).

---

### User Story 3 — IaC Deployment via CDK (Priority: P2)

A developer runs `cdk deploy` and the entire pipeline infrastructure is provisioned:
S3 buckets, Glue job, AWS Batch job definition, Step Functions state machine, EventBridge scheduler.

**Why this priority**: Reproducibility — environment must be recreatable from code.

**Independent Test**: `cdk synth` produces a valid CloudFormation template without manual resources.

**Acceptance Scenarios**:

1. **Given** AWS credentials and CDK bootstrapped,
   **When** `cdk deploy --all` is run,
   **Then** all stacks deploy successfully with zero manual console steps.

2. **Given** an existing deployment,
   **When** a CDK stack is modified,
   **Then** `cdk diff` shows only the expected delta before `cdk deploy`.

---

### Edge Cases

- What if `client_aggregates` has duplicate `customer_id` rows? → Glue deduplicates (keep latest by `snapshot_date`).
- What if the input bucket is empty? → Pipeline fails fast with a clear error log, no partial output.
- What if the model file is missing from S3? → Batch job fails with explicit error, does not fall back silently.
- What if a feature column has >10% nulls? → Log a warning but continue; document as known data quality issue.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Glue job MUST join `client_aggregates` and `client_transactions` on `customer_id` (left join, aggregates as left table).
- **FR-002**: Glue job MUST output a single Parquet file with the feature schema defined below.
- **FR-003**: ML module MUST load a pre-trained scikit-learn classification model from S3.
- **FR-004**: ML module MUST output a Parquet file = all input feature columns + `churn_predicted` (int, 0 or 1) + `churn_probability` (float, 0.0–1.0).
- **FR-005**: Output MUST be partitioned on S3 as `predictions/run_date=YYYY-MM/`.
- **FR-006**: Pipeline MUST be orchestrated by AWS Step Functions triggered monthly by EventBridge Scheduler.
- **FR-007**: All infrastructure MUST be defined in AWS CDK (Python), split across `StorageStack`, `GlueStack`, `BatchStack`, `OrchestrationStack`.
- **FR-008**: Pipeline MUST be idempotent — re-running for the same `run_date` overwrites output.
- **FR-009**: Every run MUST emit structured logs: `run_date`, `input_rows`, `output_rows`, `model_version`, `duration_seconds`, `status`.

### Data Schema

#### Input: `client_aggregates` (Parquet)
| Column | Type | Description |
|---|---|---|
| `customer_id` | string | Unique customer identifier |
| `snapshot_date` | date | Date of the aggregate snapshot |
| `age` | int | Customer age |
| `tenure_months` | int | Months since first contract |
| `contract_type` | string | `monthly` / `annual` / `two_year` |
| `monthly_charges` | float | Current monthly charge (PLN) |
| `total_charges` | float | Cumulative charges to date |
| `num_products` | int | Number of active products |
| `support_tickets_6m` | int | Support tickets raised in last 6 months |
| `payment_method` | string | `card` / `transfer` / `autopay` |
| `has_churned` | int | Ground truth label: 1 = churned (for training only; nullable in scoring) |

#### Input: `client_transactions` (Parquet)
| Column | Type | Description |
|---|---|---|
| `customer_id` | string | Unique customer identifier |
| `transaction_date` | date | Date of transaction |
| `transaction_amount` | float | Transaction value (PLN) |
| `transaction_type` | string | `payment` / `refund` / `adjustment` |
| `channel` | string | `online` / `app` / `branch` |

#### Glue Output: `features` (Parquet) — joined + engineered
| Column | Type | Description |
|---|---|---|
| `customer_id` | string | |
| `age` | int | From aggregates |
| `tenure_months` | int | From aggregates |
| `contract_type_encoded` | int | Label-encoded contract type |
| `monthly_charges` | float | |
| `total_charges` | float | |
| `num_products` | int | |
| `support_tickets_6m` | int | |
| `payment_method_encoded` | int | Label-encoded payment method |
| `tx_count_3m` | int | Transaction count last 3 months |
| `tx_count_6m` | int | Transaction count last 6 months |
| `avg_tx_amount` | float | Average transaction amount (all time) |
| `days_since_last_tx` | int | Days between snapshot_date and last transaction |
| `total_spend_6m` | float | Sum of payments last 6 months |
| `refund_rate` | float | refunds / total transactions |

#### Output: `predictions` (Parquet)
All columns from `features` + `churn_predicted` (int) + `churn_probability` (float)

### Key Entities

- **Customer**: identified by `customer_id`, appears in both input tables.
- **Pipeline Run**: identified by `run_date` (YYYY-MM), produces one output partition.
- **Model**: versioned scikit-learn artifact stored in S3 (`models/churn/v{N}/model.pkl`).

---

## Success Criteria *(mandatory)*

- **SC-001**: Pipeline completes end-to-end (Glue + Batch) in under 30 minutes for 1M rows.
- **SC-002**: Output row count equals input row count (zero row loss).
- **SC-003**: `cdk deploy --all` provisions environment from scratch in a new AWS account.
- **SC-004**: All unit tests pass locally before any AWS deployment.
- **SC-005**: Re-running pipeline for same `run_date` produces byte-identical output.

---

## Assumptions

- Input data arrives in S3 before the 1st of each month (pipeline runs on the 2nd).
- Model is pre-trained and stored in S3; training pipeline is out of scope for v1.
- Feature engineering is deterministic (no random state outside the model).
- `customer_id` is a stable, non-null identifier in both tables.
- AWS CDK v2 (Python) is used for all IaC.
- Development environment uses mocked Parquet files with the same schema as production.
