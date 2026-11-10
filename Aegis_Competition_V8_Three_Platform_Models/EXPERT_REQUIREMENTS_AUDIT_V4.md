# Aegis Competition — Industry Expert Requirements Audit (V4)

**Project type:** Competition prototype, NOT an academic publication, commercial-grade IGA product, or real-time EDR.

## What the expert actually requires

The account examples mentioned in the audio are **illustrations only**. The real application must be general across many applications and accounts; some companies have 80,000 accounts or more on one application. The model should examine approximately three months of access and entitlement evidence **before a scheduled certification campaign**, identify likely review-needed outliers, and provide a much smaller queue to system/application owners. A human independently decides to retain or recommend revocation. The model is not a real-time anomaly-alert product, and it must never automatically delete accounts.

## Expert statements mapped to V4 code

| Requirement | Status | Implementation / boundary |
| --- | --- | --- |
| Quarterly, not real-time monitoring | Implemented | `/api/review-cycles/preview`, `/api/review-cycles/create`; no streaming EDR inference |
| Compare prior ~90 days | Implemented | `periodic_features.aggregate_accounts`, 90-day model schema |
| Detect unusual **accounts**, not individual events | Implemented | One row per account; `periodic_model`, Isolation Forest |
| Look for behavior **over time** | Improved in V4 | New 30-day usage buckets (early, middle, recent) + login, resource, distinct activity day summaries |
| Provide relevant access/role evidence | Implemented as demo policies | Entitlements, admin privilege, resource sensitivity, mismatch and recent changes. Actual policies must be supplied by organizations. |
| Reduce number of engineer investigations | Improved in V4 | `make_review_queue` uses a **maximum** budget, not a quota; no forced selection of normal accounts. |
| Treat HR leavers/orphans as urgent | Implemented | Mandatory lifecycle queue cannot be suppressed by budget or ML. HR records are authoritative, not anomaly predictions. |
| Missing telemetry must not mean 'safe' | Improved in V4 | `insufficient_evidence` category, not automatically deprioritized or certified. |
| Avoid hiding review-worthy accounts due to budget | Improved in V4 | `deferred_candidate` backlog is visible and counted. |
| Let application/system owner make decision | Improved in V4 | `PLATFORM_REVIEWERS_JSON` selects an app owner reviewer; line manager/fallback if not configured. |
| One human decision per account | Implemented | Persistent `PeriodicAccountReview` task with reasons, permissions snapshot, reviewer rationale and audit trail |
| No live account removal by ML | Implemented | No actual upstream deletion. Revocation simulation requires approved human decision and admin. |
| Tens of thousands of accounts | **Partial validation** | 80,000-row *synthetic in-memory ML scoring/ranking* exercised. SQL aggregate now reduces 90-day event loading. Production DB ingestion, concurrency, APIs and actual 80,000-account SLA NOT verified. |

## What's different from V3

1. **Fixed quota bug:** V3 selected top 10% even if every account was ordinary. V4 only selects actual candidates, up to the budget plus mandatory overrides.
2. **False clearance risk:** V3 could label unknown-coverage accounts as deprioritized. V4 separates `insufficient_evidence` and `deferred_candidate` for follow-up and never automatically certifies.
3. **History blind spot:** V3 used total 90-day activity but not a time trend. V4 includes verified resource-use counts across three month-sized sections.
4. **Incorrect activity attribution:** V4 counts resource-use events as access evidence only if the resource matches an active account entitlement; last-used timestamps without an evidence source are ignored for use inference.
5. **Audit scaling issue:** V3 loaded all events into Python. V4 performs database-side grouped aggregates (still requires an actual load test and production indexes/migrations).
6. **Anomaly calibration:** V4 optionally compares model scores within `platform|account_type` cohorts with sufficient reference observations, falling back to global calibration when small.
7. **System owner routing:** V3 assigned employee line managers first. V4 supports platform/application owner routing, with a demo fallback.
8. **Model vs rules:** ML is used as a secondary signal; policy severity leads. Model accuracy must be benchmarked against a rules-only queue, not assumed.

## Output statuses (semantics matter)

- `mandatory`: HR identity or lifecycle anomaly; a human MUST investigate independently of queue size.
- `selected`: strong policy signal or model anomaly selected for human review within configured capacity.
- `deferred_candidate`: review-needed signal detected but budget did not cover it. **Not safe or approved.** Must be handled under an approved policy or expanded queue.
- `insufficient_evidence`: cannot justify an inactivity conclusion because log collection coverage is unknown or incomplete. **Not safe or approved.** Requires data-owner follow-up or independent review.
- `deprioritized_not_certified`: no configured trigger under usable evidence. **Not guaranteed safe, not automatically certified.** Applicability of full certification still depends on company policy.

Every model output is an **advisory prioritization**. No status alters real access.

## Benchmark — fully fictional held-out cases

Run `python -m app.ml.periodic_benchmark --budget 10` and `--budget 20` from `backend`.

Results from the included **synthetic** dataset with the V4 model (not real enterprise validation):

| Review budget | Rules-only recall | Hybrid recall | Positive cases missed by hybrid |
| --- | ---: | ---: | ---: |
| 10% | 45.98% | 45.98% | 47 of 87 |
| 20% | 77.01% | 74.71% | 22 of 87 |

**Important:** At 20% the ML-assisted shortlist performs WORSE than rules-only on this constructed test. Do not claim the ML module improves recall, detects all unnecessary accounts, or safely reduces mandatory certification by 80–90%. ML needs real labels and acceptance thresholds; company policy may still require every account to be reviewed.

## What is still needed before deployment

- Authoritative HR events (hires/transfers/leavers), account-owner mappings, exception/contractor/service-account types, and app-specific usage instrumentation.
- Audited resource-use **coverage at the connector/resource level**, not only platform-level attestations.
- Per-application peer baselines with sufficient sample sizes, calibrated score thresholds, and sampling for uncertain low-priority cases.
- Reviewer-verified case labels and independent (preferably future-quarter) validation; evaluate **recall@K, precision@K, missed cases and review minutes saved**, with drift and fairness checks.
- Role/permission policy import, service/break-glass exemptions, real IAM connectors, scheduled certification jobs, bulk processing, migrations, encryption and least-privilege access to logs.
- Full load testing with 80,000 live database records and millions of events. The included performance demo does not establish this.
- Organization-specific legal/compliance approval for reduced human review; NIST 800-53's periodic reviews are organization-defined, not universally optional.

## Security and ethics

Do not use passwords, message content or keystrokes. Prefer the minimum necessary entitlement/activity metadata, retention limits and reviewer access controls. Only review purpose and business need, not employee productivity. Keep human review and traceable evidence; do not revoke users automatically.

## References

- [NIST SP 800-53 Rev. 5, AC-2 Account Management](https://csf.tools/reference/nist-sp-800-53/r5/ac/ac-2/): account reviews and aligning account management with terminations and transfers.
- [NIST SP 800-53 Rev. 5, AC-6(7) Review of User Privileges](https://csf.tools/reference/nist-sp-800-53/r5/ac/ac-6/ac-6-7/): periodical review of assigned privileges as specified by the organization.
- [NIST SP 800-53 Rev. 5 official](https://csrc.nist.gov/pubs/sp/800/53/r5/final).
