# CrossBorderPay Fraud PoC — Project Governance Tracker
**Last Updated:** 2026-06-26
**Owner:** Piyush Ranjan
**Mentor Review:** Required before each notebook is marked Approved

---

## Notebook Progress Tracker

| # | Notebook | Purpose | Status | Dependencies | Est. Time | Current Stage | Review Status | Definition of Done | Approval | Completion % |
|---|---|---|---|---|---|---|---|---|---|---|
| 01 | `01_eda.ipynb` | EDA + Leakage Gate | 🔄 In Progress | V3.1 dataset | 4h | Cells 8,23,25,27,29 need re-run | Pending | Leakage gate PASS + all cells executed | ❌ | 70% |
| 02 | `02_rule_baseline.ipynb` | Rule-based comparator | ❌ Not Started | 01 approved | 3h | Not started | Not Started | Precision, recall, F1 calculated manually | ❌ | 0% |
| 03 | `03_data_cleaning.ipynb` | Missing values + dtype fixes | ❌ Not Started | 01 approved | 3h | Not started | Not Started | Clean parquet saved, fraud rate unchanged | ❌ | 0% |
| 04 | `04_feature_engineering.ipynb` | Build 30+ features | ❌ Not Started | 03 approved | 6h | Not started | Not Started | Feature matrix parquet, 30+ features, no NaN | ❌ | 0% |
| 05 | `05_feature_validation.ipynb` | Variance, correlation, leakage | ❌ Not Started | 04 approved | 3h | Not started | Not Started | Validated feature list saved | ❌ | 0% |
| 06 | `06_train_test_split.ipynb` | Temporal 70/20/10 split | ❌ Not Started | 05 approved | 2h | Not started | Not Started | 6 partition files, zero date overlap | ❌ | 0% |
| 07 | `07_logistic_regression.ipynb` | ML baseline | ❌ Not Started | 06 approved | 4h | Not started | Not Started | AUC-PR >= 0.60, model saved | ❌ | 0% |
| 08 | `08_xgboost.ipynb` | Champion candidate A | ❌ Not Started | 07 approved | 4h | Not started | Not Started | AUC-PR >= 0.72, model saved | ❌ | 0% |
| 09 | `09_lightgbm.ipynb` | Champion candidate B | ❌ Not Started | 08 approved | 4h | Not started | Not Started | Champion selected with justification | ❌ | 0% |
| 10 | `10_isolation_forest.ipynb` | Anomaly detection layer | ❌ Not Started | 09 approved | 3h | Not started | Not Started | Combined scoring evaluated | ❌ | 0% |
| 11 | `11_model_evaluation.ipynb` | Final champion selection | ❌ Not Started | 10 approved | 3h | Not started | Not Started | All success criteria evaluated on test set | ❌ | 0% |
| 12 | `12_threshold_optimization.ipynb` | Precision/recall tradeoff | ❌ Not Started | 11 approved | 2h | Not started | Not Started | Threshold saved to config/ | ❌ | 0% |
| 13 | `13_shap_explainability.ipynb` | SHAP top-5 features | ❌ Not Started | 12 approved | 4h | Not started | Not Started | shap_explanations.parquet generated | ❌ | 0% |
| 14 | `14_batch_scoring.ipynb` | End-to-end pipeline | ❌ Not Started | 13 approved | 4h | Not started | Not Started | Pipeline runs CSV-in CSV-out | ❌ | 0% |
| 15 | `15_final_demo.ipynb` | CTO demo notebook | ❌ Not Started | 14 approved | 3h | Not started | Not Started | Runs < 5 min, no errors | ❌ | 0% |

**Total Estimated Effort: ~52 hours**
**Elapsed: ~4 hours**
**Overall Completion: ~12%**

---

## Success Criteria Gate (to be filled during Notebook 11)

| Metric | Required | Rule Baseline | LR | XGBoost | LightGBM | Champion |
|---|---|---|---|---|---|---|
| AUC-PR | >= 0.72 | TBD | TBD | TBD | TBD | TBD |
| Recall at 5% FPR | >= 75% | TBD | TBD | TBD | TBD | TBD |
| Precision | >= 65% | TBD | TBD | TBD | TBD | TBD |
| F1 | >= 0.68 | TBD | TBD | TBD | TBD | TBD |
| ML vs Rule recall improvement | >= 15% | — | TBD | TBD | TBD | TBD |
| SHAP top-5 coverage | 100% | — | — | — | — | TBD |
| Reproducibility (seed=42) | 100% | — | — | — | — | TBD |

---

## Open Items

| ID | Item | Owner | Priority | Status |
|---|---|---|---|---|
| OI-01 | Real fraud case count from CrossBorderPay | CrossBorderPay CTO | P1 | Open |
| OI-03 | Current rule baseline metrics | CrossBorderPay Head of Risk | P1 | Open |
| OI-07 | Primary fraud typology by loss cost | CrossBorderPay Head of Risk | P1 | Open |
| OI-08 | BIOS VT-x enable for Docker (Week 5) | Piyush | P2 | Open |
| OI-09 | PMLA STR 7-day deadline verification | Compliance | P2 | Open |

---

## Scope Decisions (Locked)

| Decision | Outcome | Session |
|---|---|---|
| Random Forest | REMOVED from scope | Session 9 |
| Real-time API | DEFERRED to Phase 3 | Session 9 |
| Dashboard | DEFERRED to Phase 3 | Session 9 |
| Docker | Week 5 only (BIOS required) | Session 10 |
| Train/val/test split | Temporal only — no random split | Session 8 |
| Explainability | SHAP top-5 only — no reason code narratives in PoC | Session 9 |
| Champion selection | XGBoost vs LightGBM by validation AUC-PR | Session 10 |
