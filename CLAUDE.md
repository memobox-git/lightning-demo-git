# Olist Lightning Pipeline

A data pipeline built live in front of an audience:
Kaggle's Olist e-commerce dataset → S3 raw zone → RDS PostgreSQL
(bronze, silver, gold) → a dashboard.

## The situation

| Team | Source | Reported revenue |
|---|---|---|
| Finance | payments export | **R$16.0M** |
| Marketplace | items export | **R$15.8M** |

Nobody can explain the difference.

**Business question:** which products and sellers drive sales, and does late
delivery hurt reviews?

**Goal:** one number everyone agrees on, with a bridge from each team's number
to it.

## How to work

- Before each step, say in one sentence what you'll do.
- Keep answers short. Prefer tables.
- Commit after each completed step with a clear message.

## AWS

| Setting | Value |
|---|---|
| Region | `us-east-1` |
| Tag on every resource | `project=olist-lightning` |

- **Ask before creating, changing or deleting anything in AWS.**
- Never print, log or save passwords or AWS keys.
- The database password lives in Secrets Manager and is fetched at runtime.

Already exists (see `docs/SETUP.md`; do not create again):

| Resource | Name |
|---|---|
| RDS PostgreSQL instance | `lightning-demo-rds` |
| S3 bucket (raw zone) | name in `config.env` only |
| Lambda | `olist-lightning-pipeline` |

## Cloud session

- This session cannot open port 5432. **Every database step is deployed to and
  run in the Lambda `olist-lightning-pipeline`.**
- AWS keys arrive as environment variables. Leave `AWS_PROFILE` unset.

## Settings

- All settings go in `config.env`, which is git-ignored. **Never commit it.**
- **Never commit the bucket name or the account number anywhere**: not in
  code, docs, comments or commit messages. Read them from `config.env` at
  runtime.

## Layers

| Layer | Contract |
|---|---|
| **bronze** | Files exactly as landed. All columns `text`. Plus batch id, source file and load time. |
| **silver** | Correct types. Deduplicated. Bad rows moved to a quarantine table with a reason. Never deleted silently. |
| **gold** | Star schema. Every table states its grain: one row = one what? |
| **meta** | `meta.dq_results`, `meta.lineage`. |

## Business rules

- **GMV = price + freight.** GMV is **0** for canceled or unavailable orders.
- **Never join order items to payments directly.** Roll both up to one row per
  order first, then join on `order_id`.

## Data quality

- A check is a SQL query that returns the **failing rows**.
- Every check has a severity: `block` or `warn`.
- Results are recorded in `meta.dq_results`.
- Build and check each layer in **one transaction**. A failed `block` check
  rolls the layer back. A `warn` is recorded and the build continues.

## Lineage

Every step records source → target with row counts in `meta.lineage`.

## Dashboard

- Reads **gold only**.
- Connects through the read-only role **`bi_reader`**.
