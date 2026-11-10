# Aegis Competition V5 — Industry Expert Feedback Implemented

**Date:** 7 November 2026  
**Goal:** Offline 90-day access-review prioritization, not real-time intrusion/anomaly alerting.  
**Audience:** Industry expert, competition judges, and ML teammate.

## Expert feedback → concrete code changes

| Expert point | V5 change | Verification / boundary |
|---|---|---|
| Account lists may come as CSV or MySQL table dumps | Added batch CSV source importer and read-only MySQL table/view reader with explicit column mappings. Supports GitHub/GitLab as platform labels, Windows, Linux, other enterprise applications. | `backend/app/ml/source_ingest.py`; MySQL `.sql` text is **not executed**. A vetted dump must be restored into an isolated authorized MySQL environment first, then read with SELECT-only credentials. Live MySQL source not testable here. |
| AD reporting structure / internal transfers | Stored dated AD reporting snapshots (manager, department, OU); a change between two snapshots generates a reviewable scope-change event. Also accepts HR transfer event CSV. | `DirectorySnapshot`, `EmployeeTransfer`; tested first-import-no-transfer and second-snapshot-change cases. A snapshot difference is **not proof** of an HR transfer—it is a prompt to verify. |
| Multiple AD groups as ML input | Added current employee-level AD group memberships and dated group-change events; aggregate count, privileged-group count, changes, and completeness flags. | `ADGroupMembership`, `ADGroupChange`; group privilege must be supplied by verified mapping, never guessed from the group name. No live AD querying in prototype. |
| Isolation Forest only change if needed | Retained the Isolation Forest baseline with categorical encoding and numerical features; updated model feature schema and retrained; adjustable `--trees` and `--seed`. | Training requires independent labels for evaluation; model rarity is **not** access-needed probability. Continues as provisional until validated. |
| Parallel run against full manual reviews | New freeze/compare workflow: freeze AI shortlist **before** reviewers certify every account; compare recall at selected review budget, missed cases, precision and reviewed minutes. | `backend/app/ml/parallel_review.py`; synthetic tests pass. Actual parallel manual-review labels have **not** yet been collected. |
| Keep thresholds adjustable | Configurable API request `review_budget_percent`, `policy_threshold`, `anomaly_threshold`; frontend sliders; settings stored in audit and review evidence snapshot. | `/api/review-cycles/preview`, `/api/review-cycles/create` and frontend `90-Day Review AI`. Freeze settings for pilots and tune on a separate validation period. |
| Enterprise suitability can be later | Avoided adding live AD connector, event bus, production SQL migrations, or automatic revocations; focused on competition scope. | Scale demonstration remains synthetic, not enterprise-tested. |

## New normalized PostgreSQL tables

The original tables remain: `employees`, `accounts`, `entitlements`, `activities`, `activity_coverage`, `periodic_review_cycles`, `periodic_account_reviews`, and `audit`.

Four new tables support AD context:

- `directory_snapshots`: one current source-attested snapshot per employee: `reporting_manager`, `org_unit`, `ad_department`, snapshot timestamp, group coverage flag and transfer-history coverage flag.
- `ad_group_memberships`: employee-to-AD-group membership with active state, privilege map flag and source metadata.
- `employee_transfers`: dated reporting-scope changes, including department, manager and OU before/after with source.
- `ad_group_changes`: dated AD membership additions/removals.

Identity join is `Employee.code` → `Account.employee_id`, plus the four AD-context tables linked through `Employee.id`. Platform-specific application entitlements remain distinct from AD group memberships. A user can have multiple application accounts.

## ML input features added in V5

`ad_group_count`, `privileged_ad_group_count`, `ad_group_data_known`, `group_changes_window`, `transfers_window`, `manager_changes_window`, `department_changes_window`, `org_unit_changes_window`, `recent_transfer_flag`, `days_since_last_transfer`, `transfer_history_known`, `ad_department_mismatch`, `ad_reporting_known`.

These are aggregated once **per active account per review window**, alongside existing 90-day login, resource-use, privilege and monthly activity features. Employee identifiers, email addresses, names, and raw AD group names are excluded from model inputs. Group names and manager references remain available only as reviewer evidence and database metadata. No risk should be inferred from a missing AD snapshot as though it were a confirmed lack of AD groups or transfers.

## Offline model flow

Application user/permission CSV or read-only MySQL views + dated AD exports + available Windows/Linux/application evidence → normalized PostgreSQL → 90-day feature engineering → **Isolation Forest advisory percentile** + deterministic HR/policy checks → prioritized human review queue → human retain/revoke decision → audit trail. **No automatic permission removal.**

### How to use the parallel review pilot

1. Pick one complete review period and set the review budget/thresholds **in advance**.
2. Run `python -m app.ml.parallel_review freeze --out frozen.csv --budget 10 --policy 25 --anomaly 0.95` to freeze account shortlist before full manual review.
3. Have application owners independently review **all** accounts during the pilot and record one row per account in a CSV: `account_id,decision,review_minutes` with decision `retain`, `remove`, or `followup`.
4. Run `python -m app.ml.parallel_review compare --predictions frozen.csv --manual full_manual.csv`.
5. Compare missing candidates (false negatives), recall and minutes. Preserve all mandatory HR cases and unresolved low-telemetry cases. Tune thresholds **for a subsequent cycle**, not based on the same data used to claim success.

**Metrics:** recall@budget, precision@budget, missed review-needed accounts, manual-reviewed minutes, selection volume, unresolved evidence gaps. `remove` or `followup` in independent manual labels is used as a review-needed positive for the pilot, not as a malicious-activity label.

## Verification status and open limitations

- **30 backend automated tests pass**, including AD-input scenarios, historical scope changes, thresholds, and shadow-review comparison.
- Sample application + AD CSV imports were run end-to-end into a fresh isolated SQLite prototype database; a 90-day queue was frozen.
- Model was retrained from a **synthetic dataset** using updated feature schema. In a held-out fictional batch at a 20% review budget, rule-only recall was about **70.79%**, hybrid recall **67.42%**. Do **not** claim the ML method improved enterprise reviews based on this data.
- No live AD authentication/LDAP reader or live GitLab API integration has been implemented. Existing GitHub integration is optional; CSV is the demonstrated application-account intake.
- Read-only MySQL reader code is provided but not tested against a live MySQL database in this environment. Raw `.sql` dump execution is intentionally out of scope.
- AD snapshots and app inventories require source-specific privacy, quality checks and synchronization. Group deletions should arrive as explicit inactive rows or change-event imports; absence from a partial import is **not** removal.
- Existing databases may require schema migration; the prototype's `create_all()` only creates missing tables, not production-grade migrations.
- Frontend TypeScript/JSX syntax was checked; a full npm build requires installing frontend dependencies.
- A shortlist does not automatically certify or exempt unselected accounts from any mandated review.

## Questions for the industry expert

1. What exports are realistically available from Active Directory and the target application (account IDs, manager, groups, group-change history, transfer dates, entitlement grants, 90-day resource-use events)?
2. For the parallel validation pilot, what minimum recall/maximum missed-review cases would make workload reduction acceptable, and should employee transfers be mandatory-review items or just higher-priority candidates?
