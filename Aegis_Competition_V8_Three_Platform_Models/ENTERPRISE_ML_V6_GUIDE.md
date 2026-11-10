# Aegis V6 — Enterprise-Oriented ML Workstream and Integration Guide

**Audience:** Competition ML owner, log-collection engineer, API engineer, and industry expert  
**Status:** Working, tested competition prototype with selected enterprise-oriented ML safeguards. **Not production certified.**  
**Date:** 2026-11-07

## 1. Exact industry-expert requirement

Aegis is NOT intended to become an EDR/SIEM or alert on every anomalous live event. At each periodic (typically 90-day) certification, Aegis uses historical activity and permissions to highlight accounts needing human attention. Application/system owners retain final decisions and evidence obligations. The 3-account story is illustrative only; the design handles an arbitrary inventory size. **A smaller ML queue is not proof that excluded accounts are safe or exempt from mandatory reviews.**

## 2. Answers to the expert's three questions

### Q1: Different companies have different AD groups and export schemas

The model **does not learn raw group display names**. An approved organization-owned `ad_group_catalog` maps a stable directory group identity to meaning. The join key is `(organization_key, directory_key, group_key)`; use real `objectGUID` or SID when available, not display name. One company may treat a group as privileged, while another uses a similarly named group solely for distribution. Categories: `access`, `non_access`, `unknown`; privilege: `privileged`, `standard`, `unknown`.

The ingest adapter accepts source-specific column mappings (`--column-map` or `--column-map-file`). For example, one AD export's `objectSid` becomes canonical `group_key`, `employeeNumber` becomes `employee_code`; another company's export has a different map. No universally named groups are assumed. An unmapped group is **an evidence gap**, not automatically an admin group and not automatically a safe group. Group definitions need an accountable approver, source and version. Approved `allowed_departments` can identify access-bearing groups that no longer match the current department.

**Current limitations:** directory group nesting/effective rights, grant inheritance and complete export/snapshot reconciliation still require dedicated source metadata. The app-wide identity schema is not multi-tenant safe; operate in one organization's isolated database until tenant authorization is built by the API team.

### Q2: EVERY internal transfer must trigger access review

The HR transfer feed or a detected AD reporting-scope change creates a dated `employee_transfers` entry. For each active linked application account, the ML queue checks for transfers without a corresponding `transfer_access_dispositions` entry. **Any pending transfer forces `mandatory` priority; score thresholds, ML score, and review capacity cannot exclude it.** A 135-day-old unreviewed transfer remains mandatory. A reviewer decision records a separate account/transfer disposition, with reviewer, date, rationale and review cycle; other accounts remain pending. A reviewer decision does not modify upstream permissions. A new transfer arriving after a frozen review remains pending for the next review.

This uses authoritative HR changes where available and treats AD manager/OU/department differences as scope-change review evidence, not absolute proof of an HR transfer.

### Q3: Shadow pilot: ML AND full manual reviews in parallel

During the pilot **all accounts are manually reviewed**, independently of model selection. First freeze the ML model outputs and parameters, then give application owners a *blind* manual template. Compare results only after every manual account decision is complete. Measure recall, false-negative/missed-review rate, missed high/critical cases, precision, workload queue size and timed review minutes. **The company must explicitly supply the acceptable missed-case rate**; no default is silently assumed. Tune parameters on pilot A, then verify them against a separate untouched pilot B before recommending policy changes. V6 does not auto-promote pilot thresholds or remove review obligations.

## 3. Data structures, source interfaces and security boundaries

| Data | Source/Owner | Required proof | Storage |
|---|---|---|---|
| Employee HR data | HRIS/directory team | stable employee ID, current status, timestamps | `employees`, `employee_transfers` |
| Application account inventory | GitHub/GitLab/CSV or read-only MySQL table | stable account key and employee linking | `accounts` |
| Entitlements | Application owner | current grants and source/version | `entitlements` |
| AD membership | Directory team | org + directory + stable group ID, observed_at, active | `ad_group_memberships`, `ad_group_membership_history` |
| AD group meaning | AD/application security owner | access meaning, privilege, role scope, approver, version | `ad_group_catalog` |
| Directory reporting structure | AD export | dated manager, department, OU; completeness | `directory_snapshots` |
| 90-day activity | Log collection teammate | normalized event, event time, account/resource linkage | `activities` |
| Log completeness | Connector/system owner | source, interval, attester, what kind of event is covered | `activity_coverage` |
| Transfer recertification | Application owner | human decision + rationale | `transfer_access_dispositions` |
| Periodic decisions | API/review teammate | frozen evidence, decisions, audit trail | `periodic_review_cycles`, `periodic_account_reviews` |

**Important:** a successful login does not prove that an assigned permission was used. A `resource_use` event counts only when its account and resource match an active entitlement; authentication-only logs are supporting evidence. Missing coverage is never interpreted as confirmed inactivity.

## 4. ML architecture and features

Application/AD exports + normalized logs → PostgreSQL → point-in-time 90-day aggregation → source-quality gates → **Isolation Forest** anomaly reference plus deterministic policy checks → prioritized account queue → authorized human review → audit.

**Model:** `scikit-learn` IsolationForest (160 trees by default), OneHotEncoder for department/platform/account type/permission profile and StandardScaler for 35 numeric features (39 total features). The exact columns are versioned as `quarterly-2.1-governed-ad-transfer` in `backend/app/ml/periodic_features.py`. No names, employee IDs, email addresses or raw AD group names are ML features. Pending review disposition counts are a **policy/workflow input, not an ML feature**.

Feature examples: successful/failed logins, verified resource-use counts, distinct usage days and resources, three 30-day buckets, privileged entitlement counts, AD group count, *approved mapped-access group count*, mapped/non-access/unmapped group counts, out-of-scope mapped groups, catalog coverage fraction, manager/OU/department changes, age and counts of transfers, and telemetry completeness. Recent transfers remain mandatory even if model training fails.

**Why keep Isolation Forest?** Unsupervised, efficient on sparse/uneven tabular account histories, reference cohort percentile available for priority assistance. An anomaly percentile is NOT a misuse probability. The algorithm must be replaced only if independently reviewed cases show it fails against simpler rules/ranking. V6 retains the transparent rules-only comparison and does not claim ML superiority.

## 5. Operational queue logic

- **Mandatory:** inactive or unmapped employee, plus *every unreviewed internal transfer on each active linked application account*; cannot be squeezed out by a review budget.
- **Selected:** policy concerns or advisory anomaly above configurable thresholds, ranked by policy severity then ML rarity, up to the nonmandatory budget.
- **Deferred candidate:** triggered but cannot fit into target review capacity; still unresolved.
- **Insufficient evidence:** unverified activity coverage or unmapped AD group meaning; never silently certified.
- **Deprioritized, not certified:** no configured triggers; must still be handled under the organization's review policy.

Scope changes might demand a special off-cycle certification workflow; the pipeline currently detects them in the next run, while an API/notification scheduler is for the API team to implement.

## 6. Running the prototype

### A. New V6 project database

Copy `.env.example` to `.env`, configure the secrets and local database, then:

```bash
# From repository root — source-level service with Docker
# WARNING: SEED_DEMO=true is only for isolated demonstrations.
docker compose up --build
```

Backend imports (from `backend/`, with dependencies installed):

```bash
python -m app.ml.source_ingest --kind ad_group_catalog --csv ../sample-data/enterprise-ml/company_a_group_catalog.csv
python -m app.ml.source_ingest --kind ad_group_catalog --csv ../sample-data/enterprise-ml/company_b_group_catalog.csv
# Import employees, application accounts and entitlements BEFORE groups/transfers.
python -m app.ml.source_ingest --kind ad_groups --csv ../sample-data/enterprise-ml/ad_groups_export.csv
python -m app.ml.source_ingest --kind employee_transfers --csv ../sample-data/enterprise-ml/transfers.csv

# For a source with different column headings:
python -m app.ml.source_ingest --kind ad_groups --csv /path/to/ad_export.csv \
  --column-map-file ../sample-data/enterprise-ml/column_map_ad.json

# Train / demonstrate (fictional training data only)
python -m app.ml.periodic_generate --rows 1600
python -m app.ml.periodic_train
python -m app.ml.competition_demo --accounts 80000 --budget 10 --with-ml
pytest -q
```

For MySQL source tables, set `AEGIS_SOURCE_MYSQL_URL` for a SELECT-only account and use `--mysql-table TABLE_NAME` instead of `--csv`. **Never execute arbitrary `.sql` dump content.** Restore vetted backups to an isolated staging MySQL instance and ingest with a read-only connection.

### B. Upgrade an existing V5 DB (do not discard data)

Take a tested backup, then from `backend/` run:

```bash
# Export DATABASE_URL to point at the backed-up existing V5 DB.
alembic upgrade head
```

Alembic upgrade `v6_0001` converts the directory membership identity key and creates governance/history/disposition tables. It was exercised against an isolated SQLite V5 schema fixture. Run it on a staging copy of your actual PostgreSQL installation before any production application restart. Fresh V6 installations can use the prototype `Base.metadata.create_all`. The Alembic revision is a V5-to-V6 migration, NOT a replacement for a complete production migration history. This project still uses auto-create at startup for ease of competition setup.

**Docker note:** this V6 backend image includes Alembic files. Existing V5 PostgreSQL volumes must be migrated before restarting V6 app code.

## 7. Industry expert's full-manual shadow pilot

**Pilot A** — configure budget, policy, anomaly threshold in advance; freeze all 90-day account predictions using an authenticated/controlled database snapshot:

```bash
python -m app.ml.enterprise_pilot freeze \
  --frozen pilotA_predictions.json \
  --manual-template pilotA_blind_manual.csv \
  --budget 10 --policy 25 --anomaly 0.95
```

Give the independent application owner the blank manual template and the **authoritative application grants** separately, never the ML model rank. Complete every row with `decision` (`retain`, `remove`, `followup`), `review_minutes`, `severity`, `reviewer`, and `reason`. Store both artifacts in access-controlled locations. For protected environments configure `AEGIS_PILOT_SIGNING_KEY` and `AEGIS_REQUIRE_SIGNED_PILOTS=true`. SHA-256 alone detects accidental changes but does not authenticate an artifact's publisher.

```bash
python -m app.ml.enterprise_pilot evaluate \
  --frozen pilotA_predictions.json --manual pilotA_blind_manual.csv

# ONLY after the expert signs off on an acceptable missed-case tolerance:
python -m app.ml.enterprise_pilot calibrate \
  --frozen pilotA_predictions.json --manual pilotA_blind_manual.csv \
  --max-missed-case-rate 0.05 \
  --budgets 10,20,30 --policies 15,25,35 --anomalies .90,.95,.99
```

`0.05` above is an **illustrative pilot input**, not a recommended or approved missed-case rate. The calibration tool rejects a missing threshold and never deploys a setting. The output must say `candidate_requires_untouched_pilot_B`; use new time/organizations for B, then a governance review. For critical missed cases, the default pilot limit is **zero** unless an explicit decision is approved.

**Measures:** `false_negative_rate`, `recall`, `precision`, `missed_high_or_critical`, `missed_critical`, confidence upper bound for missed-case rate, review time by selected vs nonselected, platform breakdown. Estimated avoided minutes remain **hypothetical** until a future approved reduced-review process is timed.

## 8. Model monitoring and security controls

```bash
python -m app.ml.enterprise_monitor \
  --baseline ml_data/periodic_synthetic.csv \
  --current /path/to/quarterly_aggregate_features.csv \
  --psi-warning 0.25 --unknown-group-warning 0.10
```

Outputs feature distribution changes, new categorical values, and unknown catalog membership share. PSI alerts are an investigation heuristic, not an automatic reason to retrain. Document source quality/freshness, row count, unexpected column values, group catalog approval history, model schema/version/sha256, pilot settings, and false negatives. No online learning, no automatic retraining and no auto-revocation are implemented. For model artifacts use trusted storage and, in protected mode, `AEGIS_MODEL_SIGNING_KEY` and `AEGIS_REQUIRE_SIGNED_MODELS=true` before `joblib.load` (unsafe on untrusted files).

## 9. Recommended enterprise readiness phases (ML scope only)

**E1 — Data contracts:** freeze organizational IDs, stable group keys, source-specific column maps, 90-day retention, source observability, snapshot completeness, permission effective rights semantics.

**E2 — Policy management:** approved group catalog, role/scope mapping, mandatory HR transfer / termination policy, approval versioning, controlled exceptions with expiry, review ownership and segregation of duties.

**E3 — Model governance:** separately trained per-organization or approved cohort models, reliable reference labels, time-aware holdout, deterministic versioning, input drift and source fail-closed gates, protected model registry, explainable evidence.

**E4 — Shadow pilot:** full independent manual review, immutable signed predictions, false-negative/error analysis and threshold tuning on pilot A, independent validation on B, explicit owner-approved missed-case tolerance.

**E5 — Production prerequisites:** tenancy and RBAC isolation, strong user/API identity, encrypted event and model storage, key management, formal data retention, API idempotency, migrations and rollback, batch jobs/queues, processing SLAs, bounded memory, observability, stress/load tests against a real DB, secure human-only remediation and regulatory review requirements.

**Do not claim the existing application has completed E5**. V6 upgrades the competition ML component and its integration contract; the backend/frontend remain a prototype. The team's logging and API phases are separate deliverables.

## 10. Teammate handoff

**Logs/inventory teammate:** implement Windows/Linux/GitHub/GitLab/application adapters that produce canonical source and timestamp fields; attach resource-use coverage attestations, stable account IDs, identity linkage, HR changes, and AD memberships; never claim all permissions used from login logs.

**You (ML teammate):** maintain approved AD group catalog structure, feature schema, Isolation Forest training, data-quality gates, mandatory transfer rule, ranking, shadow pilot, evaluation, drift/model integrity and export specifications; do not build raw OS agents.

**API/dashboard teammate:** secure schema migrations, connect these modules to authorized API calls and scheduled jobs, expose queue/pending-transfer/evidence-gap views, enforce reviewer assignment and RBAC, store decisions/audit, provide pilot upload workflow and governance/config approval controls. Avoid automatic permission removal.

## 11. Verification status (2026-11-07)

- **41 backend tests passed** (after V6 new features and migration-related updates).
- Isolated SQLite V5-to-V6 Alembic upgrade exercised.
- 80,000 fictional accounts scored/ranked in **~2 seconds in memory** on the testing environment; no live log/database throughput proof.
- Synthetic model retrained for feature schema `quarterly-2.1-governed-ad-transfer`.
- No full frontend build verified; TSX source is updated with transfer / group-mapping summary text.
- No real enterprise manual-review labels or organizational policy catalog have been supplied.
- Current tests do NOT establish acceptable missed-case performance. Historical synthetic benchmarking previously showed the hybrid approach need not outperform rules-only. **Human review and a pilot are mandatory before any workload reduction claim.**

See `ML_DATA_CONTRACT_V6.md`, `V6_EXPERT_FEEDBACK_IMPLEMENTATION.md`, and `ENTERPRISE_READINESS_CHECKLIST.md` for handoff details.
