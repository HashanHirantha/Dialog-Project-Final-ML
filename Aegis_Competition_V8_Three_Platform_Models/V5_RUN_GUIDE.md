# Aegis Competition V5 — ML Workstream Run Guide

This is an offline 90-day review-prioritization competition prototype, **not** an EDR system.

## 1. Start the project

From repository root, copy `.env.example` to `.env`, configure secrets and run `docker compose up --build` (PostgreSQL + FastAPI + React). In the API documentation, authenticated administrators can import CSV records and preview/create review cycles.

For the command-line workflow, run from `backend/` in a configured Python environment with `pip install -r requirements.txt`. `DATABASE_URL` must refer to the same target database used by the backend. On a fresh prototype database, the importer creates missing tables. For an already deployed database, use managed migrations before schema changes.

## 2. CSV application and AD input

Import in order: employees → accounts → entitlements → directory_snapshots → ad_groups → employee_transfers → ad_group_changes → activities → coverage.

```bash
for kind in employees accounts entitlements directory_snapshots ad_groups employee_transfers ad_group_changes activities coverage; do
  python -m app.ml.source_ingest --kind "$kind" --csv "../sample-data/expert-inputs/$kind.csv"
done
```

The fictional sample bundle includes GitHub/GitLab as application platforms. The importer processes batches instead of requiring one large upload, and supports column mapping:

```bash
python -m app.ml.source_ingest --kind employees --csv /path/employee_export.csv \
  --column-map '{"employee_id":"code","full_name":"name"}'
```

Do not use real employee records without the company's permission.

## 3. MySQL table dumps

When an application provides a MySQL `.sql` dump, have an authorized operator restore it to a **separate, trusted staging database**. Aegis does not run arbitrary dump SQL and should not connect with write credentials. Query the staging table using a dedicated SELECT-only MySQL account:

```bash
export AEGIS_SOURCE_MYSQL_URL='mysql+pymysql://read_only_user:YOUR_SECRET@staging-host/staging_db'
python -m app.ml.source_ingest --kind accounts --mysql-table application_users \
  --column-map '{"app":"platform","login_name":"username","hr_code":"employee_code","account_state":"status"}'
```

The MySQL reader's schema/column mapping depends on the application's actual dump. Its live connection path was not tested here.

## 4. New AD fields

- `directory_snapshots.csv`: employee_code, snapshot_at, reporting_manager, org_unit, ad_department, groups_complete, transfer_history_complete, source.
- `ad_groups.csv`: employee_code, group_name, active, source, optional privileged flag established by group-governance mapping.
- `employee_transfers.csv`: employee_code, occurred_at, from/to department, from/to manager, source, optional from/to org_unit.
- `ad_group_changes.csv`: employee_code, group_name, action (`added`/`removed`), occurred_at, source.

If two successive AD snapshots show a manager, department or OU change, an additional `ad_snapshot_diff` transfer/scope event is created. This is a **change signal**, not proof that an HR-approved transfer took place. Source-completeness flags must be truthful.

## 5. Retrain the feature-schema-v5 model

```bash
python -m app.ml.periodic_generate --rows 1600
python -m app.ml.periodic_train --trees 160 --seed 42
python -m app.ml.periodic_benchmark --budget 20
```

The model is **Isolation Forest**, a provisional unsupervised reference baseline. Retrain any previous V4 artifact: V5's added AD/transfer features change the model schema. Do not report synthetic validation as real enterprise performance.

## 6. Preview and tune shortlist parameters

`POST /api/review-cycles/preview` (admin JWT) with:

```json
{"window_days":90,"review_budget_percent":10,"policy_threshold":25,"anomaly_threshold":0.95}
```

The 90-Day Review AI screen has matching sliders. `review_budget_percent` is a maximum nonmandatory selection capacity, not a quota. Mandatory lifecycle issues override it. Unselected accounts are **not** certified as safe. Saved review cycles snapshot all selected-account evidence, thresholds, and human-only approvals.

## 7. Parallel ML/manual evaluation

```bash
python -m app.ml.parallel_review freeze --out frozen_predictions.csv \
  --budget 10 --policy 25 --anomaly 0.95
```

Have the expert's designated application owners independently review **every account**. Fill out a CSV:

```csv
account_id,decision,review_minutes
101,retain,3.5
102,followup,5
103,remove,7
```

These IDs must match the frozen file, not demo placeholders. Then compare:

```bash
python -m app.ml.parallel_review compare \
  --predictions frozen_predictions.csv --manual full_manual_review.csv
```

The script rejects incomplete or duplicate account labels; output includes recall@budget, precision@budget, false negatives, and potential reviewer minutes outside the selected queue. **Potential minutes are not observed savings** while a parallel full review is running. Tune on this pilot, then validate on a later untouched period.

## 8. Tests

```bash
python -m pytest -q
```

The release was tested with 30 passing backend tests and an end-to-end fictional import + frozen shortlist. Full frontend npm build and live MySQL/AD tests remain unverified.
