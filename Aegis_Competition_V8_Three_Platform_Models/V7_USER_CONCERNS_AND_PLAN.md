# Aegis V7 — ML, manual-review overlap, and software/website scope

## What the project should actually solve

**Access governance, not employee surveillance:** Analyze 90 days of evidence for every included account, independently of whether the organization manually reviews that account. Rank accounts that require business justification or access recertification. Keep human control over retention and revocation. Do not treat an anomaly as proof of policy violation, wrongdoing, or unnecessary access.

V6 used an account-level Isolation Forest model trained on engineered access/usage features. It **did not parse or classify arbitrary websites/processes** and does **not learn what is wrong solely from HR people's logs**. The model learns numerical distributions; role-based *authorization* comes from authoritative HR, group-to-permission and company policy mappings. The model can highlight rarity but cannot decide a resource is prohibited without governance evidence.

## Why a full manual pilot is different from permanent operations

- **In normal operation:** ML runs over **the entire account inventory** and provides mandatory, selected, deferred, insufficient-evidence, and deprioritized classes. Humans review selected accounts, all mandatory transfer/HR cases, and any further cases company policy requires. Already completed, independently attested reviews can be reconciled to avoid duplicate investigation; *accounts are never deleted merely because they were reviewed*.
- **In a validation pilot:** If the company can continue its ordinary full manual certification (or perform a complete manual review of a smaller representative cohort), freeze ML predictions first, keep the owners blind, and compare **every decision** with the shortlist. This gives an exact missed-case count for the reviewed cohort.
- **If full independent manual review is impractical:** Predetermine a **random, blinded sample of both ML-selected and ML-unselected accounts**. V7 contains a sampled-pilot helper with explicit selection probabilities and simple weighted estimates. Those are estimates, **not** an exact enterprise missed-case rate; size/uncertainty and a separate untouched pilot B matter. If humans review only ML-selected cases, misses among unselected cases cannot be measured (verification bias).
- No acceptable missed-case tolerance is silently assumed. Only the organization may approve it. Pilot results **never** auto-promote thresholds or remove compliance review obligations.

### Already reviewed accounts

When the company has already completed a valid review of an account *for the same cycle*, its authenticated reviewer, decision, date, evidence, and transfer-specific scope must be linked with the ML queue. V7 provides `reconcile_completed_manual_reviews` as an **ML-side prototype helper**. It separates the selected accounts into `already_reviewed_and_attested` versus `still_to_review`. It never deletes accounts, changes permissions, or clears outstanding transfers without explicitly attested coverage. The API teammate must implement authenticated synchronization and audit trails in the production workflow.

## What Windows / Linux / GitHub logs can and cannot establish

| Source | Relevant to access reviews | Cannot conclude alone |
|---|---|---|
| Windows Security login events | Account authentication, last login, failure rates | A specific application entitlement was used |
| Windows process creation 4688 (when enabled) | Company-managed software execution evidence | Program is prohibited or employee had inappropriate permissions |
| Linux SSH/PAM and auditd EXECVE | Authentication and configured process execution evidence | Any arbitrary command implies policy violation |
| GitHub/GitLab audit/API | Membership, role/permission changes, supported repository usage events | All repositories/permissions are used, or all log sources have 90-day coverage |

**Important:** Windows Event ID 4688 is not captured unless process creation auditing is configured. Include-command-line auditing can expose sensitive text and should not be collected for this access-review purpose. `auditd` must be configured for the operations desired. GitHub audit event types, retention and permissions differ by deployment. All collectors need provenance and account identity mapping.

## Song / music example

**Ordinary music listening is out of scope**. The ML component must not ingest whole browser histories, music URLs, page content or productivity scores. A generic browser visit is not a current-permission mismatch. No model or rule should mark an employee risky solely for listening to music. Even if the site is blocked by a separate company network policy, that belongs to a separately governed acceptable-use/security workflow, not an automatic account-revocation recommendation.

### Company-managed resource policy checks (new V7 sidecar)

V7 introduces two ORM tables:

1. `governed_resource_policies`: organization, platform, approved company-managed business application/software/web app stable asset key, explicit allowed departments/roles, policy version, approved by, effective date.
2. `governed_resource_uses`: source-attributed, successful observed use of **only those company-managed resources** (not personal browsing), account identity, date and source. Duplicate source event keys are rejected.

`app/ml/governed_import.py` has a limited offline CSV importer that requires approved policy signers and a trusted collector-name list. It rejects unknown managed resource keys and does not keep arbitrary website visits. Production needs cryptographic source provenance, organization scoping, authentication, event schema/quality and volume limits.

`app/ml/governed_resources.py` flags an **advisory access-policy mismatch** only when all are true:

- Current company-approved, active, effective-dated mapping for the exact business asset exists;
- Existing account is linked to an employee with sufficient department/role information;
- Account's observed managed asset usage is supported by the approved collector and is not a failed attempt;
- The employee's current department/job scope conflicts with an explicit authorization policy;
- Observation did not precede the policy's effective date or the employee's latest recorded transfer.

Then `periodic_features.py` attaches `managed_policy_conflict_count` and `managed_policy_conflict_resources` to the review record. `periodic_queue.py` adds an advisory rule-based priority increment, **not a new Isolation Forest feature or claim that the user misbehaved**. The model artifact and 39 ML features remain unchanged, avoiding unvalidated retraining. This is a *policy-review enrichment*, not an EDR detector.

**Known limitation:** `organization_key='demo'` is the prototype default; it is not an authenticated tenant boundary. Complete historical role/department snapshots and event-time policy versions are required before asserting that an action was unauthorized at the time it occurred. V7 only says **review current access in light of approved policy**, not that the employee previously did something wrong.

## Pilot sampling API (new)

`app/ml/pilot_sampling.py` exports:

- `freeze_blind_sample(frozen, selected_sample=N, unselected_sample=M, seed=...)`: choose random accounts from both strata of the **already frozen full-population** ML scores.
- `write_blind_template(sample, path)`: no model selection or scores leak to reviewers.
- `evaluate_sample(sample, completed_manual)`: simple stratified inverse-probability weighted **estimated** missed-case and recall metrics, timed review minutes, explicit no-auto-promotion warning. For reliable intervals and acceptance, use adequate samples and a statistician; don't treat the estimate as a guarantee.
- `reconcile_completed_manual_reviews(frozen, manual_reviews, cycle_key=...)`: suppress **duplicate work** when a review is verifiably complete for that cycle; keeps mandatory unresolved cases pending.

The existing V6 `app/ml/enterprise_pilot.py` full-manual gold-standard path remains unchanged.

## Ownership / teammate handoff

| Owner | What changes |
|---|---|
| Log collector teammate | Normalize OS and app events; support **only managed business resource IDs** and trusted event provenance, not unrestricted website tracking |
| ML teammate (you) | Feature extraction, Isolation Forest, company-specific resource policy checks, anomaly + policy reason separation, full-account ranking, sampled pilot estimates and threshold evaluation |
| API/backend teammate | Authoritative HR/AD, resource policy approval and review authorizations, authenticated policy/event ingestion, verified manual-decision reconciliation, audit records and UI |

### CSV example usage for a configured database

```bash
cd backend
alembic upgrade head
# The security-owner approval and linux-collector trust declaration must be
# determined by an organization admin; they are illustrative in these commands.
python -m app.ml.governed_import \
  --kind policy --csv ../sample-data/managed-resources/policies.csv \
  --approved-by security-owner
python -m app.ml.governed_import \
  --kind usage --csv ../sample-data/managed-resources/managed_usage.csv \
  --trusted-source linux-collector
```

The sample usage file references an `example-hr-user` account, **which must exist** for the second command to succeed; replace with a synthetic account in your local test DB. One generic music record gets ignored because it is **not a company-managed policy resource**. No real employee data is included in the file.

### Validation

Run from `backend`: `PYTHONPATH=. pytest -q`. V7 includes regression tests for source verification, music not flagged, policy approval, blind sampling, missed-case estimation, and already-reviewed work reconciliation.

## Remaining improvements before enterprise deployment

1. Identity matching (Windows SIDs, Linux UIDs/host qualifiers, GitHub account IDs) and entitlement-to-resource usage attribution.
2. Versioned HR department/job records and policy snapshots to interpret whether access was allowed *at the time*.
3. Real source authenticity, tenant isolation/RBAC, retention and privacy approvals; prohibit unrestricted browsing collection.
4. Scalable database-side aggregates and event storage partitioning for 80k+ accounts; V7 policy sidecar currently queries scoped observations into application memory.
5. Authenticated external manual-review import and verified matching at `cycle_id + account_id + individual transfer IDs` (API team).
6. Independent pilot A threshold tuning, untouched pilot B evaluation with explicit organization-defined missed-case acceptance; no silent threshold defaults.
7. Model peer-group validation by role/application and separate baselines as evidence warrants, without unnecessary model proliferation.

## Example reviewer record (illustrative only)

```json
{
  "account_id": 101,
  "platform": "Linux",
  "review_priority": "selected",
  "policy_score": 35,
  "anomaly_percentile": null,
  "reasons": [
    "Managed business resource use differs from approved access policy; verify authorized business need"
  ],
  "activity_summary": {
    "managed_policy_conflict_count": 1,
    "managed_policy_conflict_resources": [
      {"resource_kind": "managed_software", "resource_key": "production-db-admin", "policy_version": "2026Q4"}
    ]
  },
  "automatically_certified": false
}
```

### Final distinction

- **ML outlier**: unusual combination of an account's historical access/use features. Not automatically wrongdoing.
- **Policy mismatch**: reason for a *human review* based on an approved mapping. Not automatically a violation at the time of event.
- **Ordinary personal browsing/music**: not collected or scored by this access review component.
- **Unauthorized attempt vs assigned entitlement**: not interchangeable; a denied request may belong to threat detection, while access certification asks whether an account should still hold an existing permission.
