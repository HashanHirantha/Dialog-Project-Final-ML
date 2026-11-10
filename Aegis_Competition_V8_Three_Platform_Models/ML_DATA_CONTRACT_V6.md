# Aegis V6 — ML Inputs and Canonical Data Contract

## Source-specific exports, one model contract

Don't force every company to use the same AD group name. Use a per-source column-mapping profile to normalize the columns below, then a governed catalog to explain group meanings. `organization_key + directory_key + group_key` is the scoped identifier; prefer `objectGUID` or object SID, never display name alone. If the source does not provide a stable ID, use an explicitly documented directory-qualified name fallback and mark identity quality degraded.

| Feed (`--kind`) | Required CSV columns | Optional important columns | Source/meaning |
|---|---|---|---|
| `employees` | `code,name,email,department,job_title,status` | `manager_username` | HRIS identity and status |
| `accounts` | `platform,username,employee_code,status` | `last_login,last_login_source` | App accounts e.g. GitLab/GitHub/Linux/Windows |
| `entitlements` | `platform,username,resource,permission,sensitivity` | `active,last_used_at,evidence_source,source` | Authoritative grants, not inferred from login logs |
| `activities` | `platform,username,resource,event_type,occurred_at,source` | — | Normalized OS/application audit event |
| `coverage` | `platform,event_type,start_at,end_at,source,attested_by` | — | Usage-source collection attestation |
| `directory_snapshots` | `employee_code,snapshot_at,reporting_manager,org_unit,ad_department,groups_complete,transfer_history_complete,source` | — | Source-attested AD reporting snapshot |
| `ad_groups` | `employee_code,group_name,active,source` | `organization_key,directory_key,group_key,synced_at` | Dated group membership observations |
| `ad_group_catalog` | `organization_key,directory_key,group_key,display_name,group_kind,access_relevance,privilege_tier,risk_tier,approved_by,source,policy_version` | `resource_scope,allowed_departments,effective_at` | Approved interpretation of *company-specific* groups |
| `employee_transfers` | `employee_code,occurred_at,from_department,to_department,from_manager,to_manager,source` | `from_org_unit,to_org_unit` | Dated internal transfers (always mandatory review) |
| `ad_group_changes` | `employee_code,group_name,action,occurred_at,source` | — | Group add/remove delta |

### Expected value domains

- `access_relevance`: `access`, `non_access`, `unknown`.
- `group_kind`: `security`, `distribution`, `m365`, `other`, `unknown`.
- `privilege_tier`: `privileged`, `standard`, `unknown`.
- `risk_tier`: `critical`, `high`, `medium`, `low`, `unknown`.
- `allowed_departments`: `|` separated approved department names; blank means no department policy assertion, not an allow-all rule.
- `source`: identifiable read-only connector or attested export system. Dates: ISO 8601 UTC. No credentials in datasets.

### Input boundary

`source_ingest.py` supports CSV or a read-only *table* from a vetted MySQL database. A plain `.sql` dump is **not** parsed or executed by the ML system. Restore a vetted dump outside production into an isolated staging instance and SELECT only authorized fields. Use `--column-map-file` to translate exports to canonical names. The importer limits file size and per-transaction rows; a production ingestion pipeline needs job-level idempotency, cryptographic provenance, snapshots/deltas and transactional batching.

### Activity semantics

- `login_success`/`login_failed` provide authentication evidence only.
- `resource_use` is a verified resource-use event only if it can be joined to an active permission on the same account/resource, and its source is trustworthy.
- `permission_change` is contextual, not proof that the new permission was exercised.
- `ActivityCoverage` currently records platform-level attestation. **It is not a guarantee that every individual application entitlement was observed.**

### Reviewer output contract

`periodic_preview` returns one ranked record per currently active account, fields including `account_id`, `platform`, `username`, `employee_status`, `review_priority`, `policy_score`, `anomaly_percentile`, `mandatory`, `pending_transfer_ids` (private workflow evidence), `pending_transfer_count`, `evidence_gap`, `reasons`, `activity_summary`, and `automatically_certified=false`.

Five `review_priority` classes: `mandatory`, `selected`, `deferred_candidate`, `insufficient_evidence`, `deprioritized_not_certified`. **All pending transfers are mandatory, regardless of ML score or review budget**. Output scores are not auto-revocation decisions.

### Interface ownership

Logs/inventory team owns validated account, event, entitlement and HR/AD exports. ML owns normalized feature engineering, group catalog interpretation, mandatory policy, Isolation Forest, ranking, calibration/evaluation and model monitoring. API team owns authentication/RBAC, import permissions, scheduling, reviewer decisions/audit and export/reporting. An organization-specific application-owner mapping must be authoritative before any deployment.
