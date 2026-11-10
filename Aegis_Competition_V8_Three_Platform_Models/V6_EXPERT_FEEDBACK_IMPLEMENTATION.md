# Industry Expert Feedback → V6 Implementation Audit

| Expert direction | Code implemented | Evidence/limit |
|---|---|---|
| Exports vary by application | `source_ingest.py` CSV/MySQL table adapter and source-specific column-map JSON | Read-only MySQL approach; real source not connected in test environment |
| AD groups vary by company | New `ad_group_catalog`; scoped stable `(organization,directory,group)` identity; catalog-approved access/non-access/unknown and risk/privilege; no raw group names in ML vector | Directory nesting/effective access and catalog approval processes must be validated by domain owner |
| Transfers always reviewed | `pending_transfer_count`, mandatory queue rule; per-transfer+per-application-account human disposition, including older transfers | API decision records disposition, not upstream modification |
| Continue Isolation Forest | Model retrained for 39 features, schema `quarterly-2.1-governed-ad-transfer` | Model still trained on fictional data; rules-only benchmark essential |
| Full manual vs ML pilot | New `enterprise_pilot.py`: snapshot freeze + digest/HMAC, blind full manual template, 100% completeness, FNR/recall/precision, critical misses, review minutes | Company must approve tolerated missed-case rate. No live blind pilot conducted |
| Adjustable thresholds | Existing API/UI capacity, policy and anomaly sliders; new calibration grid with safety constraint, untouched pilot B requirement | Calibration tool never updates live policy settings |
| Aim toward enterprise ML maturity | AD membership history, time-aware transfer tracking, SHA/HMAC model integrity, drift monitor, Alembic V5-to-V6 migration, modular teammate input contract | Still not enterprise production-certified; major security/performance/workflow items listed in readiness checklist |

## Special safeguards

1. A group name is never sufficient evidence of privileged access; unknown catalog mapping remains unresolved.
2. An employee transfer is a deterministic review obligation, not an optional outlier prediction.
3. Every active linked application account gets its own transfer review; resolving one account does not resolve the other accounts.
4. A transfer more than 90 days old remains mandatory until individually dispositioned.
5. A review is frozen before blind manual labels are collected; any pilot setting change is evaluated on the frozen data and then must be validated on new independent data.
6. A lower review-queue size does not prove a safe reduction in actual review work; no direct upstream access is revoked.

## Reviewer confirmation still needed

- Stable AD group IDs, group purposes, privileged mappings and owners for the target company.
- Clear semantics of HR transfer vs manager/OU change; the project treats all scope changes as mandatory pending human verification.
- Missed-case tolerance and high-severity requirements during shadow pilot.
- Enterprise target data SLAs, audit retention and access certification policy. No assumed approval thresholds.
