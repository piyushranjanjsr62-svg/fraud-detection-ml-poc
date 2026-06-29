# Cross-Border Transaction Risk Scoring Engine — PoC

**Company:** a cross-border payments platform
**Project:** ML-based Fraud Detection PoC — Phase 1
**Dataset:** Synthetic V3.1 (50,800 transactions · 500 customers · 780 accounts)
**Status:** Week 1 — EDA Complete

---

## Objective

Build an interpretable, explainable ML model to score cross-border payment transactions by fraud risk. Replace the current rule-based system with a model that delivers ≥15% recall improvement while maintaining RBI/FEMA compliance and full SHAP-based explainability.

## Models in Scope

| Model | Role |
|---|---|
| Rule-Based Baseline | Comparator — benchmark before ML |
| Logistic Regression | Interpretable ML baseline |
| XGBoost | Champion candidate A |
| LightGBM | Champion candidate B |
| Isolation Forest | Anomaly detection layer |

## Success Criteria

| Metric | Threshold |
|---|---|
| AUC-PR | ≥ 0.72 |
| Recall at 5% FPR | ≥ 75% |
| Precision | ≥ 65% |
| F1 | ≥ 0.68 |
| ML vs Rule baseline recall improvement | ≥ 15% |

## Repository Structure

```
├── config/             Project configuration (paths, params, thresholds)
├── data/
│   ├── raw/            V3.1 synthetic dataset (not committed to git)
│   ├── processed/      Cleaned data and feature matrix
│   └── output/         Scored transactions, audit logs
├── notebooks/          One notebook per lifecycle stage
├── src/
│   ├── data/           Data loading and cleaning utilities
│   ├── features/       Feature engineering functions
│   ├── models/         Rule baseline and ML model wrappers
│   ├── scoring/        Batch scoring pipeline
│   └── explainability/ SHAP utilities
├── models/             Saved model artifacts (.pkl, .joblib)
├── reports/            Evaluation reports, governance tracker
├── docs/               Learning roadmap, architecture notes
└── tests/              Unit tests for src/ modules
```

## Notebook Sequence

| # | Notebook | Purpose | Status |
|---|---|---|---|
| 01 | `01_eda.ipynb` | Exploratory Data Analysis + Leakage Gate | ✅ Complete |
| 02 | `02_rule_baseline.ipynb` | Rule-based comparator (mandatory before ML) | ✅ Complete |
| 03 | `03_data_cleaning.ipynb` | Missing value treatment, dtype fixes, deduplication | ✅ Complete |
| 04 | `04_feature_engineering.ipynb` | Build 30+ features across 5 categories | ✅ Complete |
| 05 | `05_feature_validation.ipynb` | Variance, correlation, leakage, importance check | ✅ Complete |
| 06 | `06_train_test_split.ipynb` | Temporal 70/20/10 split + stratification | ✅ Complete |
| 07 | `07_logistic_regression.ipynb` | Interpretable ML baseline | ✅ Complete |
| 08 | `08_xgboost.ipynb` | Champion candidate A | ✅ Complete |
| 09 | `09_lightgbm.ipynb` | Champion candidate B | ❌ Not Started |
| 10 | `10_isolation_forest.ipynb` | Anomaly detection layer | ❌ Not Started |
| 11 | `11_model_evaluation.ipynb` | Champion selection + metric comparison | ❌ Not Started |
| 12 | `12_threshold_optimization.ipynb` | Precision/recall tradeoff + threshold tuning | ❌ Not Started |
| 13 | `13_shap_explainability.ipynb` | SHAP values + top-5 feature explanations | ❌ Not Started |
| 14 | `14_batch_scoring.ipynb` | End-to-end scoring pipeline (CSV in → CSV out) | ❌ Not Started |
| 15 | `15_final_demo.ipynb` | PoC demo notebook — clean, presentation-ready | ❌ Not Started |

## Compliance Notes

- All data is synthetic — no real customer PII
- Feature design must exclude post-event columns (fraud_type, sanction_list_source)
- SHAP output format: Feature Name · Feature Value · Contribution Score (no reason code narratives in PoC Phase 1)
- Audit log: JSONL format for PMLA compliance
- RBI mandate: failed transaction auto-reversal within T+1

## Environment

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```
