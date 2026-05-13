# AWS ML Batch Pipeline Constitution

## Core Principles

### I. Modular ML Pipeline
Each ML model type (classification, regression, clustering) is an independent, swappable module.
Modules share a common interface: `fit(df) → model`, `predict(df) → df_with_predictions`.
No business logic inside model modules — only ML logic.

### II. Data Contract First
All inputs and outputs are defined as explicit schemas (Parquet columns + dtypes) before implementation.
Mocked data must be schema-compatible with production data.
Predictions output = original input columns + `prediction` column, saved to S3 as Parquet.

### III. Monthly Batch — Idempotency (NON-NEGOTIABLE)
Every pipeline run must be idempotent: re-running for the same month produces identical results.
Outputs are partitioned by `run_date=YYYY-MM` on S3.
No state is stored outside S3.

### IV. Test with Mocks, Deploy with Real
All development uses mocked Parquet data stored locally or in a dev S3 bucket.
Production reads from external Parquet source via S3.
Mock and real data share the same schema — no code changes between environments.

### V. Observability
Every pipeline run logs: start time, input row count, model used, output row count, S3 output path, duration.
Errors are surfaced immediately and halt the pipeline — no silent failures.

## AWS Constraints

- **Compute**: AWS Batch (or SageMaker Processing Jobs) for monthly runs
- **Storage**: S3 for input data, models, and output predictions
- **Orchestration**: Step Functions or EventBridge Scheduler for monthly trigger
- **Data Prep**: AWS Glue Job joins `client_aggregates` + `client_transactions` → feature-ready Parquet
- **Dependencies**: managed via Docker container or SageMaker-managed environment
- **IAM**: least-privilege roles — pipeline reads input bucket, writes output bucket only

### VI. Infrastructure as Code (NON-NEGOTIABLE)
All AWS infrastructure is defined in AWS CDK (Python).
No manual console changes — every resource must exist in CDK stacks.
CDK stacks are split by concern: `GlueStack`, `BatchStack`, `OrchestrationStack`, `StorageStack`.
`cdk diff` before every `cdk deploy` — changes are reviewed, not applied blindly.

## Development Workflow

1. Define schema → 2. Write mocked data → 3. Implement Glue job → 4. Implement model module → 5. Test locally → 6. Deploy via CDK

All code changes require passing unit tests before deployment.
Infrastructure changes (CDK/CloudFormation) are reviewed before apply.

## Governance

This constitution supersedes all other practices.
Amendments require updating this document and communicating to all contributors.
All implementation decisions are evaluated against these principles first.

**Version**: 1.0.0 | **Ratified**: 2026-05-13 | **Last Amended**: 2026-05-13
