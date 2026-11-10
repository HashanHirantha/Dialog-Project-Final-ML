# ML Team Handoff — Aegis V4

## Overview

Your ML part supports **periodic (quarterly) access review prioritization**, not real-time intrusion detection. The three-account story is merely a storyboard; there is no three-account size limit. The key deliverable is a reviewed shortlist with honest evidence, data-quality states, and a human decision trail.

## Run the full project

1. In the project root, copy `.env.example` to `.env` and set secure credentials and a 32+ character `JWT_SECRET`.
2. Start Docker: `docker compose up --build`.
3. Frontend: `http://localhost:5173`; API docs: `http://localhost:8000/docs`.
4. Open **90-Day Review AI** to preview a queue and create a review campaign. Only admins can create campaigns.
5. Reviewers can retain or recommend revocation with a reason; simulated revocation does not change upstream access.

If Python dependencies are installed locally, from `backend`:

```bash
python -m app.ml.periodic_generate --rows 1600
python -m app.ml.periodic_train
python -m app.ml.periodic_benchmark --budget 10
python -m app.ml.periodic_benchmark --budget 20
python -m app.ml.competition_demo --accounts 10000 --budget 10 --with-ml
python -m app.ml.competition_demo --accounts 80000 --budget 10 --with-ml
PYTHONPATH=. pytest -q
```

This project bundles a **synthetic-only** model artifact for showing the workflow; retrain before presenting after any feature changes. Only load your own trusted `.joblib` files (joblib/pickle is not safe for unknown sources).

## Key ML files

| File | Purpose |
| --- | --- |
| `backend/app/ml/periodic_features.py` | Data from PostgreSQL/SQLite; SQL grouped event counts; 3 monthly buckets; entitlement and evidence matching; coverages |
| `backend/app/ml/periodic_model.py` | Isolation Forest, peer calibration, group-heldout model evaluation, model serialization |
| `backend/app/ml/periodic_queue.py` | Mandatory HR cases + policy score + ML percentile + budget cap + unresolved backlogs |
| `backend/app/ml/periodic_generate.py` | Fictional benchmark generator; training data only |
| `backend/app/ml/periodic_train.py` | Retrain after matching input schema |
| `backend/app/ml/periodic_benchmark.py` | Rules-only vs hybrid at review-budget thresholds; missed positive cases |
| `backend/app/ml/competition_demo.py` | In-memory storyboards and synthetic 80k ranking, optional `--with-ml` |
| `backend/tests/test_periodic_review.py` | Evidence coverage, time windows, normal queue, backlog, inactive HR, resource mapping |

## Integration contract for data collector

At minimum provide (a) **authoritative entitlements**: account ID, platform, resource, permission, sensitivity, active status; (b) HR or identity owner status; (c) verified account/resource-use events: account, resource, type `resource_use`, UTC time, source; (d) attested collection-coverage windows for the relevant resource/event type. A successful login is **not** proof of use of a given permission. Without reliable log coverage, do not classify access as unused.

Input features include department, platform, account type (must be supplied by a real connector; current DB defaults to unknown), permission profile, entitlement count, privilege counts, role-policy mismatches, verified resource-use events, monthly usage buckets, successful/failed logins, active days, last use age, and evidence coverage. Human names, account IDs and emails must NOT be model inputs.

The existing CSV adapter accepts at most 3,000 rows per upload and is insufficient for a real 80,000-account deployment. Replace it with streaming/batched enterprise connectors in a production design.

## Model and review logic

- **Hard HR/identity rules** are always mandatory; do not ask ML whether the departed employee's status is genuine.
- **Model** scores 90-day outlierness relative to trained normal accounts, gated on near-complete usage-collection coverage.
- **Policy** is the primary triage signal; ML may surface additional unusual accounts but cannot certify normal accounts.
- **Budget** is an upper target, not a fixed quota. Mandatory cases can exceed it.
- **Deferred candidates** and **insufficient evidence** remain open work, never marked automatically approved.
- **Human review** alone decides retain or recommend revoke. Upstream revocation is simulation-only.

## Competition demo

Show a user-defined population (e.g. 10,000 or 80,000 fictional accounts) and:

1. Overall account count and policy-required reviews.
2. Review budget **vs actual selected**; show it doesn't select ordinary accounts just to fill a quota.
3. `deferred_candidate` and `insufficient_evidence` counts separately.
4. Actual ML-scored records and percentile (rarity, not probability of misuse).
5. Human reviewer decision, rationale, audit export and simulated remediation.
6. Rules-only vs ML hybrid comparison **with failure cases**. In the included synthetic benchmark the hybrid is not proven better than the rules-only baseline.

To show ML in the scale script, include `--with-ml`; without that switch the script benchmarks **policy-only** ranking. Neither run is a database load test or a real-world accuracy metric.

## Reviewer routing

Set `PLATFORM_REVIEWERS_JSON` in `.env`, e.g. `{\"GitHub\":\"reviewer\",\"Linux\":\"reviewer\"}`. The chosen username must already exist in the app's users. This is a demo-only replacement for an authoritative application-owner directory; if not configured, employee line managers are used as fallback.

## Safety acceptance criteria

- [ ] No false claim that missing logs mean unused access.
- [ ] No actual deletion/grant of production accounts.
- [ ] Every mandatory lifecycle case goes to a reviewer even if over budget.
- [ ] Normal-only population produces an empty shortlist, not an arbitrary fixed 10%.
- [ ] Backlogged candidates stay visibly unresolved.
- [ ] Evaluate recall@K/false-negative count against **independently labeled** ground truth.
- [ ] Benchmark application-owner review workflow and measure **actual time saved** on controlled trials.

