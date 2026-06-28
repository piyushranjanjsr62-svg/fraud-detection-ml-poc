# Cross-Border Transaction Risk Scoring Engine
## Sandbox Infrastructure Design
### Minimum Viable Infrastructure for Phase 1 PoC

**Document Version:** 1.0
**Date:** June 2026
**Status:** Draft — Pending CrossBorderPay CTO Review
**Scope:** Single-contributor · Local machine · Synthetic data only · Batch scoring only

> **Confidential — For Internal Use Only**

---

## Document Control

| Field | Detail |
|---|---|
| Document Title | Sandbox Infrastructure Design — Fraud Detection PoC |
| Version | 1.0 |
| Scope | Phase 1 PoC only — 50,800 transactions, 3 models, batch pipeline |
| Out of Scope | Production infrastructure · Cloud-native services · Distributed systems · Kubernetes · Streaming · Real-time API · Dashboard |
| Placement | Incorporated into Section 12: Sandbox Deployment Plan of the PoC Plan V3.0 |
| Prepared By | PoC Delivery Team |
| Review Required | CrossBorderPay CTO before Week 1 start |

---

## Table of Contents

1. Component Analysis by Architecture Layer
2. Sandbox Infrastructure Design
3. Detailed Infrastructure Architecture
4. Hardware Sizing
5. Software Stack
6. Sandbox Folder Structure
7. Sandbox Deployment Checklist
8. Infrastructure Readiness Checklist
9. Estimated Infrastructure Cost
10. Document Placement Recommendation
11. Open Items Register

---

# Section 1: Component Analysis by Architecture Layer

> **Assessment basis:** 8-layer architecture as defined in `CrossBorderPay_Architecture_V2.xml` and PoC Plan V3.0. Every component is sized only for the approved V3.1 dataset: 50,800 transactions · 500 customers · 780 accounts · 0.5% fraud rate · 254 fraud cases.

---

## Layer 1 — Data Layer

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| Synthetic Dataset Generator | Produces V3.1 CSVs — already executed; dataset exists on disk | Negligible | < 256 MB | < 5 MB output | Python 3.11, Pandas, NumPy |
| customers.csv (500 rows) | Customer master — business type, KYC, turnover | None (static file) | < 1 MB loaded | < 0.1 MB | CSV reader |
| accounts.csv (780 rows) | Account master — type, currency, OD limit | None (static file) | < 1 MB loaded | < 0.1 MB | CSV reader |
| transactions.csv (50,800 rows) | Transaction records with is_fraud label and beneficiary columns | None (static file) | ~25–40 MB loaded in Pandas | ~15 MB on disk | CSV reader |

> **Assumption A-INF-01:** Dataset is already on disk at `e:\project\03_workstreams\ml_fraud_detection\`. No re-generation is needed in Week 1. If any file is found corrupt or missing, re-generation requires a Python script run — estimated < 2 minutes on minimum hardware.

---

## Layer 2 — Data Processing

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| Data Validation (Great Expectations) | Schema checks, null rate checks, referential integrity between CSVs | Low (single-threaded) | ~200 MB | < 10 MB (expectation suite JSON) | great-expectations 0.18.x |
| Data Quality Checks | Duplicate detection, value range checks, timestamp sanity | Low | Included in Pandas load (~40 MB) | Negligible | Pandas 2.x |
| Temporal Split Engine | Orders by submission_timestamp; creates 70/20/10 train/val/test index splits | Low | ~5 MB (index only) | < 20 MB (Parquet split files) | Pandas 2.x, PyArrow |
| Label Leakage Validation | 3-feature LR smoke test; AUC must be < 0.90 — gate before any model training | Low | < 100 MB | Negligible | scikit-learn 1.4.x |

---

## Layer 3 — Feature Engineering

> **This is the most computationally intensive step in the pipeline.** Rolling behavioural window features require `groupby → rolling → shift` operations on 50,800 rows across 500 customers. The feature matrix itself is small; the computation of derived features is the bottleneck.

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| Customer Features (7) | Derived from customers.csv join | Low | < 50 MB | Negligible | Pandas, NumPy |
| Account Features (5) | Derived from accounts.csv join | Low | < 50 MB | Negligible | Pandas, NumPy |
| Transaction Features (8) | Derived from transaction-level fields | Low–Medium | < 100 MB | Negligible | Pandas, NumPy |
| Behavioural Features (11) | Rolling windows: 1h, 24h, 7d, 30d, 90d · velocity · Z-scores · peer percentiles | **Medium–High** (parallelisable with joblib) | **Peak ~500 MB–1 GB** during rolling window computation | < 30 MB (Parquet output) | Pandas 2.x, NumPy, joblib |
| Risk & Regulatory Features (6) | FATF scores, threshold proximity, purpose code deviation | Low | < 50 MB | Negligible | Pandas, NumPy |
| **Feature Matrix (combined)** | 50,800 rows × 37 float64 features | — | **~15 MB in-memory** (50,800 × 37 × 8 bytes) | ~12 MB Parquet | PyArrow |

> **Assumption A-INF-02:** Rolling window features are the peak RAM bottleneck — not the feature matrix itself. The `groupby().rolling()` Pandas operations hold intermediate sorted copies of sub-dataframes in memory. Peak RAM during this step is estimated at 500 MB–1 GB. This is well within the 8 GB minimum configuration.

---

## Layer 4 — Model Development

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| Rule-Based Baseline | 5–7 hardcoded rules; comparator benchmark | Negligible | < 20 MB | < 0.1 MB | Python 3.11 |
| Logistic Regression | sklearn baseline ML; class_weight='balanced' | Low (seconds on 35K rows) | < 200 MB | < 0.1 MB (pickle) | scikit-learn 1.4.x |
| XGBoost Champion | scale_pos_weight ≈ 199; hyperparameter tuning via 5-fold stratified CV | **Medium** (minutes; uses n_jobs=-1 across all available cores) | **Peak ~400–800 MB** during training with early stopping | 2–5 MB (pickle + ONNX) | xgboost 2.0.x |
| Isolation Forest | Unsupervised anomaly; contamination=0.005 | Medium (parallelisable) | ~300 MB | 1–2 MB (pickle) | scikit-learn 1.4.x |
| MLflow Experiment Tracking | Logs hyperparameters, metrics, and artifacts per run | Low | < 100 MB | 50–200 MB (all runs) | mlflow 2.x |

> **Assumption A-INF-03:** XGBoost with n_estimators up to 500, max_depth 6, and 5-fold CV on 35,560 training rows will complete in approximately 3–8 minutes on the minimum configuration and under 2 minutes on the recommended configuration. Actual time depends on hyperparameter search space.

---

## Layer 5 — Evaluation

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| Precision / Recall / F1 / AUC-PR | Standard sklearn metrics on val/test sets | Negligible | < 50 MB | Negligible | scikit-learn 1.4.x |
| Threshold Selection (F1-maximising) | Loop over probability thresholds on validation set | Low | < 50 MB | Negligible | NumPy, scikit-learn |
| Score Calibration (Platt / Isotonic) | Maps raw probabilities to calibrated 0–100 score | Low | < 50 MB | < 0.1 MB (calibration.pkl) | scikit-learn CalibratedClassifierCV |
| Visualisation (PR curves, confusion matrix, SHAP plots) | Charts for notebook and demo | Low | < 200 MB | 5–20 MB (PNG/SVG saves) | matplotlib 3.8.x, seaborn 0.13.x |

---

## Layer 6 — Explainability

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| SHAP TreeExplainer | Exact SHAP values for all 50,800 scored transactions | **Medium** (single-threaded; ~2–5 min on full dataset) | **Peak ~300–600 MB** (SHAP value matrix: 50,800 × 37 × 8 bytes ≈ 14 MB plus model tree traversal overhead) | Negligible (passed directly to scorer) | shap 0.44.x |
| Feature Importance Ranking | `np.argsort(np.abs(shap_values), axis=1)[:, -5:]` — selects top 5 by absolute SHAP | Negligible | < 50 MB | Negligible | NumPy |
| Top-5 Feature Extraction | Writes f1_name through f5_name per output row | Negligible | < 50 MB | Negligible | NumPy, Pandas |

> **Assumption A-INF-04:** SHAP TreeExplainer for XGBoost is exact (not approximate). For 50,800 rows it completes in approximately 2–5 minutes. If this exceeds the 5-minute batch target, switch to `shap.TreeExplainer(model).shap_values(X, approximate=True)` — this reduces runtime by 60–70% with minimal accuracy loss on tree models.

---

## Layer 7 — Batch Scoring

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| batch_scorer.py | Orchestrates: load → validate → feature engineering → infer → calibrate → SHAP → tier → write | Low–Medium (sequential) | ~800 MB peak (overlapping feature eng + SHAP) | Negligible | Python 3.11, all ML libs |
| Risk Scoring Engine | XGBoost predict_proba on feature matrix | Low (inference < 5 sec for 50K rows) | < 200 MB | Negligible | xgboost 2.0.x |
| Risk Tier Assignment | Scalar comparison: 0–39 = GREEN · 40–79 = AMBER · 80–100 = RED | Negligible | Negligible | Negligible | NumPy |

---

## Layer 8 — Output

| Component | Purpose | CPU | RAM | Disk | Software Required |
|---|---|---|---|---|---|
| scored_transactions.csv | 50,800 rows × 21 columns (score + tier + 5×3 SHAP columns) | Negligible | < 50 MB | ~8–12 MB | Pandas |
| audit_log.jsonl | 50,800 JSONL records — one per transaction; PMLA audit trail | Negligible | < 100 MB (written streaming) | ~25–35 MB | Python json module |
| model_card.md | Complete model documentation — constraints, metrics, limitations | Negligible | Negligible | < 0.1 MB | Any text editor |
| regulatory_memo.md | RBI/PMLA/FEMA positioning statement (pending legal sign-off) | Negligible | Negligible | < 0.1 MB | Any text editor |

---

### Layer-by-Layer Summary

| Layer | Peak RAM | Disk Output | Runtime Estimate (Recommended Config) | Bottleneck? |
|---|---|---|---|---|
| 1 — Data Layer | 40 MB | 15 MB (input CSVs) | < 1 sec | No |
| 2 — Data Processing | 200 MB | 20 MB | ~30 sec | No |
| 3 — Feature Engineering | **500 MB–1 GB** | 50 MB | ~3–5 min | **Yes — rolling windows** |
| 4 — Model Development | 800 MB | 10 MB | ~5–8 min | **Yes — XGBoost CV** |
| 5 — Evaluation | 200 MB | 20 MB | ~1 min | No |
| 6 — Explainability | **600 MB** | Negligible | ~2–3 min | **Yes — SHAP computation** |
| 7 — Batch Scoring | 800 MB (overlapping L3+L6) | 50 MB | ~3–5 min | No |
| 8 — Output | 50 MB | 50 MB | < 30 sec | No |
| **Total (sequential)** | **~1 GB peak** | **~215 MB total** | **~15–25 min** | **Manageable on 8 GB laptop** |

---

# Section 2: Sandbox Infrastructure Design

## 2.1 Overview

The CrossBorderPay Fraud PoC is a single-contributor, notebook-driven, batch-only ML experiment running entirely on a local developer machine. No cloud services, no containers, no distributed compute, and no external databases are required.

The V3.1 synthetic dataset is small enough (50,800 rows, ~15 MB on disk) that all processing — from data validation through SHAP computation — completes within the memory and compute budget of a modern laptop. The total peak RAM requirement across the entire pipeline is approximately 1 GB. Any machine with 8 GB RAM and 4 CPU cores can execute the full pipeline.

The infrastructure exists to solve three problems only:

| Problem | Solution |
|---|---|
| **Reproducibility** | Anyone re-runs the pipeline and gets identical results | Fixed random seeds (42) · Pinned library versions in environment.yml · MLflow run tracking |
| **Organisation** | Structured folder tree with version-controlled code that a CTO can navigate | Numbered notebooks · importable Python modules · Git repository |
| **Demo reliability** | Pipeline runs without failure during the Week 5 live presentation | Pre-run backup CSV · 3-pass demo rehearsal · Static fallback |

---

## 2.2 Infrastructure Objectives

| Objective | How It Is Achieved |
|---|---|
| Full reproducibility | Fixed random seeds; pinned library versions; conda environment YAML; MLflow experiment tracking |
| Demo reliability | Pre-run scored_transactions.csv saved to artifacts/demo/ as guaranteed backup |
| PMLA audit compliance | audit_log.jsonl written per batch run; every scored row logged before CSV write |
| Data governance | Raw CSVs treated as read-only; all processing writes to data/processed/; no in-place file modification |
| Version control | Git repository; .gitignore excludes data/raw/, model pickles, .env files, mlruns/ |
| Zero external dependencies | No cloud APIs, no database connections, no network calls at pipeline runtime |
| Single-environment execution | One conda environment (CrossBorderPay-poc) runs all notebooks and scripts |

---

## 2.3 Environment Assumptions

| # | Assumption | Impact If Wrong |
|---|---|---|
| A-INF-01 | V3.1 dataset (3 CSVs) is already on disk — no re-generation needed in Week 1 | If files are missing, re-generation adds ≤ 1 day |
| A-INF-02 | The developer machine runs Windows 10/11, macOS 12+, or Ubuntu 20.04+ | All libraries are cross-platform; no OS-specific risk |
| A-INF-03 | Internet access is available during environment setup for package downloads | After initial setup, all pipeline steps run fully offline |
| A-INF-04 | Git is installed and the developer has a remote account for version control | See OI-INF-01 below |
| A-INF-05 | The developer machine has a minimum of 8 GB RAM free after OS overhead | Peak ML pipeline RAM is ~1 GB; 8 GB free leaves a comfortable margin |
| A-INF-06 | Draw.io Desktop is installed for architecture diagram editing | Alternatively app.diagrams.net (browser-based) works with no installation |

---

## 2.4 Execution Environment

| Attribute | Value |
|---|---|
| Execution mode | Interactive (Jupyter Notebooks) + Script (batch_scorer.py CLI) |
| Kernel | Python 3.11 via Anaconda conda environment named `CrossBorderPay-poc` |
| Notebook server | JupyterLab 4.x — local server, browser access at localhost:8888 |
| Script runner | `python src/scoring/batch_scorer.py` from activated conda environment |
| IDE | VS Code with Python + Jupyter extensions (recommended); any editor is acceptable |
| Claude Code | Invoked inside the project directory; uses the same conda Python interpreter |
| Terminal | Windows Terminal / PowerShell / bash — both supported |

---

## 2.5 Development Environment

| Stage | Tool | Purpose |
|---|---|---|
| EDA and exploration | Jupyter Notebooks (01–06) | Interactive, visual, iterative analysis |
| Feature engineering code | Python modules in src/features/ | Importable, testable, reusable across notebooks |
| Model training | Notebooks + MLflow | Track all hyperparameters, metrics, and artifacts per run |
| Batch scoring | src/scoring/batch_scorer.py | CLI-executable pipeline for demo and integration testing |
| Unit tests | tests/unit/ — pytest | Verify each feature module produces correct output |
| Adversarial tests | tests/adversarial/ — pytest | Verify model handles 3 hard-case scenarios correctly |

---

## 2.6 Testing Environment

| Test Type | Scope | Tool | When Run |
|---|---|---|---|
| Great Expectations suite | V3.1 schema, null rates, referential integrity | great-expectations | Week 1 — before any model training |
| Label leakage smoke test | 3-feature LR AUC < 0.90 gate | scikit-learn | Week 1 — immediately after GE suite passes |
| Unit tests (features) | Each of 5 feature category modules | pytest | Week 2 — during feature development |
| Adversarial test cases | 3 hardcoded scenarios with expected score ranges | pytest + batch_scorer.py | Week 4 |
| End-to-end integration test | Full pipeline CSV-in → CSV-out on V3.1 (50,800 rows) | Manual run + row count assertion | Week 5 — before demo day |
| Reproducibility check | Same seed → identical scored_transactions.csv output | `diff` / pandas DataFrame comparison | Week 5 — before demo day |

---

## 2.7 Demo Environment

| Item | Primary Setup | Backup |
|---|---|---|
| Demo machine | Developer's local laptop — the same machine used for development | None needed |
| Demo dataset | 30 pre-selected transactions covering 4 scenarios (Section 18 of PoC Plan) | Pre-scored backup CSV in artifacts/demo/ |
| Pipeline run | batch_scorer.py run live on 30-transaction demo set | Pre-run scored CSV displayed if live run fails |
| Presentation | 10-slide deck displayed separately from notebook | PDF backup if PPTX fails to open |
| Jupyter display | JupyterLab opened before demo; relevant notebook pre-executed and outputs visible | Static HTML export of notebooks as secondary backup |
| Network dependency | None — no external calls at demo runtime | N/A |

---

# Section 3: Detailed Infrastructure Architecture

```
Developer Laptop (Single Machine — All Components Co-located)
│
├── Anaconda Distribution
│   └── conda env: CrossBorderPay-poc (Python 3.11, pinned packages)
│       ├── Jupyter Runtime     → localhost:8888 (JupyterLab)
│       ├── Python Runtime      → 3.11.x interpreter
│       ├── ML Libraries        → pandas, numpy, sklearn, xgboost, shap, mlflow
│       └── Dev Tools           → pytest, great_expectations, black, isort
│
├── Project Root: CrossBorderPay-fraud-poc/
│   ├── data/
│   │   ├── raw/                → Read-only V3.1 CSVs (source of truth; never modified)
│   │   ├── processed/          → Feature-engineered Parquet files (reproducible)
│   │   └── output/             → scored_transactions.csv + audit_log.jsonl
│   ├── models/                 → Trained model artifacts (pickle + ONNX + calibration)
│   ├── notebooks/              → 6 numbered Jupyter notebooks
│   ├── src/                    → Importable Python modules
│   ├── tests/                  → pytest test suite
│   ├── docs/                   → model_card.md, data_card.md, regulatory_memo.md
│   ├── mlruns/                 → MLflow artifact store (local filesystem, not Git-tracked)
│   ├── logs/                   → Pipeline run logs (JSONL, not Git-tracked)
│   ├── config/                 → environment.yml, .env.example
│   └── artifacts/              → Demo CSV, demo scripts, presentation backup (Git-tracked)
│
├── Model Storage   → models/ directory on local filesystem
│   ├── xgb_model.pkl           → XGBoost champion (2–5 MB)
│   ├── lr_model.pkl            → Logistic Regression (< 0.1 MB)
│   ├── if_model.pkl            → Isolation Forest (1–2 MB)
│   └── calibration.pkl         → Score calibration transformer (< 0.1 MB)
│
├── Dataset Storage → data/ directory on local filesystem
│   ├── raw/                    → ~15 MB total (3 CSVs — read-only)
│   └── processed/              → ~50 MB total (Parquet splits + feature matrix)
│
├── Output Storage  → data/output/ directory
│   ├── scored_transactions.csv → ~10 MB (50,800 rows × 21 columns)
│   └── audit_log.jsonl         → ~30 MB (50,800 JSONL records)
│
├── Documentation   → docs/ directory
│   ├── model_card.md
│   ├── data_card.md
│   └── regulatory_memo.md
│
├── Version Control → Git (local repository + remote push)
│   ├── Tracked:   src/ · notebooks/ · tests/ · docs/ · config/ · artifacts/
│   └── Ignored:   data/raw/ · data/output/ · models/*.pkl · mlruns/ · .env · __pycache__/
│
└── Backup Strategy
    ├── Primary:   Git push to remote after each work session
    │              (code + notebooks + config — everything reproducible)
    ├── Secondary: Copy of data/raw/ CSVs to a separate folder or USB drive
    │              (raw data is the only non-reproducible asset)
    └── Demo:      Pre-run scored_transactions.csv committed to artifacts/demo/
                   as a guaranteed fallback before the Week 5 presentation
```

---

# Section 4: Hardware Sizing

## 4.1 Minimum Configuration

| Attribute | Specification |
|---|---|
| CPU | Intel Core i5 (8th generation or newer) or AMD Ryzen 5 — 4 physical cores |
| RAM | 8 GB |
| Disk | 50 GB free SSD space |
| Operating System | Windows 10 64-bit · macOS 12 Monterey · Ubuntu 20.04 LTS |

**Estimated pipeline completion times on minimum configuration:**

| Step | Estimated Duration |
|---|---|
| Data validation (Great Expectations) | ~2 minutes |
| Feature engineering (rolling windows) | ~10–15 minutes |
| XGBoost training + CV | ~8–12 minutes |
| SHAP computation (50,800 rows) | ~5–8 minutes |
| Batch scoring pipeline (end-to-end) | ~25–35 minutes total |

> **Note:** Minimum configuration is functional but slow. Not recommended as the demo machine. Behavioural rolling-window features and SHAP computation are the bottlenecks. Full pipeline completes but requires patience.

---

## 4.2 Recommended Configuration

| Attribute | Specification |
|---|---|
| CPU | Intel Core i7 (10th generation or newer) or AMD Ryzen 7 — 8 cores / 16 threads |
| RAM | 16 GB |
| Disk | 100 GB free SSD space |
| Operating System | Windows 11 64-bit · macOS 13 Ventura · Ubuntu 22.04 LTS |

**Estimated pipeline completion times on recommended configuration:**

| Step | Estimated Duration |
|---|---|
| Data validation (Great Expectations) | ~30 seconds |
| Feature engineering (rolling windows) | ~3–5 minutes |
| XGBoost training + CV | ~3–5 minutes |
| SHAP computation (50,800 rows) | ~2–3 minutes |
| Batch scoring pipeline (end-to-end) | ~10–15 minutes total |

> **This is the target PoC configuration.** Comfortable for development, fast enough for iteration, appropriate for demo day.

---

## 4.3 Ideal Configuration

| Attribute | Specification |
|---|---|
| CPU | Intel Core i9 (12th generation or newer) or AMD Ryzen 9 — 12+ cores / 24 threads |
| RAM | 32 GB |
| Disk | 256 GB free SSD (NVMe preferred) |
| Operating System | Windows 11 64-bit · Ubuntu 22.04 LTS |

**Estimated pipeline completion times on ideal configuration:**

| Step | Estimated Duration |
|---|---|
| Data validation (Great Expectations) | ~15 seconds |
| Feature engineering (rolling windows) | ~1–2 minutes |
| XGBoost training + CV | ~1–2 minutes |
| SHAP computation (50,800 rows) | ~1 minute |
| Batch scoring pipeline (end-to-end) | ~5 minutes total |

> **Future-proof configuration.** If the PoC is approved and extended to 150,000+ transactions in Phase 2, this machine handles it without modification. Recommended if the developer plans to use this machine through Phase 2.

---

## 4.4 Configuration Comparison Summary

| Attribute | Minimum | Recommended | Ideal |
|---|---|---|---|
| CPU cores | 4 physical | 8 cores / 16 threads | 12+ cores / 24 threads |
| RAM | 8 GB | 16 GB | 32 GB |
| Disk | 50 GB SSD | 100 GB SSD | 256 GB NVMe SSD |
| End-to-end pipeline runtime | 25–35 min | 10–15 min | ~5 min |
| Demo suitability | Marginal | Good | Excellent |
| Phase 2 readiness (150K rows) | No | Borderline | Yes |

---

# Section 5: Software Stack

## 5.1 Core Environment

| Software | Required Version | Purpose | Source |
|---|---|---|---|
| Operating System | Windows 10/11 · macOS 12+ · Ubuntu 20.04+ | Host environment | — |
| Anaconda Distribution | 2024.x (Individual Edition) | Package and environment management | anaconda.com |
| Python | **3.11.x** (managed via conda) | Runtime interpreter | Bundled with Anaconda |
| JupyterLab | **4.1.x** | Interactive notebook server at localhost:8888 | `conda install -c conda-forge jupyterlab` |
| Git | **2.44.x** | Version control for all code, notebooks, and configs | git-scm.com |
| VS Code | Latest stable | IDE with Python and Jupyter extensions (optional but recommended) | code.visualstudio.com |
| Draw.io Desktop | Latest stable | Architecture diagram editing for CrossBorderPay_Architecture_V2.xml | app.diagrams.net |

---

## 5.2 Core ML Libraries

| Library | Required Version | Purpose | Install Command |
|---|---|---|---|
| pandas | **2.1.x** | Data loading, manipulation, rolling window feature computation | `conda install pandas=2.1` |
| numpy | **1.26.x** | Numerical operations, SHAP array handling | `conda install numpy=1.26` |
| scikit-learn | **1.4.x** | Logistic Regression, Isolation Forest, CV, metrics, calibration | `conda install scikit-learn=1.4` |
| xgboost | **2.0.x** | Champion ML model — scale_pos_weight=199 | `conda install -c conda-forge xgboost=2.0` |
| shap | **0.44.x** | SHAP TreeExplainer — top-5 features per transaction | `pip install shap==0.44.1` |
| imbalanced-learn | **0.12.x** | SMOTE — conditional use only if AUC-PR < 0.65 | `pip install imbalanced-learn==0.12.0` |
| mlflow | **2.10.x** | Experiment tracking, model registry, artifact storage | `pip install mlflow==2.10.0` |
| great-expectations | **0.18.x** | Data validation suite — schema, null rates, integrity | `pip install great-expectations==0.18.19` |
| pyarrow | **14.x** | Parquet file read/write for feature matrix storage | `conda install -c conda-forge pyarrow=14` |
| joblib | **1.3.x** | Parallel jobs and model serialization backup | Bundled with scikit-learn |
| scipy | **1.12.x** | Calibration functions and statistical utilities | `conda install scipy=1.12` |

---

## 5.3 Visualisation and Reporting

| Library | Required Version | Purpose | Install Command |
|---|---|---|---|
| matplotlib | **3.8.x** | All charts — PR curves, confusion matrix, score histograms | `conda install matplotlib=3.8` |
| seaborn | **0.13.x** | Statistical visualisation — score distributions, heatmaps | `conda install seaborn=0.13` |
| plotly | **5.18.x** (optional) | Interactive charts in JupyterLab — useful for demo | `pip install plotly==5.18.0` |

---

## 5.4 Development and Testing

| Library | Required Version | Purpose | Install Command |
|---|---|---|---|
| pytest | **8.x** | Unit test and adversarial test runner | `pip install pytest==8.0.0` |
| black | **24.x** | Code formatter — enforces consistent style | `pip install black==24.0.0` |
| isort | **5.13.x** | Import sorter | `pip install isort==5.13.2` |
| python-dotenv | **1.0.x** | Load environment variables from .env file | `pip install python-dotenv==1.0.0` |

---

## 5.5 Conda Environment File

Save as `config/environment.yml`. This is the single source of truth for the development environment. Anyone with Anaconda installed can recreate the exact environment from this file.

```yaml
name: CrossBorderPay-poc
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.11
  - pandas=2.1
  - numpy=1.26
  - scikit-learn=1.4
  - matplotlib=3.8
  - seaborn=0.13
  - jupyterlab=4.1
  - pyarrow=14
  - scipy=1.12
  - pip
  - pip:
    - xgboost==2.0.3
    - shap==0.44.1
    - mlflow==2.10.0
    - great-expectations==0.18.19
    - imbalanced-learn==0.12.0
    - pytest==8.0.0
    - python-dotenv==1.0.0
    - black==24.0.0
    - isort==5.13.2
    - plotly==5.18.0
```

**Setup commands:**

```bash
# Create environment from file
conda env create -f config/environment.yml

# Activate environment
conda activate CrossBorderPay-poc

# Verify Python version
python --version   # Should return Python 3.11.x

# Export pinned lockfile after first successful setup
conda env export > config/environment_lock.yml
```

---

# Section 6: Sandbox Folder Structure

```
CrossBorderPay-fraud-poc/
│
├── README.md                               # Project overview; how to set up env and run pipeline
│
├── config/
│   ├── environment.yml                     # Conda environment definition — single source of truth
│   ├── environment_lock.yml                # Pinned lockfile exported after first successful setup
│   ├── .env.example                        # Environment variable template (no secrets — safe to commit)
│   └── great_expectations/
│       └── expectations/
│           └── CrossBorderPay_v31_suite.json    # GE expectation suite for V3.1 dataset
│
├── data/
│   ├── raw/                                # V3.1 APPROVED DATASET — READ ONLY — NEVER MODIFIED
│   │   ├── CrossBorderPay_customers.csv         # 500 rows
│   │   ├── CrossBorderPay_accounts.csv          # 780 rows
│   │   └── CrossBorderPay_transactions.csv      # 50,800 rows (is_fraud + beneficiary columns embedded)
│   ├── processed/                          # Output of feature engineering — reproducible from code
│   │   ├── feature_matrix_train.parquet    # ~35,560 rows × 37 features
│   │   ├── feature_matrix_val.parquet      # ~10,160 rows × 37 features
│   │   ├── feature_matrix_test.parquet     # ~5,080 rows × 37 features
│   │   └── feature_matrix_full.parquet     # 50,800 rows × 37 features (for batch scoring)
│   └── output/                             # Pipeline outputs — reproducible from code + models
│       ├── scored_transactions.csv         # 50,800 rows × 21 columns
│       └── audit_log.jsonl                 # 50,800 JSONL records (PMLA audit trail)
│
├── models/                                 # Trained model artifacts — EXCLUDED FROM GIT
│   ├── xgb_model.pkl                       # XGBoost champion model (2–5 MB)
│   ├── lr_model.pkl                        # Logistic Regression baseline (< 0.1 MB)
│   ├── if_model.pkl                        # Isolation Forest anomaly model (1–2 MB)
│   └── calibration.pkl                     # Score calibration transformer (< 0.1 MB)
│
├── notebooks/                              # Executed in numbered order — Week 1 through Week 4
│   ├── 01_data_validation.ipynb            # GE suite + null checks + label leakage smoke test
│   ├── 02_eda.ipynb                        # Distributions, fraud breakdown, time span, temporal split
│   ├── 03_feature_engineering.ipynb        # Build all 37 features, save Parquet splits
│   ├── 04_model_training.ipynb             # Train LR + XGBoost + IF, MLflow tracking, model comparison
│   ├── 05_model_evaluation.ipynb           # Metrics, PR curves, threshold selection, comparison table
│   └── 06_batch_scoring.ipynb              # Interactive demo notebook — wraps batch_scorer.py
│
├── src/
│   ├── __init__.py
│   ├── data/
│   │   ├── __init__.py
│   │   ├── loader.py                       # load_transactions(), load_customers(), load_accounts()
│   │   └── validator.py                    # run_ge_suite(), label_leakage_check()
│   ├── features/
│   │   ├── __init__.py
│   │   ├── customer_features.py            # 7 customer-level features
│   │   ├── account_features.py             # 5 account-level features
│   │   ├── transaction_features.py         # 8 transaction-level features
│   │   ├── behavioral_features.py          # 11 rolling-window behavioural features (peak bottleneck)
│   │   └── risk_features.py               # 6 risk/regulatory features + FATF lookup
│   ├── models/
│   │   ├── __init__.py
│   │   ├── baseline_rules.py               # Rule-based comparator (5–7 rules)
│   │   ├── logistic_regression.py          # Training wrapper + evaluation
│   │   ├── xgboost_model.py                # Training wrapper + CV + early stopping
│   │   └── isolation_forest.py             # Training wrapper + anomaly score extraction
│   ├── scoring/
│   │   ├── __init__.py
│   │   ├── batch_scorer.py                 # CLI entry point — full pipeline orchestration
│   │   ├── calibration.py                  # Raw probability → calibrated 0–100 integer score
│   │   └── shap_explainer.py              # SHAP TreeExplainer wrapper + top-5 extraction
│   └── utils/
│       ├── __init__.py
│       ├── metrics.py                      # AUC-PR, FDR, threshold sweep helpers
│       └── audit_logger.py                 # Writes per-transaction JSONL audit records
│
├── tests/
│   ├── __init__.py
│   ├── unit/
│   │   ├── test_customer_features.py
│   │   ├── test_behavioral_features.py
│   │   ├── test_risk_features.py
│   │   └── test_calibration.py
│   └── adversarial/
│       ├── test_adversarial_cases.py       # 3 hardcoded scenario score-range assertions
│       └── adversarial_fixtures.py         # 30-transaction demo dataset (hard-coded fixtures)
│
├── docs/
│   ├── model_card.md                       # Complete model documentation
│   ├── data_card.md                        # V3.1 dataset documentation
│   └── regulatory_memo.md                  # RBI/PMLA/FEMA positioning (pending legal sign-off)
│
├── logs/
│   └── pipeline_runs/                      # Per-run batch scoring logs — NOT Git-tracked
│       └── run_YYYYMMDD_HHMMSS.log
│
├── mlruns/                                 # MLflow artifact store — NOT Git-tracked
│
└── artifacts/
    ├── demo/
    │   ├── demo_dataset_30txns.csv          # 30-transaction demo set (Git-tracked)
    │   ├── demo_scored_backup.csv           # Pre-run scored output for demo fallback (Git-tracked)
    │   └── demo_script.md                  # Step-by-step demo narration notes
    ├── diagrams/
    │   └── CrossBorderPay_Architecture_V2.xml   # Draw.io source file
    └── presentation/
        └── CrossBorderPay_FraudPoC_Demo.pdf     # PDF backup of 10-slide deck
```

**Git-tracked vs excluded:**

| Git-tracked | Excluded (.gitignore) |
|---|---|
| src/ · notebooks/ · tests/ · docs/ · config/ · artifacts/ | data/raw/ · data/processed/ · data/output/ |
| README.md · .env.example | models/*.pkl · mlruns/ · logs/ |
| environment.yml · environment_lock.yml | .env · __pycache__/ · .ipynb_checkpoints/ |

---

# Section 7: Sandbox Deployment Checklist

## 7.1 Environment Setup

- [ ] Anaconda installed (version 2024.x or later)
- [ ] `conda env create -f config/environment.yml` executes without errors
- [ ] `conda activate CrossBorderPay-poc` activates successfully
- [ ] `python --version` confirms Python 3.11.x inside activated environment
- [ ] `jupyter lab` launches and opens at localhost:8888 without errors
- [ ] All imports verified in a blank notebook cell: `import pandas, numpy, sklearn, xgboost, shap, mlflow, great_expectations`
- [ ] MLflow UI confirmed: `mlflow ui` launches at localhost:5000
- [ ] Git repository initialised and remote configured (see OI-INF-01)

## 7.2 Library Installation Verification

- [ ] `xgboost.__version__` returns 2.0.x
- [ ] `shap.__version__` returns 0.44.x
- [ ] `sklearn.__version__` returns 1.4.x
- [ ] `mlflow.__version__` returns 2.10.x
- [ ] `great_expectations.__version__` returns 0.18.x
- [ ] `pytest --version` returns 8.x
- [ ] Conda lock file exported: `conda env export > config/environment_lock.yml`

## 7.3 Data Preparation

- [ ] V3.1 CSVs present in data/raw/: customers.csv (500 rows) · accounts.csv (780 rows) · transactions.csv (50,800 rows)
- [ ] Row counts verified after load: `assert len(df_transactions) == 50800`
- [ ] Fraud count verified: `assert df_transactions['is_fraud'].sum() == 254`
- [ ] Great Expectations suite passes — notebook 01 completes without failures
- [ ] Label leakage smoke test passes: 3-feature LR AUC < 0.90
- [ ] Time span (min/max of submission_timestamp) confirmed and documented in notebook 02
- [ ] Temporal split boundaries defined and documented

## 7.4 Feature Engineering

- [ ] All 5 feature modules run without errors
- [ ] Feature matrix shape verified: `assert feature_matrix.shape == (50800, 37)`
- [ ] Zero null values in feature matrix after imputation: `assert feature_matrix.isnull().sum().sum() == 0`
- [ ] Train / val / test Parquet files saved to data/processed/
- [ ] Split sizes verified: ~35,560 train · ~10,160 val · ~5,080 test
- [ ] Fraud cases per split verified: ~178 train · ~51 val · ~25 test
- [ ] Unit tests pass: `pytest tests/unit/ -v` — all green

## 7.5 Model Training

- [ ] Rule-based baseline trained and benchmarked (recall, precision, AUC-PR documented)
- [ ] Logistic Regression trained; MLflow run logged with metrics and parameters
- [ ] XGBoost trained with scale_pos_weight=199; MLflow run logged
- [ ] Isolation Forest trained; MLflow run logged
- [ ] All models compared on validation set; champion (XGBoost) confirmed
- [ ] Model artifacts saved: xgb_model.pkl · lr_model.pkl · if_model.pkl · calibration.pkl
- [ ] Reproducibility check: re-run notebook with same seeds → identical metric values

## 7.6 Model Validation

- [ ] Champion XGBoost AUC-PR on validation set ≥ 0.70 (hard gate)
- [ ] XGBoost recall ≥ 15% better than rule-based baseline (hard gate)
- [ ] Recall at 5% FPR ≥ 75% on validation set
- [ ] Calibration verified: raw probabilities map correctly to 0–100 scale
- [ ] F1-maximising threshold documented and submitted to CrossBorderPay (Open Item OI-08)
- [ ] Final evaluation metrics recorded on held-out test set — run once only, results locked

## 7.7 SHAP Validation

- [ ] SHAP TreeExplainer initialises without errors on loaded XGBoost model
- [ ] SHAP values computed on test set (5,080 rows) — shape verified: (5080, 37)
- [ ] Top-5 feature extraction verified: every row has exactly 5 non-null SHAP feature entries
- [ ] SHAP waterfall chart renders correctly in notebook
- [ ] SHAP contribution directions verified manually on 5 transactions: days_since_beneficiary_first_seen shows positive contribution for new beneficiaries; amount_zscore_90d shows positive contribution for outlier amounts

## 7.8 Batch Scoring Validation

- [ ] `batch_scorer.py` runs end-to-end on full V3.1 dataset without errors
- [ ] Output row count: scored_transactions.csv has exactly 50,800 rows
- [ ] Audit log row count: audit_log.jsonl has exactly 50,800 lines
- [ ] SHAP coverage: zero rows with null f1_name through f5_name
- [ ] Risk tier distribution sanity-checked: majority GREEN (~98%) · minority AMBER + RED (~2%)
- [ ] Reproducibility confirmed: run twice with same seed → bit-for-bit identical CSV
- [ ] Full pipeline completion time recorded and within 30-minute ceiling

## 7.9 Demo Readiness

- [ ] Demo dataset isolated: artifacts/demo/demo_dataset_30txns.csv — 30 transactions, 4 scenarios
- [ ] Batch pipeline run on 30-transaction demo set produces correct scores for all 4 scenarios
- [ ] Pre-run scored output saved as backup: artifacts/demo/demo_scored_backup.csv
- [ ] Demo narration script prepared: artifacts/demo/demo_script.md
- [ ] PDF presentation backup saved: artifacts/presentation/
- [ ] JupyterLab opens cleanly with CrossBorderPay-poc kernel selected
- [ ] Demo rehearsed minimum 3 times end-to-end without error
- [ ] Demo machine plugged into power — not running on battery during presentation

---

# Section 8: Infrastructure Readiness Checklist

```
INFRASTRUCTURE READINESS — GO / NO-GO GATE
============================================

PRE-DEVELOPMENT ENVIRONMENT
□ Python 3.11 installed (via Anaconda)
□ Anaconda installed and conda command available in terminal
□ Git 2.44+ installed and configured (user.name + user.email set)
□ JupyterLab 4.x working at localhost:8888
□ VS Code (or preferred IDE) installed with Python extension

LIBRARY ENVIRONMENT
□ conda env CrossBorderPay-poc created from config/environment.yml
□ All imports verified in blank notebook (pandas, numpy, sklearn, xgboost, shap, mlflow, great_expectations)
□ environment_lock.yml exported with pinned versions

DATA LAYER
□ V3.1 dataset present (3 CSV files in data/raw/)
□ Row counts verified: 500 / 780 / 50,800
□ Fraud count verified: 254 cases (0.5% of 50,800)
□ Great Expectations suite passes (notebook 01 complete)
□ Label leakage smoke test passes (3-feature LR AUC < 0.90)
□ Temporal split boundaries confirmed from submission_timestamp min/max

FEATURE ENGINEERING
□ All 37 features engineered and saved to Parquet (data/processed/)
□ Feature matrix shape confirmed: (50,800, 37)
□ Zero null values in feature matrix after imputation
□ Unit tests passing: pytest tests/unit/ — all green

MODEL TRAINING
□ Rule-based baseline built and benchmarked
□ Logistic Regression trained and logged to MLflow
□ XGBoost trained (scale_pos_weight=199) and logged to MLflow
□ Isolation Forest trained and logged to MLflow
□ Champion model (XGBoost) selected and confirmed
□ All model artifacts saved to models/

MODEL VALIDATION
□ XGBoost AUC-PR ≥ 0.70 on validation set — HARD GATE
□ XGBoost recall ≥ 15% better than rule baseline — HARD GATE
□ Calibration confirmed: raw probability → 0–100 scale correct
□ Final test set evaluation run once; results recorded and locked

SHAP
□ SHAP running without errors on full feature matrix
□ Top-5 features extracted correctly for every transaction
□ SHAP values directionally sensible (manually verified on 5 transactions)

BATCH PIPELINE
□ batch_scorer.py runs end-to-end on 50,800 transactions
□ scored_transactions.csv: exactly 50,800 rows, 21 columns
□ audit_log.jsonl: exactly 50,800 lines (PMLA compliance)
□ SHAP coverage: 100% — no null feature names in any row
□ Pipeline reproducible: same seed → identical output CSV

DEMO ASSETS
□ Demo dataset ready: 30 transactions, 4 scenarios
□ Demo backup CSV pre-generated and saved to artifacts/demo/
□ Demo narration script written and reviewed
□ PDF presentation backup saved to artifacts/presentation/
□ Demo rehearsed minimum 3 times without error
□ Laptop charged and power adapter packed for presentation day
```

---

# Section 9: Estimated Infrastructure Cost

## 9.1 Option A — Local Developer Laptop (Recommended)

| Attribute | Detail |
|---|---|
| Monthly Cost | **₹0 / $0 per month — no additional cost** |
| One-time cost | ₹0 — Anaconda, Python, Git, JupyterLab are free and open source |
| Setup time | 30–60 minutes (environment creation + library installation) |

| Pros | Cons |
|---|---|
| Zero ongoing cost | Performance limited by available hardware |
| Full offline capability — no network dependency at runtime | No remote access by default |
| No data egress risk — V3.1 data never leaves the machine | If laptop fails, environment must be rebuilt (mitigated by Git push + environment.yml) |
| No cloud account setup or IAM complexity | |
| No data residency compliance risk | |
| Instant start — no provisioning wait | |

> **Recommendation: This is the correct choice for this PoC.** 50,800 transactions, 37 features, and 3 models are fully manageable on a modern laptop. There is no technical justification for any cloud spend at this stage.

---

## 9.2 Option B — AWS EC2 (Fallback If Local Machine Is Insufficient)

| Configuration | Instance | vCPU | RAM | On-Demand Cost | Spot Cost |
|---|---|---|---|---|---|
| Minimum | t3.large | 2 | 8 GB | ~$0.10/hr · ~$72/mo | ~$0.03/hr · ~$22/mo |
| Recommended | m5.xlarge | 4 | 16 GB | ~$0.19/hr · ~$137/mo | ~$0.06/hr · ~$43/mo |
| Ideal | m5.2xlarge | 8 | 32 GB | ~$0.38/hr · ~$274/mo | ~$0.11/hr · ~$79/mo |

> **Region:** ap-south-1 (Mumbai) — mandatory for RBI India data residency alignment.

| Pros | Cons |
|---|---|
| Accessible from any machine | Ongoing monthly cost |
| Snapshot-based backup | Requires IAM setup and AWS account |
| Scalable for Phase 2 (150K rows) | V3.1 synthetic data must be uploaded (see OI-INF-02) |
| Spot pricing significantly reduces cost when terminated when idle | Adds network dependency at runtime |

> **Recommendation:** Only justified if the developer's local machine is below minimum specifications (< 8 GB RAM or no SSD). If used, deploy in ap-south-1, use Spot instances to minimise cost, and terminate the instance when not actively running to avoid idle charges.

---

## 9.3 Option C — Azure VM (Not Recommended)

| Configuration | Instance | vCPU | RAM | Monthly Cost |
|---|---|---|---|---|
| Recommended | B4ms | 4 | 16 GB | ~$0.17/hr · ~$122/mo |
| Ideal | D4s v5 | 4 | 16 GB | ~$0.19/hr · ~$137/mo |

| Pros | Cons |
|---|---|
| Enterprise Azure account may already exist | AWS→Azure migration was formally dropped (2026-06-11) |
| Good Jupyter / VS Code integration | CrossBorderPay infrastructure is AWS-only (ap-south-1) |
| | Cross-platform environment adds complexity with no PoC benefit |
| | Higher cost than AWS Spot for equivalent compute |

> **Recommendation: Not recommended.** the platform's confirmed infrastructure direction is AWS ap-south-1. Azure adds no value at the PoC stage and introduces unnecessary vendor complexity.

---

## 9.4 Option D — GCP VM (Not Recommended)

| Configuration | Instance | vCPU | RAM | Monthly Cost |
|---|---|---|---|---|
| Recommended | n2-standard-4 | 4 | 16 GB | ~$0.19/hr · ~$137/mo |

| Pros | Cons |
|---|---|
| Competitive pricing | No existing GCP relationship at CrossBorderPay |
| Good Vertex AI integration for future phases | Adds a new vendor for zero PoC benefit |

> **Recommendation: Not recommended for this PoC.**

---

## 9.5 Cost Summary

| Option | Monthly Cost | Suitability | Recommendation |
|---|---|---|---|
| Local Laptop | ₹0 | Full PoC | **YES — Primary choice** |
| AWS EC2 (Spot, m5.xlarge) | ~$43/mo | Full PoC + Phase 2 ready | Only if local machine < minimum spec |
| Azure VM | ~$122/mo | Full PoC | No — wrong cloud provider |
| GCP VM | ~$137/mo | Full PoC | No — no existing relationship |

---

# Section 10: Document Placement Recommendation

## 10.1 Recommendation

> **Incorporate this content into Section 12: Sandbox Deployment Plan of the PoC Plan V3.0.**

Section 12 already exists in V1.0 and V3.0 with the correct title and intent. Its current subsections (12.1 Infrastructure, 12.2 Folder Structure, 12.3 Monitoring) are the placeholder that this full infrastructure assessment replaces and expands. No new major section is created. No section numbers change.

---

## 10.2 Revised Section 12 Structure

The content of current Sections 12.1, 12.2, and 12.3 is absorbed into the following expanded subsection structure:

| Subsection | Title | Source |
|---|---|---|
| 12.1 | Infrastructure Overview and Objectives | Replaces current 12.1 |
| 12.2 | Environment Assumptions | New in-place addition |
| 12.3 | Execution Environment | New in-place addition |
| 12.4 | Component Analysis by Architecture Layer | New in-place addition |
| 12.5 | Hardware Sizing | New in-place addition |
| 12.6 | Software Stack | New in-place addition |
| 12.7 | Sandbox Folder Structure | Expands and replaces current 12.2 |
| 12.8 | Sandbox Deployment Checklist | New in-place addition |
| 12.9 | Infrastructure Readiness Checklist | New in-place addition |
| 12.10 | Estimated Infrastructure Cost | New in-place addition |
| 12.11 | Monitoring and Logging | Preserves current 12.3 |

This structure is fully compliant with Document Governance Rules: V1.0 section numbering is preserved, no new major section is introduced, and all additions are in-place expansions within the existing Section 12 boundary.

---

# Section 11: Open Items Register

| ID | Question | Priority | Owner | Impact If Unanswered |
|---|---|---|---|---|
| OI-INF-01 | Does the developer have an existing Git remote (GitHub / GitLab / Bitbucket) where the codebase should live? What is the repository URL? | P2 | Developer / CTO | Remote backup strategy undefined; version control remains local-only with no offsite copy |
| OI-INF-02 | If AWS EC2 is used as a fallback, does uploading the V3.1 synthetic dataset to a cloud instance require any confirmation from CrossBorderPay, even though the data is fully synthetic with no real customer information? | P2 | CrossBorderPay Legal / CTO | Cloud option cannot be exercised until confirmed; local laptop must be used in the interim |
| OI-INF-03 | What is the developer's actual machine specification (CPU model, core count, RAM, disk type and free space)? | **P1** | Developer | Cannot confirm whether minimum, recommended, or ideal configuration applies; cannot set demo machine expectations or decide if cloud fallback is needed |

> **OI-INF-03 is the highest priority of the three.** It should be answered before Week 0 ends. The developer's actual machine spec determines whether the local laptop is viable or whether cloud fallback is required.

---

## Constraint Separation — PoC vs Production

The table below explicitly separates what is in scope for this PoC from what belongs to future phases. This boundary is enforced by Data Governance Rules and Document Governance Rules.

| Infrastructure Component | PoC (Phase 1) | Production (Phase 3+) |
|---|---|---|
| Compute | Local developer laptop | AWS ECS/Fargate on ap-south-1 |
| Data storage | Local filesystem (CSV + Parquet) | S3 + DynamoDB |
| Model storage | Local pickle files | S3 model registry + MLflow tracking server |
| Feature computation | Batch (pre-computed Parquet) | Real-time Redis feature store |
| Scoring mode | Batch CSV → CSV | Real-time FastAPI endpoint (< 100ms) |
| Experiment tracking | Local MLflow (localhost:5000) | Managed MLflow or SageMaker Experiments |
| Containerisation | Dockerfile present but optional | Mandatory — ECS task definition |
| Orchestration | Manual notebook execution | Airflow or Step Functions |
| Monitoring | Log files + manual checks | Prometheus + Grafana + EvidentlyAI |
| Security | .env for local secrets | AWS Secrets Manager + IAM + VPC |
| CI/CD | Git push only | GitHub Actions → ECR → ECS |

---

*End of Document*

*Cross-Border Transaction Risk Scoring Engine — Sandbox Infrastructure Design V1.0*
*June 2026 · Prepared by PoC Delivery Team · For CrossBorderPay CTO Review*
* | cross-border payments company | Bengaluru*
