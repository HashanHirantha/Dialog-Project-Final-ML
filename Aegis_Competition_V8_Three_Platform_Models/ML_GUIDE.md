**V4 NOTICE:** This legacy guide is supplemented and superseded by `ML_V4_RUN_GUIDE.md` and `EXPERT_REQUIREMENTS_AUDIT_V4.md`. The competition is not an academic publication, and the example employee accounts do not define scope.

# Aegis ML workstream — implementation and 7-day development plan

**Owner:** Member B (Machine Learning / Explainable Risk Analysis)

**Goal:** Detect **unusual entitlement patterns** and contribute a small, clearly identified advisory signal to an existing **human-reviewed** access governance workflow. This is **not** employee behavior monitoring, threat attribution, verified policy compliance, or autonomous account removal.

## 1. Your concrete deliverables

| File | Why it exists |
|---|---|
| `backend/app/ml/features.py` | Shared 12-feature schema; extraction from live ORM objects; missing-evidence handling |
| `backend/app/ml/generate.py` | Deterministic, clearly synthetic training/evaluation example data |
| `backend/ml_data/synthetic_demo.csv` | 1,250 synthetic records (1,000 routine, 250 scenario-defined suspicious) |
| `backend/app/ml/model.py` | One-hot encoding + Isolation Forest + group holdout + metrics + model persistence |
| `backend/app/ml/train.py` | Command-line training, validation and saved model metadata |
| `backend/app/ml/demo.py` | Predict on four illustrative single-entitlement cases |
| `backend/app/ml/benchmark.py` | Compare rule-only risk thresholds with ML on the **same** held-out examples |
| `backend/app/risk.py` | Rules-based risk score enhanced by a separately trained ML anomaly flag |
| `backend/tests/test_ml.py` | Automated tests for model, features and API integration safety |
| `backend/ml_artifacts/` | Local-only trusted `.joblib` + report `.json` generated during training |

**Shared contracts with teammates:** Member A supplies `Employee`, `Account` and `Entitlement` with **verified** permission information and optionally `last_used_at` + evidence source. Member C exposes `/api/ml/status`, `/api/risk/recompute`, `/api/entitlements` and an ML percentile in the entitlement detail panel. Agree together before changing input schemas.

## 2. ML pipeline

```text
Windows/Linux/GitHub inventories + HR identity mapping
                 |
        ENTITLEMENT / ACCOUNT records
                 |
       Privacy-minimized features (12)
         /                      \
  Explainable rules      Trusted offline model artifact
         |                      |
   Policy risk points      Isolation Forest inference
         |                      |
         +-----------+----------+
                     |
   +10 points ONLY if ML anomaly > held-out-independent
          normal training threshold
                     |
      Risk explanations + review recommendation
                     |
            HUMAN ACCESS REVIEW
                     |
        Approve retain / approve revoke
                     |
              SIMULATED removal
```

**Critically:** The model is an *anomaly detector*, not a calibrated 0–100 risk predictor. The displayed risk score is a transparent rule-driven composite with a **maximum 10-point, unvalidated ML flag**. The `anomaly_score` API field contains an **empirical normal-training percentile between 0 and 1**, not the likelihood of misuse.

## 3. Step-by-step training (Windows PowerShell / macOS / Linux)

All commands below start from `backend/` inside the updated project.

### Option A: Local Python

```bash
cd backend
python -m venv .venv
# Windows PowerShell:
.venv\Scripts\Activate.ps1
# Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
python -m app.ml.generate --out ml_data/synthetic_demo.csv
python -m app.ml.train --csv ml_data/synthetic_demo.csv
python -m app.ml.demo
python -m app.ml.benchmark --csv ml_data/synthetic_demo.csv
python -m pytest -q
```

The trained model will be saved to `backend/ml_artifacts/access_anomaly_model.joblib`, together with a JSON metadata file. The generated model is a **trusted local artifact**: do **not** download, unpickle, or load model files from strangers (joblib uses pickle). If scikit-learn versions differ, retrain with the local version rather than attempting to load a foreign artifact.

### Option B: Docker (no local Python setup)

```bash
# From project root; configure .env first, then:
docker compose up -d --build
# Inside the backend container, train against the supplied synthetic CSV:
docker compose exec backend python -m app.ml.train --csv ml_data/synthetic_demo.csv
# Print example model predictions:
docker compose exec backend python -m app.ml.demo
docker compose exec backend python -m app.ml.benchmark --csv ml_data/synthetic_demo.csv
# Then sign into the UI as admin and click Recalculate Risk.
```

The Docker volume persists the generated model on your host in `backend/ml_artifacts/`. Running `docker compose down` does not delete this folder. Training **is never triggered by HTTP**; it runs in the trusted development environment only.

## 4. Feature engineering

| Feature | Type | Interpretation |
|---|---|---|
| `department` | Category | Team of the verified owner, or `Unknown` |
| `platform` | Category | `Linux`, `GitHub`, `Windows AD`, etc. |
| `permission_class` | Category | `read`, `write`, `privileged`, `other` |
| `sensitivity` | Category | Resource sensitivity reported by the data owner |
| `employee_status` | Category | `active`, `inactive`, `unknown` |
| `account_status` | Category | Account lifecycle as reported by source |
| `unmapped` | Boolean | No verified employee identity mapping |
| `outside_role` | Boolean | Outside the demonstrative department allowlist (needs policy review) |
| `resource_use_observed` | Boolean | Verified **resource-use** evidence exists |
| `last_used_days` | 0–365 | Age of most recently **verified resource-use** evidence; 0 when unknown |
| `last_login_observed` | Boolean | Account login timestamp available |
| `last_login_days` | 0–365 | Age of last account login; 0 when unknown |

No employee name, email, username, IDs, or raw per-event behavioral logs are passed to the model. When `resource_use_observed = 0`, `last_used_days = 0` is only a placeholder **paired with the observed flag**, not evidence of use or non-use. A fresh login **does not reset** verified resource-use age.

## 5. Training and evaluation design

1. **Data**: Start from `ml_data/synthetic_demo.csv` to validate your pipeline; its `scenario` and `label` are generated by us. This dataset cannot establish generalization or real-world performance.
2. **Target**: For real evaluation, have qualified IAM reviewers independently annotate whether each access grant **requires investigation** (`label=1`) or is a **well-supported normal grant** (`label=0`). Keep evidence, reviewer, policy version, and date outside the ML feature vector.
3. **Group split**: Use `group_id` (employee/account identity only for grouping) to keep all of a person's entitlements together in train or test, avoiding leakage.
4. **Fit**: Isolation Forest is fitted **only on label=0** records in the training partition. It does not train on the held-out labels.
5. **Threshold**: Flag an access as anomalous if its raw anomaly score is greater than the **95th percentile** among confirmed-normal training observations. The precise operating threshold requires tuning against independently reviewed operational data.
6. **Metrics**: On the untouched held-out partition report precision, recall, F1, ROC-AUC, average precision and confusion matrix in this order: `TN, FP, FN, TP`. Report class sizes alongside metrics. Inspect performance by platform, department, and missing-evidence status where sample sizes support it.
7. **Compare to rules**: Use rule-only risk scoring as your baseline. Document whether statistical anomalies identify genuinely useful additional cases, not merely restate existing rules.
8. **Validate drift**: Track entitlement source completeness, input-feature distributions, false positives, reviewer overrides, and retraining schedule; avoid automatically updating the model during the review process.

Do **not** claim model precision/recall based on synthetic labels equals operational detection accuracy. A model predicting a human-created anomaly scenario is not proof it detects unnecessary employee access.

## 6. API contract (for your teammate)

| Method | Endpoint | Who | Response |
|---|---|---|---|
| GET | `/api/ml/status` | Logged-in | `ready`, state, model name, training/evaluation metadata |
| POST | `/api/risk/recompute` | Admin | `analyzed`, risk counts, `anomaly_model_used`, `ml_status`, `ml_issue` |
| GET | `/api/entitlements` | Logged-in | `risk_score`, `reasons`, `anomaly_score`, `anomaly_metric` per entitlement |

To test the REST flow: log into `http://localhost:8000/docs` with a demo administrator, call the status endpoint, then recompute risk, then query entitlements. `anomaly_score=null` means ML did not run. `anomaly_score=0.98` means 98th percentile of the normal reference **not 98% probability of abuse**. `risk_score` remains a human-review priority, not a risk probability.

When no model is present, the entire application still runs with **rule-only** scoring and the API explicitly reports model unavailable. If model validation fails, the backend warns and returns rule-only scores rather than silently reporting ML output.

## 7. Illustrative evaluation using the generated synthetic CSV

The supplied dataset is made up of 1,250 **fictional scenario examples**. At seed 42 and 160 trees, the held-out split yielded:

| Held-out metric | Demonstration value |
|---|---:|
| Total | 315 |
| Normal (`label=0`) | 234 |
| Scenario-defined anomaly (`label=1`) | 81 |
| Precision | 0.8571 |
| Recall | 0.8889 |
| F1 | 0.8727 |
| ROC-AUC | 0.9636 |
| Average precision | 0.9122 |
| Confusion matrix `TN,FP,FN,TP` | `222,12,9,72` |

The accompanying `ML_EVALUATION.md` compares two policy thresholds and ML on exactly the same held-out records; on this scenario-created dataset the medium-risk policy baseline **does better** than ML. This is expected, because the scenarios were constructed from policy-like signals. These numbers **must be labeled “synthetic pipeline sanity check”** in your presentation. They were observed in the included reproducible local run, not measured against a real company. They may change with dependency versions or revised scenarios.

## 8. Your 7-day timetable

| Day | Your tasks and expected completion evidence |
|---|---|
| 1 | Agree with teammates on data fields, annotation target, audit semantics; complete `features.py` explanation |
| 2 | Inspect missingness and evidence provenance. Run synthetic data generator and document every field |
| 3 | Run OneHotEncoder + Isolation Forest training; save local artifact and metadata; validate threshold |
| 4 | Perform group-held-out evaluation, inspect confusion matrix, compare against rule-only baseline |
| 5 | Integrate with `/api/risk/recompute`; confirm `anomaly_score` appears in backend and dashboard |
| 6 | Run 7+ tests; manually verify orphan, unrelated admin, unknown last-use and normal cases; note false positives |
| 7 | Present evaluation charts, training reproducibility, demo outputs, operational limitations and handover |

## 9. Presentation explanation (example)

**Case:** Finance Analyst owns Linux `dev:production-db` administrative access. A resource-use audit record says the entitlement was last used 95 days ago. The rule-based engine reports privilege, sensitivity, policy mismatch and stale verified use; it creates a human review priority. Separately, a trained Isolation Forest compares the *combination* of features with confirmed-normal entitlements and may flag it as unusual. The reviewer must verify job necessity and authorized exceptions before deciding. The system cannot prove malicious conduct, nor can it execute an external revocation.

**Second case:** The same access has **no last-use data**, but a login happened yesterday. It is not valid to say “the permission has not been used in 95 days.” The feature pipeline records **missing use evidence** instead. That is a critical quality assurance check.

## 10. Recommended next development iteration

- Collect de-identified, permission-grounded real-world inventory data with authorization; separate development/testing organizations or time periods.
- Establish independent reviewed labels and policy exceptions; compare a policy-only baseline to anomaly detection and document incremental detection value.
- Evaluate threshold trade-offs, high false-positive costs, rare teams/platforms, drift and feature coverage.
- Add model registry/versioning, signed model artifacts, secure per-tenant storage, monitoring and human feedback gates.
- Do not train against open-ended raw employee surveillance logs, and never grant AI direct revoke privileges.
