# CrossBorderPay ML PoC — Learning Roadmap
**Goal:** Learn the complete ML lifecycle by building an enterprise-grade fraud detection system.
**Outcome:** Be able to independently build the model from scratch on real CrossBorderPay production data.

---

## Where You Are Now

```
[Stage 3 of 15 — Data Quality — 70% complete]

Stage 1: Dataset Fundamentals       ✅ Complete
Stage 2: Pandas Fundamentals        ✅ Complete
Stage 3: Data Quality               🔄 In Progress (invalid values + dtype pending)
Stage 4: EDA                        🔄 Partially done in Untitled-1.ipynb
Stage 5: Fraud-Focused EDA          🔄 Partially done in Untitled-1.ipynb
Stage 6: Feature Engineering        ❌ Not Started
Stage 7: Feature Validation         ❌ Not Started
Stage 8: Train/Test Split           ❌ Not Started
Stage 9: Logistic Regression        ❌ Not Started
Stage 10: XGBoost                   ❌ Not Started
Stage 11: LightGBM                  ❌ Not Started
Stage 12: Isolation Forest          ❌ Not Started
Stage 13: Model Evaluation          ❌ Not Started
Stage 14: Explainability (SHAP)     ❌ Not Started
Stage 15: End-to-End Pipeline       ❌ Not Started
```

---

## The Roadmap

```
TODAY
  │
  ▼
[01_eda.ipynb — Complete]
  Run Restart + Run All. Confirm leakage gate PASS.
  Gate: All cells output. AUC < 0.90.
  │
  ▼
[02_rule_baseline.ipynb]
  Build 4 rules. Calculate precision, recall, F1 manually.
  Gate: You can state the rule baseline recall from memory.
  This number is the bar every ML model must beat.
  │
  ▼
[03_data_cleaning.ipynb]
  Fix missing values, drop zero-variance, convert dtypes.
  Gate: Zero NaN in output. Fraud rate unchanged.
  │
  ▼
[04_feature_engineering.ipynb]
  Build 30+ features. Join all 3 tables.
  Build behavioural features using only historical data.
  Gate: Clean feature matrix saved. No raw columns. No leakage.
  │
  ▼
[05_feature_validation.ipynb]
  Remove redundant and leaked features.
  Gate: Final feature list with justification for every column.
  │
  ▼
[06_train_test_split.ipynb]
  Temporal 70/20/10 split. Verify no date overlap.
  Gate: Fraud rate ~0.49% in all three partitions.
  │
  ▼
[07_logistic_regression.ipynb]     ← FIRST ML MODEL
  Scale features. Train LR. Evaluate on validation set.
  Interpret coefficients in fraud terms.
  Gate: AUC-PR >= 0.60. You can explain every coefficient.
  │
  ▼
[08_xgboost.ipynb]
  Train XGBoost. Compare vs LR and Rule Baseline.
  Gate: AUC-PR >= 0.72. Feature importance matches domain intuition.
  │
  ▼
[09_lightgbm.ipynb]
  Train LightGBM. Select champion.
  Gate: Champion declared with written justification.
  │
  ▼
[10_isolation_forest.ipynb]
  Add anomaly detection layer. Combine with champion score.
  Gate: Combined model evaluated. Clear conclusion on whether it helps.
  │
  ▼
[11_model_evaluation.ipynb]        ← TEST SET USED FOR FIRST TIME
  Evaluate ALL models on held-out test set exactly once.
  Gate: All 5 PoC success criteria pass for champion model.
  │
  ▼
[12_threshold_optimization.ipynb]
  Find optimal threshold. Map to GREEN / AMBER / RED tiers.
  Gate: Threshold saved. Business cost analysis complete.
  │
  ▼
[13_shap_explainability.ipynb]
  Generate SHAP values. Top-5 explanation per transaction.
  Gate: Every RED-tier transaction has a human-readable explanation.
  │
  ▼
[14_batch_scoring.ipynb]
  CSV in → scored CSV + JSONL audit log out.
  Gate: Pipeline runs on any CSV in V3.1 schema.
  │
  ▼
[15_final_demo.ipynb]              ← POC COMPLETE
  Clean demo. Runs < 5 minutes. Ready for CTO presentation.
  │
  ▼
POC SIGN-OFF
  │
  ▼
PHASE 2: Real CrossBorderPay Data
  Apply everything you built to production data.
  You do this independently.
```

---

## How Each Notebook Builds Toward the Final Model

| Notebook | What You Gain | What Breaks Without It |
|---|---|---|
| 01 EDA | Understanding of the data before touching it | You would build features from columns you don't understand |
| 02 Rule Baseline | The benchmark every ML model must beat | No way to claim ML adds value |
| 03 Data Cleaning | Clean, consistent input | NaN values crash model training |
| 04 Feature Engineering | The signals the model learns from | Raw columns have too little signal |
| 05 Feature Validation | Confident the matrix has no leakage or redundancy | Model appears to work but fails in production |
| 06 Train/Test Split | Honest metric evaluation | Inflated metrics that don't hold on new data |
| 07 LR | Interpretable baseline ML + scaling knowledge | No ML baseline to compare against |
| 08 XGBoost | Non-linear pattern capture, champion candidate A | Miss the majority of multi-signal fraud |
| 09 LightGBM | Champion candidate B, model selection | May miss a better model |
| 10 Isolation Forest | Catch never-seen fraud patterns | Completely blind to new typologies |
| 11 Evaluation | Honest final metrics on unseen data | Cannot claim PoC success |
| 12 Threshold | Correct operational risk tier cutoffs | Wrong block/review/pass decisions |
| 13 SHAP | Explainable decisions for RBI audit | Cannot justify any model output |
| 14 Batch Scoring | Deployable pipeline | PoC has no production path |
| 15 Final Demo | CTO sign-off | PoC cannot be presented |

---

## Mentor Rules (Active for All Notebooks)

1. You write all code. Mentor explains, reviews, and challenges.
2. Answer the "Before You Code" questions in every notebook before writing a single line.
3. Each notebook must be approved before the next one opens.
4. Mentor will ask you to explain your output in domain terms — not just show the number.
5. No skipping notebooks. No jumping ahead.
6. When real CrossBorderPay data arrives, you rebuild from Notebook 03 independently.
