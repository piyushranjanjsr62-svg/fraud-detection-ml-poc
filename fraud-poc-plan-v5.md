# Cross-Border Transaction Risk Scoring Engine
## Foundational Fraud Risk Scoring Capability — Proof of Concept Plan
### Version 5.0 | June 2026

> **Confidential — For CEO & CTO Review Only**

---

## Document Control

| Field | Detail |
|---|---|
| Document Title | Transaction Risk Scoring Engine — Proof of Concept Plan |
| Version | 5.0 |
| Status | Draft — Pending CrossBorderPay Approval |
| Prepared By | PoC Joint Delivery Team |
| Date | June 2026 |
| Review Date | TBD — Pending Open Questions Resolution (Section 16) |
| Governing Review | Internal Committee Review (June 2026) |
| Next Milestone | Week 0 Open Question Resolution with CrossBorderPay |

> ⚠️ This is a Proof of Concept document. No engineering begins until P1 Open Items (OI-01, OI-03, OI-07) are resolved with CrossBorderPay. See Section 16.

---

## Table of Contents

1. Executive Summary
2. Business Context
3. PoC Scope
4. Solution Architecture
5. Data Strategy
6. Feature Engineering Plan
7. Model Development Plan
8. Model Evaluation Framework
9. Explainability Framework
10. Batch Scoring Pipeline
11. Dashboard Design
12. Sandbox Deployment Plan
13. Delivery Plan
14. Work Breakdown Structure
15. RAID Log
16. Open Items Register
17. Weekly Status Report Templates
18. Demo Plan
19. Future Roadmap
20. Final Recommendation

---

# Section 1: Executive Summary

## 1.1 Business Problem

CrossBorderPay operates as a cross-border A2A payment platform serving Indian businesses — exporters, importers, IT services firms, EdTech companies, SaaS providers, and SMEs. Every transaction carries regulatory risk under FEMA and PMLA, reputational risk with banking partners and financial risk from fraud typologies specific to the Indian cross-border payments corridor.

CrossBorderPay is building its ML based fraud scoring capability from the ground up. The PoC will establish a foundational capability — not replace an existing one.

## 1.2 Current Challenges

| Challenge | Business Impact |
|---|---|
| No ML-based fraud detection capability | Complex fraud patterns go undetected without an adaptive model |
| No calibrated risk score | Investigation team cannot prioritise review queues by risk severity |
| No explainability for flagged transactions | Inability to respond to client disputes or regulatory queries |
| No behavioural baseline per customer | Unusual-for-this-customer transactions treated same as unusual-for-all |

## 1.3 Proposed Solution

A Machine Learning–based Transaction Risk Scoring Engine that:

- Assigns every cross-border transaction a calibrated risk score from 0 to 100
- Produces the top 5 contributing SHAP/LIME features driving each score (Feature Name, Feature Value, Contribution Score)
- Classifies transactions into three action tiers: Approved (GREEN), Review (AMBER), Reject (RED)
- Learns behavioural baselines per customer, business type, and corridor
- Runs as a batch scoring pipeline (CSV in → CSV out) on the approved synthetic dataset
- Maintains a full audit trail suitable for PMLA and RBI/FIU-IND examination

## 1.4 Expected Outcomes

| Outcome | Target | Measurement |
|---|---|---|
| Fraud Detection Rate | > 80% of injected fraud cases detected | Recall on held-out test set |
| False Positive Rate | < 3% of legitimate transactions flagged | 1 − Precision on negative class |
| Improvement over rule baseline | > 15% lift in fraud detection recall | Baseline vs. ML model comparison |
| SHAP explanation coverage | 100% of scored transactions | Audit log completeness |
| Experiment reproducibility | 100% | Identical output on same seed |

## 1.5 PoC Objectives

1. Validate and profile the existing approved synthetic dataset (50K+ transactions) through EDA and data quality checks
2. Engineer 30+ features across 5 categories: customer, account, transaction, behavioural, risk/regulatory
3. Build and compare multiple ML models (Logistic Regression, XGBoost, LightGBM) against a rule-based baseline
4. Produce a calibrated risk score with top-5 contributing SHAP/LIME features for every transaction
5. Build a batch scoring pipeline (CSV in → scored CSV out) demonstrable on the daily dataset

## 1.6 Success Criteria

### Technical Success Criteria (Hard Gates)

| Criterion | Pass Threshold |
|---|---|
| AUC-PR (Precision-Recall) on held-out test set | ≥ 0.72 |
| Fraud Detection Rate (Recall) at 5% FPR | ≥ 75% |
| Precision at chosen operating threshold | ≥ 65% |
| F1 Score | ≥ 0.68 |
| ML model recall vs. rule-based baseline recall | ≥ 15% improvement |
| SHAP top-5 feature coverage (all scored transactions) | 100% |
| Experiment reproducibility | 100% |

### Business Success Criteria (CEO Gate)

| Criterion | Pass Threshold |
|---|---|
| Batch pipeline runs end-to-end on V3.1 dataset without errors | Pass |
| SHAP/LIME explanations are interpretable by a non-technical fraud analyst | Confirmed in demo |
| Model card is complete and honest about dataset constraints | Signed off |
| Regulatory positioning memo completed | Signed off |

## 1.7 Executive Benefits

### Business
- Demonstrates the platform's risk management sophistication to partners
- Creates a productisable fraud scoring capability — competitive differentiator
- Strengthens the regulatory relationships

### Technical
- Establishes the foundational ML pipeline: data validation → feature engineering → model → explainability → scoring
- Produces a reproducible, version-controlled ML codebase that can be extended to a production API (Phase 3)
- Creates audit-ready output suitable for PMLA compliance review
- Delivers the MLOps foundation for continuous model improvement

### For Partners
- Provides a shared risk intelligence narrative that strengthens the correspondent banking relationship
- Demonstrates that the platform's transaction screening is developing toward bank-grade fraud controls

---

# Section 2: Business Context

## 2.1 Customer Segments (Assumption)

| Segment | Share | Primary Corridor | Typical Amount | Fraud Risk | Key Fraud Typology |
|---|---|---|---|---|---|
| Exporters | 25% | India → USA, UK, UAE | USD 10K–200K | Medium | TBML over-invoicing, phantom shipments |
| Importers | 20% | USA, China, UAE → India | USD 5K–200K | High | Under-invoicing, advance fee fraud |
| IT Services | 20% | India → USA, UK, Singapore | USD 20K–200K | Low–Medium | Salary diversion, ATO |
| EdTech | 15% | India → Global | USD 500–50K | Medium | Refund fraud, fake enrollments |
| SaaS | 10% | India → Global | USD 1K–100K | Low | Purpose code misdeclaration |
| SME | 10% | India → UAE, UK | USD 1K–50K | High | Hawala mirror, structuring |

## 2.2 Fraud Typologies Covered in This PoC

| # | Typology | Description | Detection Signal |
|---|---|---|---|
| 1 | TBML — Over-invoicing | Export invoice value inflated to extract more FX than goods are worth | Invoice amount vs. GST filing, corridor × business type anomaly |
| 2 | TBML — Under-invoicing | Import invoice understated; balance settled via hawala | Amount significantly below peer group median for corridor |
| 3 | Round-tripping | Funds leave India as business payment, return as FDI via offshore SPV | Corridor + purpose code combination |
| 4 | Account Takeover (ATO) | Compromised credentials lead to fraudulent outward remittances | New beneficiary, amount deviation, off-hours submission |
| 5 | Smurfing / Structuring | Large payment broken into sub-threshold transactions | Velocity burst + amount clustering near reporting thresholds |
| 6 | Dormant Account Reactivation | Long-dormant account suddenly active with high-value transaction | Days since last transaction, amount × dormancy interaction |
| 7 | First-Party Fraud (Purpose Code Abuse) | Business misdeclares purpose code to bypass scrutiny | Purpose code deviation from historical pattern |

## 2.3 Assumptions

| # | Assumption | Validation Required |
|---|---|---|
| A1 | Synthetic dataset V3.1 is sufficient for PoC model training and evaluation | Confirmed — no real transaction data access at PoC stage |
| A2 | PoC validated against held-out test set from the same V3.1 generator with temporal split | Architectural — no real fraud cases available |
| A3 | CrossBorderPay may have an existing rule-based fraud detection approach — this is unconfirmed. A synthetic rule baseline is used as the PoC comparator regardless. | Must be confirmed with CrossBorderPay (Open Item OI-03) |
| A4 | Payment rail distribution: SWIFT 60%, Fedwire 15%, CHIPS 10%, SEPA 10%, CHAPS 5% | Must be confirmed with CrossBorderPay (Open Item OI-13) |
| A5 | Approved fraud rate for the V3.1 dataset is 0.5% flat across all business types (254 fraud cases from 50,800 transactions) | Approved — governs all model training and evaluation |
| A6 | Explainability outputs (top-5 SHAP/LIME features) are for internal investigation use only at PoC stage | Not for client-facing use without regulatory review |

---

# Section 3: PoC Scope

## 3.1 In Scope

| # | Item | Description |
|---|---|---|
| 1 | Approved Synthetic Dataset V3.1 | 50,800 transactions · 500 customers · 780 accounts · 0.5% flat fraud rate · 254 fraud cases · 7 fraud typologies documented in Section 2.2. Beneficiary pool embedded as columns in transactions.csv. Dataset already generated and on disk. |
| 2 | Feature Engineering Pipeline | 30+ features across 5 categories: customer, account, transaction, behavioural, risk/regulatory |
| 3 | Rule-Based Baseline System | Minimum viable rule set used as comparison benchmark |
| 4 | ML Model Training | Logistic Regression (baseline ML), XGBoost/LightGBM (champion candidates), Isolation Forest (anomaly layer) |
| 5 | Model Evaluation Framework | AUC-PR, Recall, Precision, F1, FDR, FPR |
| 6 | Calibrated Risk Score (0–100) | Probability-calibrated score mapped to 0–100 scale with three action tiers |
| 7 | Top-5 SHAP/LIME Features per Transaction | SHAP TreeExplainer (primary) / LIME (validation) — top 5 features by absolute contribution. Output: Feature Name, Feature Value, Contribution Score only. |
| 8 | Batch Scoring Pipeline | Python script: CSV in → feature engineering → XGBoost/LightGBM score + SHAP → scored CSV out |
| 9 | Docker Containerisation | Reproducible container build with pinned dependencies |
| 10 | PoC to Production Roadmap | 10-slide deck for CEO/CTO demo |

## 3.2 Feature Classification (MoSCoW)

| Feature | Priority | Rationale |
|---|---|---|
| Transaction risk score (0–100) | **Must Have** | Core deliverable |
| Top-5 SHAP/LIME features per transaction | **Must Have** | PMLA audit requirement + analyst explainability |
| Three-tier action classification (GREEN/AMBER/RED) | **Must Have** | Operational workflow requirement |
| Behavioural deviation features | **Must Have** | Primary ML signal |
| Rule-based baseline comparison | **Must Have** | Cannot prove ML value without it |
| Batch scoring pipeline (CSV in → CSV out) | **Must Have** | Demo requirement |
| Isolation Forest anomaly detection | **Should Have** | Catches fraud types not in training labels |
| Adversarial test cases (3 scenarios) | **Should Have** | Production readiness signal |
| Regulatory positioning memo | **Should Have** | CEO/board requirement |
| Network / graph features | **Won't Have** (Phase 4–5) | Graph infrastructure not available in PoC |
| Analyst review queue dashboard | **Won't Have** (Phase 3) | Deferred |
| Business cost model in rupees | **Won't Have** (Phase 2) | FP costs not validated at PoC stage |
| Real-time scoring API | **Won't Have** (Phase 3–4) | Batch scoring only in Phase 1 |

---

# Section 4: Solution Architecture

## 4.1 Reference Rule-Based Architecture (Baseline Comparator)

> **Note:** The rule-based baseline below represents the comparator being built for this PoC, not a confirmed description of the platform's current system. Whether CrossBorderPay has an existing rule-based fraud detection system is an open item (OI-03).

```
Reference Rule-Based Architecture (Baseline Comparator — Built in Week 1)

Transaction Submitted → Rule Engine (Static Rules) → Threshold Check
  ├── PASS → Payment Processed
  └── FAIL → Manual Review Queue → Analyst Decision
                              ├── Approve → Payment Processed
                              └── Reject → Transaction Blocked → [Manual STR Filing]

Limitations: No learning, no calibration, no explainability beyond rule fired,
no behavioural baseline, no network signals, no audit trail.
```

## 4.2 Target PoC Architecture

```
Layer 1 — Data Input Layer
Existing Synthetic Dataset V3.1 (on disk):
  customers.csv (500) · accounts.csv (780) · transactions.csv (50,800 rows, is_fraud embedded)
→ Data Validation (Great Expectations) → Temporal Split → Train/Val/Test Sets

Layer 2 — Data Processing Layer
Data Cleaning Pipeline → Feature Store (Parquet) → Temporal Split
  Train: ~35,560 txns (~178 fraud) · Val: ~10,160 (~51 fraud) · Test: ~5,080 (~25 fraud)
  [Time span: Pending Confirmation from CrossBorderPay — verify submission_timestamp min/max]

Layer 3 — Feature Engineering Layer
Customer Features | Account Features | Transaction Features
| Behavioural Features | Risk/Regulatory Features
[Total: 30+ features across 5 categories]
[Network/graph features: Deferred to Phase 4–5]

Layer 4 — Model Training Layer
Logistic Regression (Baseline ML) | XGBoost (Champion Candidate, scale_pos_weight≈199)
| LightGBM (Champion Candidate) | Isolation Forest (Anomaly) | Rule-Based (Comparator)

Layer 5 — Model Registry
MLflow Experiment Tracking → Model Artifacts (Pickle + ONNX)
→ Model Card Documentation → Version Control

Layer 6 — Batch Scoring Pipeline
Feature Matrix → Champion Model Inference → Score Calibration (0–100)
→ SHAP/LIME Explainer (Top-5 Features) → Audit Logger
→ scored_transactions.csv

Layer 7 — Dashboard (Phase 3 — Not in PoC Scope)
[Deferred]

Layer 8 — Sandbox Deployment Layer
Docker Container (Batch Pipeline) | Prometheus Metrics (optional) | Logging (JSONL)
```

## 4.3 Batch Scoring Pipeline Flow

```
Batch Scoring Sequence

1. Load V3.1 dataset (transactions.csv + customers.csv + accounts.csv)
2. Run data validation checks (schema + null rates)
3. Compute 30+ features (feature engineering pipeline)
4. Load champion model (xgb_model.pkl or lgbm_model.pkl — selected in Week 3)
5. Champion: Predict(feature_matrix) → Raw probability per transaction
6. Calibrate raw probability → 0–100 risk score
7. Assign risk tier: GREEN (0–39) | AMBER (40–79) | RED (80–100)
8. SHAP TreeExplainer: Compute SHAP values → Rank by |SHAP| → Select top 5
9. Write audit record to JSONL audit log (PMLA compliance)
10. Append row to scored_transactions.csv:
    [transaction_id, risk_score, risk_tier, scored_at, model_version,
     f1_name, f1_value, f1_contribution, f2_name ... f5_contribution]
```

## 4.4 Future Production State Architecture

```
Production Architecture (Phase 3–4)

Real-Time Ingestion:   Payment API (ISO 20022 / MT103) → Kafka Event Stream

Real-Time Feature Store: Redis (Customer Baselines)
                         | DynamoDB (Beneficiary History)
                         | AWS Neptune (Transaction Graph — Phase 4)

Scoring Service (ECS/Fargate): Model Inference → Score Calibration
                                → SHAP/LIME Explainer → Rule Override Engine

Decision Workflow:     Approved Queue | Review Queue
                       | Rejected Queue | STR Filing → FIU-IND

Dashboard (Phase 3):   Executive View | Operational View | Analyst View (Streamlit)

MLOps Monitoring:      MLflow Registry | Prometheus/Grafana | EvidentlyAI Drift
                       | Retraining Pipeline Trigger

All infrastructure deployed in AWS ap-south-1 (Mumbai) — India data residency.
```

---

# Section 5: Data Strategy

## 5.1 Dataset Structure (V3.1 — Current Approved Dataset)

> **DATA GOVERNANCE:** The values below are the approved, source-of-truth dataset parameters. No parameter may be increased, rebalanced, or modified without explicit CrossBorderPay approval.

| Dataset | Records | Key Fields | Format |
|---|---|---|---|
| customers.csv | 500 | customer_id, business_type, gst_number, kyc_status, annual_turnover_inr, city, incorporation_date, risk_tier | CSV |
| accounts.csv | 780 | account_id, customer_id, account_type (Current/EEFC/OD), currency, opened_date, status, od_limit | CSV |
| transactions.csv | 50,800 | transaction_id, customer_id, account_id, amount_usd, currency, corridor, purpose_code, payment_rail, beneficiary_id, submission_timestamp, status, is_fraud, fraud_typology + beneficiary pool columns | CSV |

## 5.2 Approved Dataset Parameters

| Parameter | Approved Value |
|---|---|
| Total transactions | 50,800 |
| Customers | 500 |
| Accounts | 780 |
| Fraud rate (flat, all segments) | 0.5% |
| Total fraud cases | 254 |
| Maximum transaction amount | USD 200,000 |
| Business mix — Exporter | 25% |
| Business mix — Importer | 20% |
| Business mix — IT Services | 20% |
| Business mix — EdTech | 15% |
| Business mix — SaaS | 10% |
| Business mix — SME | 10% |
| Payment rails | SWIFT, Fedwire, CHIPS, SEPA, CHAPS |
| Fraud typologies (PoC focus) | 7 (as documented in Section 2.2) |

> **Constraint:** V3.1 contains 254 total fraud cases. After temporal split, the training set will contain approximately 178 fraud cases. This is a small positive class. See RAID Risk R-02 for mitigation approach.

## 5.3 Data Dictionary — Core Fields

| Field | Type | Description | Fraud Relevance |
|---|---|---|---|
| transaction_id | UUID | Unique transaction identifier | Audit trail |
| customer_id | UUID | Customer reference | Behavioural baseline lookup |
| amount_usd | Float | Transaction amount in USD equivalent | Threshold exploitation, peer deviation |
| corridor | String | Country-to-country pair (e.g., IND-UAE) | FATF risk weight |
| purpose_code | String | RBI purpose code (e.g., P0101 = export of goods) | Misdeclaration detection |
| payment_rail | Enum | SWIFT, Fedwire, CHIPS, SEPA, CHAPS | Rail × corridor mismatch |
| beneficiary_id | UUID | Beneficiary reference (embedded in transactions.csv) | New beneficiary flag |
| submission_hour | Int | Hour of submission (0–23) | Off-hours transaction anomaly |
| submission_dow | Int | Day of week (0=Mon, 6=Sun) | Weekend submission anomaly |
| days_since_last_txn | Int | Days since customer's last transaction | Dormancy signal |
| days_since_beneficiary_first_seen | Int | Age of beneficiary relationship | New beneficiary risk |
| amount_zscore_90d | Float | Amount Z-score vs. 90-day customer average | Amount deviation |
| txn_count_24h | Int | Transactions in last 24 hours | Velocity / smurfing |
| corridor_fatf_score | Float | FATF risk score for destination country (0–10) | Corridor risk |
| purpose_code_deviation | Binary | 1 if purpose code differs from customer's modal code | Purpose code abuse |
| amount_near_threshold | Binary | 1 if amount within 10% of RBI reporting threshold | Structuring signal |
| is_fraud | Binary | 0 = legitimate, 1 = confirmed fraud | Target variable |
| fraud_typology | String | Fraud type label (TBML, ATO, smurfing, etc.) | Multi-class analysis |

## 5.4 V3.1 Dataset Characteristics

The V3.1 synthetic dataset is already generated and approved. No data generation is required in Phase 1.

**Approved noise characteristics present in V3.1:**

| Characteristic | Description | Handling |
|---|---|---|
| Real NaN values | Missing values present in selected fields | Imputation during feature engineering |
| Duplicate transactions | ~0.2% approximate duplicate rate | Detect and remove pre-training |
| Failed transactions | ~3% of transactions have status = Failed | Retain; lifecycle status is a feature |
| Reversed transactions | ~0.7% Reversed | Retain; reversal is a behavioural signal |
| Cancelled transactions | ~1.4% Cancelled | Retain; cancellation after submission is suspicious |
| Beneficiary pool | Beneficiary attributes embedded as columns in transactions.csv | Use as-is |
| FEMA purpose codes | RBI purpose codes on all transactions | Use as-is |
| Transaction timing | Temporal timestamps enabling behavioural window computation | Use as-is |

> **Time Span:** Pending Confirmation from CrossBorderPay. Run `df['submission_timestamp'].min()` and `.max()` immediately after loading V3.1 to determine the actual temporal range before defining split boundaries.

## 5.5 Data Validation Framework

| Check | Tool | Trigger | Action on Failure |
|---|---|---|---|
| Schema validation | Great Expectations | Pre-training | Block training, alert |
| Null value thresholds | Great Expectations | Pre-training | Log and impute or block |
| Fraud rate within expected bounds (0.4%–0.6%) | Custom check | Post-load | Alert if outside range |
| Temporal consistency | Custom check | Pre-training | Block training |
| Duplicate transaction IDs | Great Expectations | Pre-training | Block training |
| **Label leakage test** | Logistic Regression smoke test (3 features) | Post-load | **Flag for redesign if LR AUC > 0.90** |
| Class balance verification | Custom check | Pre-training | Log warning — V3.1 has 254 fraud cases (known constraint; see RAID R-02) |

---

# Section 6: Feature Engineering Plan

## 6.1 Customer Features

| Feature Name | Description | Business Rationale | Fraud Relevance | Predictive Power |
|---|---|---|---|---|
| customer_risk_tier | Pre-assigned risk tier (Low/Med/High) based on business type and KYC | Prior risk classification informs baseline expectation | High-risk tier customers have higher fraud base rate | Medium |
| business_type_encoded | One-hot encoded business type | Different business types have different fraud profiles | Enables per-segment model calibration | Medium |
| days_since_incorporation | Company age in days at transaction date | Very new companies transacting at high volume = red flag | Shell companies often recently incorporated | High |
| kyc_completeness_score | Percentage of KYC fields populated | Incomplete KYC correlates with fraud risk | Missing documentation is a fraud signal | Medium |
| annual_turnover_usd | Annual turnover converted to USD | Anchor for amount reasonableness | Transaction disproportionate to turnover is suspicious | High |
| turnover_to_txn_amount_ratio | Transaction amount as % of annual turnover | Is this transaction size consistent with business scale? | Single transaction = 50% of annual turnover is suspicious | High |
| gst_registration_status | Active/Inactive/Pending | Active GST = legitimate operating business | Inactive GST with outward remittances = red flag | Medium |

## 6.2 Account Features

| Feature Name | Description | Business Rationale | Fraud Relevance | Predictive Power |
|---|---|---|---|---|
| account_type | Current / EEFC / OD | EEFC accounts are for forex retention; OD has credit exposure | OD account transactions at limit = risk signal | Medium |
| account_age_days | Days since account opened | New account + high value = ATO or shell signal | ATO involves newly opened or recently captured accounts | High |
| account_od_utilisation | OD utilisation % at time of transaction | High OD utilisation + outward remittance = capital flight risk | Draining OD before flight is a fraud pattern | Medium |
| currency_match | Does transaction currency match account EEFC currency? | EEFC accounts should operate in their designated currency | Currency mismatch suggests account misuse | Low |
| eefc_balance_flag | Is the EEFC account balance sufficient for this transaction? | EEFC transactions beyond retained FX = regulatory breach | Insufficient EEFC balance forces non-compliant workarounds | Medium |

## 6.3 Transaction Features

| Feature Name | Description | Business Rationale | Fraud Relevance | Predictive Power |
|---|---|---|---|---|
| log_amount_usd | Log-transformed amount | Normalises skewed amount distribution | Handles extreme outliers without losing signal | Medium |
| submission_hour | Hour of transaction submission | Legitimate B2B payments occur in business hours | 2 AM submission = potential bot or ATO | High |
| submission_dow | Day of week (0=Mon) | Weekend submissions unusual for B2B | Weekend = higher suspicion for large amounts | Medium |
| is_near_threshold | Amount within 10% of RBI reporting limit | Structuring exploits threshold-awareness | Strong smurfing signal | High |
| amendment_count | Number of amendments before final submission | Multiple amendments suggest tampering | High amendment count = deliberate modification | Medium |
| time_to_submit_hours | Hours between transaction initiation and submission | Very fast = bot; very slow = potential tampering | Extreme values in either direction are suspicious | Medium |
| rail_corridor_mismatch | Payment rail unusual for declared corridor | SEPA for India-USA is impossible | Impossible rail-corridor = data falsification | High |
| purpose_amount_anomaly | Amount unusually high for declared purpose code | Each purpose code has a typical amount range | Amount far outside purpose code norm = TBML signal | High |

## 6.4 Behavioural Features

| Feature Name | Description | Business Rationale | Fraud Relevance | Predictive Power |
|---|---|---|---|---|
| amount_zscore_30d | Z-score of amount vs. 30-day customer average | Is this amount unusual for this customer right now? | Strong signal for sudden large transactions | High |
| amount_zscore_90d | Z-score of amount vs. 90-day customer average | Medium-term behavioural baseline | Catches gradually escalating fraud | High |
| amount_pctile_peer | Amount percentile within same business type + corridor | Is this amount unusual for peers? | Peer deviation catches TBML over-invoicing | High |
| txn_count_1h | Transactions in last 1 hour | Burst in 1h = smurfing or bot | Very high burst = smurfing | High |
| txn_count_24h | Transactions in last 24 hours | Daily velocity | Elevated daily velocity = structuring | High |
| txn_count_7d | Transactions in last 7 days | Weekly velocity | Sustained elevation = systematic fraud | Medium |
| velocity_acceleration | Rate of change in velocity | Accelerating velocity more dangerous than sustained | Rapid acceleration = smurfing escalation | High |
| days_since_last_transaction | Days since customer's previous transaction | Dormancy followed by high-value = reactivation fraud | Dormant account reactivation | High |
| corridor_consistency | Is this corridor used in customer's recent history? | New corridor for established customer = risk | First use of a corridor is elevated risk | Medium |
| purpose_code_consistency | Does this purpose code match customer's historical mix? | Established customers have stable purpose profiles | Sudden purpose code change = misdeclaration | High |
| quarter_end_flag | Is submission within 5 days of quarter-end? | Quarter-end legitimate spikes need adjustment | Prevents false positives on legitimate Q-end activity | Medium |

## 6.5 Risk and Regulatory Features

| Feature Name | Description | Business Rationale | Fraud Relevance | Predictive Power |
|---|---|---|---|---|
| corridor_fatf_score | FATF risk score of destination country (0–10 scale) | Regulatory-aligned corridor risk | High FATF score = enhanced due diligence required | High |
| corridor_aml_list | Is corridor on FATF grey or blacklist? | Binary flag for high-risk jurisdiction | Grey/black list = maximum scrutiny | High |
| transaction_limit_breach | Is amount above any applicable RBI LRS or business limit? | Limit breaches are regulatory violations | Breach = automatic escalation | High |
| prior_sar_customer | Has this customer had a previous Suspicious Activity flag? | Prior suspicion elevates current risk | Prior SAR = strong prior probability | High |
| prior_suspicious_beneficiary | Has this beneficiary been linked to a previous suspicious transaction? | Beneficiary risk history | Repeat suspicious beneficiary = high risk | High |
| firc_amount_discrepancy | Difference between declared amount and FIRC amount | FIRC should match transaction amount | Discrepancy = documentation fraud | High |

---

# Section 7: Model Development Plan

## 7.1 Candidate Models Overview

```
Model Selection Flow

Training Data (Feature Matrix) →
  [Logistic Regression | XGBoost | LightGBM | Isolation Forest | Rule-Based Baseline]
→ Model Comparison Report → Champion Selection (XGBoost vs. LightGBM)
→ Champion (primary) + Isolation Forest (secondary anomaly signal)
```

## 7.2 Model Specifications

### Model 1: Rule-Based Baseline (Comparator)

| Attribute | Detail |
|---|---|
| Purpose | Establishes the performance floor; proves ML adds value |
| Rules | New beneficiary + amount > ₹50L: flag · Corridor FATF score > 7: flag · txn_count_24h > 10: flag · amount_zscore_90d > 3: flag · Purpose code deviation: flag |
| Advantages | Fully interpretable, zero latency, no training required |
| Limitations | Cannot learn, does not combine signals, high FPR, misses complex multi-signal fraud |
| Explainability | Perfect — single rule that fired |
| Expected Performance | Recall ~50%, Precision ~45%, AUC-PR ~0.45 |
| Role in PoC | Benchmark. Every ML model must beat this by ≥ 15% on recall. |

### Model 2: Logistic Regression (Interpretable ML Baseline)

| Attribute | Detail |
|---|---|
| Purpose | Establishes ML performance floor with maximum interpretability |
| Advantages | Fully interpretable coefficients, probability calibration easy, fast inference |
| Limitations | Cannot capture non-linear relationships or feature interactions |
| Explainability | Coefficient weights directly interpretable; SHAP also applicable |
| Expected Performance | Recall ~60%, Precision ~55%, AUC-PR ~0.58 |
| Role in PoC | Proves ML feature set has predictive value; establishes minimum ML bar |
| Hyperparameters | C (regularisation strength), solver, class_weight='balanced' |

### Model 3: XGBoost (Champion Candidate)

| Attribute | Detail |
|---|---|
| Purpose | Champion candidate — gradient boosted trees optimised for tabular fraud data |
| Advantages | State-of-the-art on tabular fraud data; handles imbalanced classes via scale_pos_weight; fast inference; native SHAP support via TreeExplainer |
| Limitations | More hyperparameters than LR; requires careful tuning to avoid overfitting |
| Explainability | SHAP TreeExplainer — fast, exact SHAP values; top-5 features per transaction |
| Expected Performance | Recall ~82%, Precision ~74%, AUC-PR ~0.79 |
| Role in PoC | Champion candidate 1; compared against LightGBM on validation AUC-PR |
| Class Imbalance | **scale_pos_weight = n_negative / n_positive = 50,546 / 254 ≈ 199** |
| Hyperparameters | n_estimators, max_depth, learning_rate, scale_pos_weight=199, subsample, colsample_bytree, min_child_weight |

### Model 4: LightGBM (Champion Candidate)

| Attribute | Detail |
|---|---|
| Purpose | Champion candidate — fast gradient boosting with leaf-wise tree growth; often outperforms XGBoost on large sparse feature sets |
| Advantages | Faster training than XGBoost; better handling of high-cardinality categoricals; native class_weight support; SHAP-compatible |
| Limitations | More susceptible to overfitting on small datasets; requires careful num_leaves tuning |
| Explainability | SHAP TreeExplainer — compatible; top-5 features per transaction |
| Expected Performance | Recall ~80–84%, Precision ~72–76%, AUC-PR ~0.77–0.81 (to be confirmed empirically) |
| Role in PoC | Champion candidate 2; if AUC-PR > XGBoost on validation set, LightGBM becomes champion |
| Class Imbalance | **is_unbalance=True** or **scale_pos_weight≈199** (equivalent to XGBoost setting) |
| Hyperparameters | num_leaves, max_depth, learning_rate, n_estimators, is_unbalance, min_child_samples, feature_fraction |

### Model 5: Isolation Forest (Anomaly Detection Layer)

| Attribute | Detail |
|---|---|
| Purpose | Unsupervised anomaly detection — catches novel fraud patterns not in training labels |
| Advantages | Does not require fraud labels; catches emerging fraud typologies; complements supervised model |
| Limitations | Cannot be calibrated to a probability; false positive rate harder to control |
| Expected Performance | Recall ~55% on labelled fraud; catches ~20% of fraud not detected by champion model |
| Role in PoC | Secondary signal — elevates champion score when Isolation Forest also flags the transaction |

## 7.3 Champion Model Selection Strategy

**Recommended champion selection: XGBoost vs. LightGBM comparison on validation AUC-PR**

1. Train all models (LR, XGBoost, LightGBM, Isolation Forest) and rule baseline on training set
2. Evaluate all on validation set using AUC-PR as primary metric
3. Select champion: whichever of XGBoost or LightGBM achieves higher AUC-PR on validation set (minimum threshold: ≥ 5% improvement over LR)
4. Run Isolation Forest as parallel layer; use anomaly score as additional feature in final scoring
5. Deploy champion model; document all results in model comparison report

**Threshold Selection:** Do NOT use default 0.5 threshold. Use F1-maximising threshold on validation set. Document precision-recall curve at multiple thresholds. Recommend operational threshold to CrossBorderPay for final approval (Open Item OI-08).

## 7.4 Class Imbalance Handling

| Technique | Applied To | Rationale |
|---|---|---|
| scale_pos_weight ≈ 199 in XGBoost | Training | Native XGBoost imbalance handling; critical at 0.5% fraud rate |
| is_unbalance=True in LightGBM | Training | LightGBM native imbalance handling (equivalent effect) |
| class_weight='balanced' in LR | Training | Sklearn imbalance handling |
| Threshold tuning on validation set | Post-training | Operating threshold ≠ 0.5 |
| Stratified k-fold CV (k=5) | Cross-validation | Ensures fraud cases in every fold |
| SMOTE (conditional) | Training augmentation | Only if AUC-PR < 0.65 after threshold tuning |
| Calibration (Platt scaling / Isotonic) | Post-training | Ensures score = calibrated probability |

---

# Section 8: Model Evaluation Framework

## 8.1 Primary Metrics

| Metric | Formula | Minimum Target | Strong Target | Why This Metric |
|---|---|---|---|---|
| AUC-PR | Area under Precision-Recall curve | ≥ 0.70 | ≥ 0.79 | Better than ROC-AUC for imbalanced classes; not inflated by true negatives |
| Recall (Fraud Detection Rate) | TP / (TP + FN) | ≥ 75% | ≥ 82% | Primary metric — how much fraud do we catch? |
| Precision | TP / (TP + FP) | ≥ 60% | ≥ 74% | Cost of false positives — legitimate transactions blocked |
| F1 Score | 2 × (P × R) / (P + R) | ≥ 0.66 | ≥ 0.75 | Harmonic balance of precision and recall |
| False Positive Rate | FP / (FP + TN) | ≤ 3% | ≤ 1.5% | % of legitimate transactions wrongly flagged |
| ROC-AUC | Area under ROC curve | ≥ 0.88 | ≥ 0.93 | Secondary metric — overall discrimination |

## 8.3 Evaluation Protocol

```
Temporal Train / Validation / Test Split (Critical — must be time-based, not random)

Full Dataset: 50,800 transactions (V3.1 approved)
  ├── 70% Train      → ~35,560 transactions · ~178 fraud cases
  ├── 20% Validation → ~10,160 transactions · ~51 fraud cases (threshold tuning + model selection)
  └── 10% Test       → ~5,080 transactions  · ~25 fraud cases (HOLDOUT — final evaluation only)

Time span: Pending Confirmation from CrossBorderPay.
Run df['submission_timestamp'].min() / .max() immediately on data load
to establish actual temporal split boundary dates.

A random split would allow the model to see future behavioural patterns during training,
which is impossible in production. Temporal split simulates realistic production conditions.
```

## 8.4 Model Comparison Matrix

| Metric | Rule Baseline | Logistic Regression | XGBoost | LightGBM | Target |
|---|---|---|---|---|---|
| AUC-PR | TBD | TBD | TBD | TBD | ≥ 0.70 |
| Recall @ 5% FPR | TBD | TBD | TBD | TBD | ≥ 75% |
| Precision @ threshold | TBD | TBD | TBD | TBD | ≥ 60% |
| F1 Score | TBD | TBD | TBD | TBD | ≥ 0.66 |
| Batch inference time (50K rows) | < 1s | < 10s | < 60s | < 45s | < 5 min |
| Explainability | Rule fired | Coefficient | SHAP top-5 | SHAP top-5 | Full |

*TBD values to be populated during model training in Week 3.*

---

# Section 9: Explainability Framework

## 9.1 Explainability Architecture

```
Transaction Feature Vector
→ Champion Model (XGBoost or LightGBM — Predict Probability)
→ Score Calibration (0–100 scale)
→ Score Tier Assignment (GREEN / AMBER / RED)
→ SHAP TreeExplainer (Primary — exact SHAP values for all features)
   [LIME available as alternative for validation and non-tree model comparison]
→ Rank Features by |SHAP contribution|
→ Select Top 5 Features
→ Audit Logger (JSONL — PMLA compliance)
→ Batch Output Row (CSV)
```

## 9.2 Top-5 SHAP/LIME Output Specification

Each scored transaction outputs the top 5 features by absolute contribution.

**Primary method: SHAP TreeExplainer** (exact, fast, native to XGBoost and LightGBM).
**Validation method: LIME** (model-agnostic; used for cross-checking SHAP explanations and for Logistic Regression comparison).

**Output format: Feature Name · Feature Value · Contribution Score only. No reason codes. No plain-language narratives.**

| Output Column | Content | Data Type |
|---|---|---|
| f1_name | Name of feature with highest \|contribution\| | String |
| f1_value | Actual feature value for this transaction | Float |
| f1_contribution | SHAP contribution (positive = pushes toward fraud; negative = pushes toward legitimate) | Float |
| f2_name | Feature with 2nd highest contribution | String |
| f2_value | Actual value | Float |
| f2_contribution | Contribution score | Float |
| f3_name through f5_name | Features 3–5 by contribution rank | String |
| f3_value through f5_value | Actual values | Float |
| f3_contribution through f5_contribution | Contribution scores | Float |

## 9.3 Sample Batch Output Row

```
transaction_id,risk_score,risk_tier,scored_at,model_version,
f1_name,f1_value,f1_contribution,
f2_name,f2_value,f2_contribution,
f3_name,f3_value,f3_contribution,
f4_name,f4_value,f4_contribution,
f5_name,f5_value,f5_contribution

TXN-2024-09-182736,91,RED,2024-09-18T14:32:17Z,xgb-v1.0.0,
days_since_beneficiary_first_seen,3.0,0.34,
amount_zscore_90d,4.2,0.28,
corridor_fatf_score,8.5,0.19,
txn_count_1h,4.0,0.12,
submission_hour,2.0,0.09
```

> **PMLA Audit Requirement:** Every scored row — including all 5 SHAP/LIME features — is written to both the output CSV and the JSONL audit log.

## 9.4 Explainability Use Cases

| Audience | Method | Format | Content |
|---|---|---|---|
| Fraud Analyst | SHAP (primary) | Top-5 features in batch output CSV | Feature name and value allow analyst to understand what drove the score |
| PMLA Audit Log | SHAP (primary) | JSONL audit record per transaction | Full contribution values + risk score + tier + model version |
| Model Validation | LIME (validation) | Comparison report | Cross-check SHAP explanations; flag disagreements between methods |

---

# Section 10: Batch Scoring Pipeline

## 10.1 Pipeline Overview

The batch scoring pipeline replaces a real-time API for Phase 1 PoC. It processes the full V3.1 dataset (or any subset) offline and produces a scored output CSV with risk scores and SHAP/LIME explainability. Real-time API scoring is a Phase 3–4 deliverable.

| Attribute | Detail |
|---|---|
| Input | transactions.csv (V3.1 — 50,800 rows) + feature-engineered matrix |
| Output | scored_transactions.csv (50,800 rows with risk score, tier, top-5 SHAP features) |
| Execution | Python script: `src/scoring/batch_scorer.py` or `notebooks/06_batch_scoring.ipynb` |
| Runtime | Target: complete full dataset in < 5 minutes on standard developer hardware |
| Model | Champion model (xgb_model.pkl or lgbm_model.pkl) + Isolation Forest (if_model.pkl) |
| Explainability | SHAP TreeExplainer (primary); LIME (validation) |

## 10.2 Pipeline Configuration

| Parameter | Value |
|---|---|
| Input file | `data/processed/feature_matrix.parquet` |
| Model path | `models/champion_model.pkl` |
| Calibration path | `models/calibration.pkl` |
| Output file | `data/output/scored_transactions.csv` |
| Audit log | `data/output/audit_log.jsonl` |
| Batch size | 5,000 rows (configurable — for memory management) |
| Random seed | 42 (fixed for reproducibility) |
| Model version | Populated from MLflow experiment tag |

## 10.3 Output CSV Schema

```
transaction_id       STRING   — Original transaction ID from V3.1
risk_score           INT      — 0–100 calibrated risk score
risk_tier            STRING   — GREEN | AMBER | RED
scored_at            DATETIME — ISO 8601 timestamp of scoring run
model_version        STRING   — MLflow run ID / model tag (e.g., xgb-v1.0.0)
f1_name              STRING   — Top SHAP feature name
f1_value             FLOAT    — Feature value for this transaction
f1_contribution      FLOAT    — SHAP contribution (positive = fraud direction)
f2_name              STRING
f2_value             FLOAT
f2_contribution      FLOAT
f3_name              STRING
f3_value             FLOAT
f3_contribution      FLOAT
f4_name              STRING
f4_value             FLOAT
f4_contribution      FLOAT
f5_name              STRING
f5_value             FLOAT
f5_contribution      FLOAT
```

## 10.4 Risk Tier Assignment

| Tier | Score Range | Action |
|---|---|---|
| GREEN | 0–39 | Approved — payment proceeds without analyst intervention |
| AMBER | 40–79 | Review — payment queued for human decision |
| RED | 80–100 | Rejected — payment held pending analyst investigation |

> **Threshold Note:** Tier boundaries (40 and 80) are initial defaults. Final thresholds are tuned on the validation set using the F1-maximising threshold and submitted to CrossBorderPay for operational approval (Open Item OI-08).

## 10.5 PMLA Audit Compliance

Every transaction scored by the pipeline generates an audit record written to `audit_log.jsonl`:

```json
{
  "transaction_id": "TXN-2024-09-182736",
  "risk_score": 91,
  "risk_tier": "RED",
  "scored_at": "2024-09-18T14:32:17Z",
  "model_version": "xgb-v1.0.0",
  "top_5_shap_features": [
    {"rank": 1, "name": "days_since_beneficiary_first_seen", "value": 3.0, "contribution": 0.34},
    {"rank": 2, "name": "amount_zscore_90d",                 "value": 4.2, "contribution": 0.28},
    {"rank": 3, "name": "corridor_fatf_score",               "value": 8.5, "contribution": 0.19},
    {"rank": 4, "name": "txn_count_1h",                      "value": 4.0, "contribution": 0.12},
    {"rank": 5, "name": "submission_hour",                   "value": 2.0, "contribution": 0.09}
  ]
}
```

> **PMLA Requirement:** Logging every fraud score plus the top-5 features used per decision satisfies the PMLA obligation to maintain auditable ML decision records.

## 10.6 Error Handling

| Condition | Handling |
|---|---|
| Missing feature values (NaN) | Impute using training-set medians; flag in audit log |
| Transaction already scored | Skip and log warning; do not overwrite unless --overwrite flag set |
| Model file not found | Raise exception; do not produce partial output |
| Output write failure | Raise exception; retain input data |
| SHAP computation failure | Log error for affected row; write score without SHAP columns; flag in audit |

---

---

# Section 11: Dashboard Design

*Dashboard design has been removed from the Phase 1 PoC scope. See Section 19 (Future Roadmap � Phase 3) for planned dashboard deliverables.*

---

# Section 12: Sandbox Deployment Plan

## 12.1 Sandbox Environment Specification

| Component | Specification | Purpose |
|---|---|---|
| Execution environment | Developer laptop / local workstation | No cloud infrastructure in PoC |
| OS requirement | Windows 10+ or Ubuntu 20.04+ | Standard development environment |
| Python version | 3.10 (pinned in pyproject.toml / requirements.txt) | Reproducibility |
| Package manager | pip + virtualenv (or conda) | Dependency isolation |
| Container | Docker Desktop (for reproducible pipeline delivery) | Ensures demo runs identically on any machine |
| Version control | Git (local repo; GitHub private repo for delivery) | Code handoff and version tracking |
| Experiment tracking | MLflow (local tracking server � runs in Docker) | Model comparison and version management |
| Data storage | Local disk (data/ directory in project repo) | V3.1 CSV files; no cloud storage |
| Secrets management | .env file (gitignored) | API keys if required; none expected for PoC |

## 12.2 Hardware Requirements (Minimum)

| Requirement | Minimum Spec | Recommended Spec |
|---|---|---|
| RAM | 8 GB | 16 GB |
| CPU | 4 cores | 8 cores |
| Disk | 10 GB free | 20 GB free |
| GPU | Not required (CPU training sufficient for V3.1 dataset) | � |

> **Open Item OI-INF-03:** Developer machine specification for the CrossBorderPay-side team is pending confirmation. See Section 16.

## 12.3 Project Directory Structure

```
CrossBorderPay-fraud-poc/
+-- data/
�   +-- raw/               # V3.1 source CSVs (customers, accounts, transactions)
�   +-- processed/         # Feature matrices (Parquet)
�   +-- output/            # Scored output CSVs + JSONL audit logs
+-- notebooks/
�   +-- 01_eda.ipynb
�   +-- 02_data_validation.ipynb
�   +-- 03_feature_engineering.ipynb
�   +-- 04_model_training_xgb.ipynb
�   +-- 04b_model_training_lgbm.ipynb
�   +-- 05_model_evaluation.ipynb
�   +-- 06_batch_scoring.ipynb
�   +-- 07_shap_explainability.ipynb
+-- src/
�   +-- data/              # Data loaders + validation
�   +-- features/          # Feature engineering pipeline
�   +-- models/            # Model training + calibration scripts
�   +-- scoring/           # Batch scorer (batch_scorer.py)
�   +-- explainability/    # SHAP + LIME wrappers
+-- models/                # Serialised model artifacts (.pkl)
+-- tests/                 # Unit tests (pytest)
+-- mlruns/                # MLflow local tracking directory
+-- Dockerfile
+-- requirements.txt       # Pinned dependencies
+-- pyproject.toml
+-- .env.example
+-- README.md
```

## 12.4 Key Python Dependencies

| Package | Version (pinned) | Purpose |
|---|---|---|
| xgboost | = 2.0 | XGBoost champion candidate |
| lightgbm | = 4.0 | LightGBM champion candidate |
| scikit-learn | = 1.4 | Logistic Regression, Isolation Forest, metrics, calibration |
| shap | = 0.45 | SHAP TreeExplainer for XGBoost and LightGBM |
| lime | = 0.2 | LIME for model-agnostic validation explanations |
| pandas | = 2.1 | Data manipulation |
| numpy | = 1.26 | Numerical computation |
| great-expectations | = 0.18 | Data validation |
| mlflow | = 2.10 | Experiment tracking |
| imbalanced-learn | = 0.11 | SMOTE (conditional) |
| matplotlib / seaborn | = 3.8 / = 0.13 | EDA and evaluation plots |

## 12.5 Reproducibility Constraints

| Constraint | Enforcement |
|---|---|
| Random seeds | Set `random_state=42` for all models and splits; `np.random.seed(42)` at notebook top |
| Pinned dependencies | `requirements.txt` frozen; no floating `>=` for core ML packages |
| MLflow experiment logging | Every training run tracked with parameters, metrics, and model artifact |
| Docker build | `Dockerfile` pins base image digest, not just tag |

---

# Section 13: Delivery Plan

## 13.1 Five-Week PoC Timeline

| Phase | Duration | End Condition |
|---|---|---|
| Week 0 | Pre-start | Open items OI-01, OI-03, OI-07 resolved; kickoff with CrossBorderPay completed |
| Week 1�5 | PoC execution | PoC demo delivered to CrossBorderPay team |
| Total estimated effort | ~170 person-hours | Across PoC delivery team |

## 13.2 Week-by-Week Delivery Schedule

### Week 1: Environment Setup + EDA + Data Validation

| Activity | Owner | Output |
|---|---|---|
| Confirm all open items with CrossBorderPay (OI-01, OI-03, OI-07) | Lead | Open items resolved or escalated |
| Set up project repo, virtualenv, Docker, MLflow | Dev | Working sandbox environment |
| Load V3.1 dataset; run schema and null validation | Dev | Data validation report (Great Expectations) |
| Conduct EDA: amount distribution, corridor mix, fraud rate verification, temporal range | Dev/ML | EDA notebook (01_eda.ipynb) |
| **Label leakage test:** Run Logistic Regression with 3 features � if AUC > 0.90, STOP and investigate | ML | Leakage gate pass/fail |
| Build rule-based baseline (4 rules) | ML | Rule baseline script |

**Week 1 acceptance criteria:** Dataset loaded, EDA complete, leakage gate passed, rule baseline running.

### Week 2: Feature Engineering

| Activity | Owner | Output |
|---|---|---|
| Build customer and account features (7 + 5 = 12 features) | Dev | Feature module: customer_features.py, account_features.py |
| Build transaction features (8 features) | Dev | transaction_features.py |
| Build behavioural window features (11 features): 30d/90d Z-scores, velocity counts (1h, 24h, 7d), corridor consistency, purpose code consistency | ML | behavioural_features.py |
| Build risk/regulatory features (6 features) | ML | risk_features.py |
| Compile 30+ feature matrix; write to Parquet | Dev | data/processed/feature_matrix.parquet |
| Run full data validation on feature matrix | Dev | Validation report |

**Week 2 acceptance criteria:** 30+ feature matrix produced, validated, written to Parquet. Feature engineering pipeline fully reproducible from seed.

### Week 3: Model Training + Champion Selection

| Activity | Owner | Output |
|---|---|---|
| Train Logistic Regression baseline with class_weight='balanced' | ML | LR model artifact + MLflow run |
| Train XGBoost with scale_pos_weight�199 (hyperparameter tuning via grid search or Optuna) | ML | XGBoost model artifact + MLflow run |
| Train LightGBM with is_unbalance=True (hyperparameter tuning) | ML | LightGBM model artifact + MLflow run |
| Train Isolation Forest on full feature matrix | ML | Isolation Forest model artifact |
| Run rule-based baseline evaluation | ML | Rule baseline results |
| Compare XGBoost vs. LightGBM on validation AUC-PR | ML | Champion selection decision |
| Select champion (XGBoost or LightGBM); document rationale | ML | Champion selection memo |
| Score calibration (Platt scaling) on champion model | ML | calibration.pkl |
| Calibrate risk scores to 0�100 scale | ML | Calibration verified |

**Week 3 acceptance criteria:** All models trained; champion selected; AUC-PR = 0.70 on validation set.

### Week 4: Evaluation + Explainability + Pipeline

| Activity | Owner | Output |
|---|---|---|
| Run champion model on held-out test set; compute all metrics (AUC-PR, Recall, Precision, F1, FPR) | ML | Model evaluation report (05_model_evaluation.ipynb) |
| Generate precision-recall curve; identify F1-maximising threshold | ML | Operating threshold recommendation |
| Run SHAP TreeExplainer on champion model; compute top-5 features per transaction | ML | SHAP output in scored CSV |
| Run LIME on a sample of transactions for cross-validation against SHAP | ML | LIME comparison report (07_shap_explainability.ipynb) |
| Build batch scoring pipeline (batch_scorer.py): CSV in ? feature engineering ? score + SHAP ? scored CSV out | Dev | src/scoring/batch_scorer.py |
| Write PMLA audit log (JSONL) from pipeline | Dev | data/output/audit_log.jsonl |
| Containerise pipeline in Docker | Dev | Dockerfile + Docker image |

**Week 4 acceptance criteria:** Test set metrics meet targets (AUC-PR = 0.70, Recall = 75%); SHAP/LIME explanations verified; batch pipeline runs end-to-end; JSONL audit log produced.

### Week 5: Model Card + Regulatory Memo + PoC Demo

| Activity | Owner | Output |
|---|---|---|
| Write model card (model purpose, training data, evaluation metrics, limitations, governance) | ML | model_card.md |
| Write regulatory positioning memo (PMLA/RBI alignment, STR workflow note, FEMA purpose code alignment) | Lead | regulatory_memo.md |
| Prepare PoC to Production roadmap deck (10 slides) | Lead | slides/ (PPT or Gamma) |
| Run full adversarial test (3 scenarios: TBML, ATO, Smurfing) | ML | Adversarial test results |
| Final demo run: batch pipeline on V3.1 ? scored_transactions.csv | Dev | Demo verified end-to-end |
| Demo delivery to CrossBorderPay team | Lead | PoC sign-off |

**Week 5 acceptance criteria:** Model card complete; regulatory memo signed off; batch pipeline demonstrated end-to-end; PoC demo delivered to CrossBorderPay team; all technical success criteria met.

---

# Section 14: Work Breakdown Structure

## 14.1 WBS Summary

| WBS ID | Work Package | Owner | Estimate (hrs) | Week |
|---|---|---|---|---|
| 1.0 | Project Initiation | Lead | 12 | W0 |
| 1.1 | Kickoff meeting with CrossBorderPay (OI resolution) | Lead | 4 | W0 |
| 1.2 | Environment setup (repo, Docker, MLflow) | Dev | 8 | W1 |
| 2.0 | Data Foundation | Dev + ML | 24 | W1 |
| 2.1 | Load V3.1 dataset + schema validation | Dev | 4 | W1 |
| 2.2 | EDA (distributions, fraud rate, temporal range) | ML | 8 | W1 |
| 2.3 | Label leakage gate (3-feature LR smoke test) | ML | 4 | W1 |
| 2.4 | Rule-based baseline build and evaluation | ML | 8 | W1 |
| 3.0 | Feature Engineering | Dev + ML | 32 | W2 |
| 3.1 | Customer + account features (12 features) | Dev | 8 | W2 |
| 3.2 | Transaction features (8 features) | Dev | 8 | W2 |
| 3.3 | Behavioural window features (11 features) | ML | 12 | W2 |
| 3.4 | Risk/regulatory features (6 features) | ML | 4 | W2 |
| 4.0 | Feature matrix compilation + validation | Dev | 8 | W2 |
| 4.1 | 30+ feature matrix ? Parquet | Dev | 4 | W2 |
| 4.2 | Great Expectations data validation | Dev | 4 | W2 |
| 5.0 | Model Training | ML | 40 | W3 |
| 5.1 | Logistic Regression (baseline ML) | ML | 4 | W3 |
| 5.2 | XGBoost (candidate champion, scale_pos_weight�199) | ML | 12 | W3 |
| 5.3 | LightGBM (candidate champion, is_unbalance=True) | ML | 12 | W3 |
| 5.4 | Isolation Forest (anomaly layer) | ML | 6 | W3 |
| 5.5 | Rule baseline evaluation | ML | 2 | W3 |
| 5.6 | Score calibration (Platt scaling / Isotonic) | ML | 4 | W3 |
| 5.7 | Champion selection (XGBoost vs. LightGBM on validation AUC-PR) | ML | 4 | W3 |
| 6.0 | Evaluation + Explainability | ML | 28 | W4 |
| 6.1 | Held-out test set evaluation (all metrics) | ML | 8 | W4 |
| 6.2 | Precision-recall curve + operating threshold | ML | 4 | W4 |
| 6.3 | SHAP TreeExplainer (all transactions) | ML | 8 | W4 |
| 6.4 | LIME cross-validation sample | ML | 4 | W4 |
| 6.5 | Adversarial test cases (3 scenarios) | ML | 4 | W4 |
| 7.0 | Batch Scoring Pipeline | Dev | 24 | W4 |
| 7.1 | batch_scorer.py (CSV in ? scored CSV out) | Dev | 12 | W4 |
| 7.2 | PMLA JSONL audit logger | Dev | 4 | W4 |
| 7.3 | Docker containerisation | Dev | 8 | W4 |
| 8.0 | Documentation + Demo | Lead + ML | 20 | W5 |
| 8.1 | Model card | ML | 4 | W5 |
| 8.2 | Regulatory positioning memo | Lead | 6 | W5 |
| 8.3 | PoC to Production roadmap deck | Lead | 6 | W5 |
| 8.4 | Demo preparation + dry run | Dev + ML | 4 | W5 |
| **Total** | | | **~188 person-hours** | |

> **Note:** Target is ~170 person-hours. Buffer of ~18 hours accounts for open item resolution (W0), integration testing, and stakeholder availability.

---

# Section 15: RAID Log

## 15.1 Risks

| ID | Risk | Probability | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R-01 | Label leakage: synthetic generator created features that directly encode is_fraud | Low | Critical | Label leakage gate in Week 1: if LR AUC (3 features) > 0.90, stop and regenerate | ML Lead |
| R-02 | Small positive class (254 fraud cases, ~178 in training): champion model fails AUC-PR target | Medium | High | scale_pos_weight�199 for XGBoost; is_unbalance=True for LightGBM; compare both candidates; SMOTE if AUC-PR < 0.65 after tuning; Isolation Forest as secondary signal | ML Lead |
| R-03 | Temporal split results in fewer than expected fraud cases in training set | Low | Medium | Verify fraud case count post-split; if training fraud < 150, adjust split boundaries | Dev |
| R-04 | SHAP computation slow for full 50,800 transactions | Low | Low | SHAP TreeExplainer is O(n�depth) � should complete in < 5 min; use background summary if needed | Dev |
| R-05 | Open items (OI-01, OI-03, OI-07) not resolved before Week 1 | Medium | High | Proceed with assumptions; flag and document risk in model card | Lead |
| R-06 | LightGBM and XGBoost produce near-identical AUC-PR (tie) | Low | Low | If AUC-PR difference < 0.01, select XGBoost for stronger SHAP ecosystem integration; document decision | ML Lead |

## 15.2 Assumptions

| ID | Assumption | Confirmed By | Risk if Wrong |
|---|---|---|---|
| A-01 | V3.1 dataset (50,800 rows, 0.5% fraud rate) is sufficient for training a demonstrable model | Dataset governance � confirmed | Low (dataset already generated) |
| A-02 | Developer machine meets 8 GB RAM minimum; Docker Desktop available | Pending (OI-INF-03) | Medium � may need to reduce dataset or use cloud notebook |
| A-03 | XGBoost or LightGBM (or both) will exceed LR by = 15% on recall on V3.1 | Assumption based on typical tabular fraud patterns | High � would require redesign of model strategy |
| A-04 | CrossBorderPay team can attend Week 5 demo (Weeks 0�5 span covers no critical business blackout) | Pending confirmation | Medium � demo can be rescheduled |
| A-05 | SHAP TreeExplainer output is acceptable for PMLA audit purposes (no formal RBI ruling on specific explainability method required) | Assumed � verify with compliance team | High � may require formal method change |

## 15.3 Issues

| ID | Issue | Status | Owner | Resolution Path |
|---|---|---|---|---|
| I-01 | OI-03: CrossBorderPay existing fraud detection system not confirmed | Open | Lead | Resolve in Week 0 kickoff |
| I-02 | OI-07: Operational fraud detection threshold not confirmed | Open | Lead | Resolve in Week 0; default 40/80 used until confirmed |
| I-03 | Actual temporal range of V3.1 dataset unknown until data is loaded | Open | Dev | Run `submission_timestamp.min()/.max()` in Week 1 Day 1 |

## 15.4 Dependencies

| ID | Dependency | On | Required By |
|---|---|---|---|
| D-01 | V3.1 dataset on disk and accessible | CrossBorderPay / Dataset team | Week 1 Day 1 |
| D-02 | Open items OI-01, OI-03, OI-07 resolved | CrossBorderPay | Week 1 Day 1 |
| D-03 | Python 3.10 + Docker Desktop installed on all developer machines | Infrastructure | Week 1 Day 1 |
| D-04 | XGBoost or LightGBM champion selected | Week 3 model training | Week 4 SHAP + pipeline |
| D-05 | Scored output CSV available | Week 4 pipeline | Week 5 demo |

---

# Section 16: Open Items Register

## 16.1 P1 � Must Resolve Before Week 1 Start

| ID | Open Item | Impact if Unresolved | Owner | Target Date |
|---|---|---|---|---|
| OI-01 | Confirm whether V3.1 dataset's fraud_typology field contains usable labels for training (not masked or anonymised) | Without labels, supervised training is impossible | CrossBorderPay | Week 0 |
| OI-03 | Does CrossBorderPay have an existing rule-based fraud detection system in production? | If yes, PoC baseline and scope definition changes | CrossBorderPay | Week 0 |
| OI-07 | What fraud detection threshold does the platform's risk team consider operationally acceptable? (FPR = ?) | Cannot set operating threshold without business input | CrossBorderPay | Week 0 |

## 16.2 P2 � Resolve During PoC

| ID | Open Item | Impact if Unresolved | Owner | Target Date |
|---|---|---|---|---|
| OI-08 | Confirm operational risk tier boundaries (40/80 default) for Approved/Review/Reject tiers | Tier boundaries go to default until confirmed | CrossBorderPay | Week 3 |
| OI-09 | Confirm PMLA audit log format acceptable for internal compliance review | Pilot JSONL format until confirmed | Compliance | Week 4 |
| OI-13 | Confirm payment rail distribution (SWIFT 60%, Fedwire 15%, CHIPS 10%, SEPA 10%, CHAPS 5%) | Rail distribution affects corridor feature credibility | CrossBorderPay | Week 2 |
| OI-INF-03 | Developer machine specification for CrossBorderPay team | May require cloud Jupyter environment if machine is under spec | CrossBorderPay | Week 0 |

## 16.3 P3 � Post-PoC (Phase 2 Planning)

| ID | Open Item | Impact | Owner | Target Date |
|---|---|---|---|---|
| OI-P2-01 | Real transaction data access strategy for Phase 2 (production labels, PII masking, data residency) | Phase 2 cannot start without | CrossBorderPay + Compliance | Phase 2 kickoff |
| OI-P2-02 | STR filing workflow integration: which system does FIU-IND submission go through? | Determines regulatory integration in Phase 2 | CrossBorderPay + DBS/JP Morgan | Phase 2 kickoff |
| OI-P2-03 | Model monitoring and drift detection ownership after PoC | Determines Phase 3 MLOps scope | CrossBorderPay | Phase 2 kickoff |

---
# Section 17: Weekly Status Report Templates

*Weekly status report templates have been removed from the Phase 1 PoC scope. Status reporting format to be agreed with CrossBorderPay at Week 0 kickoff.*

---

# Section 18: Demo Plan

*Demo plan details have been removed from the Phase 1 PoC scope. Demo delivery to CrossBorderPay team is the Week 5 end condition (Section 13.1). Demo preparation is tracked in WBS item 8.4.*

---

# Section 19: Future Roadmap

## 19.1 Phase Overview

| Phase | Scope | Trigger |
|---|---|---|
| Phase 1 (This PoC) | Batch scoring, synthetic dataset, champion model selection (XGBoost or LightGBM) | In progress |
| Phase 2 | Real transaction data, production labels, model retraining, STR filing workflow | PoC sign-off from CrossBorderPay |
| Phase 3 | Real-time scoring API (ECS/Fargate), analyst dashboard, MLOps monitoring | Phase 2 AUC-PR > 0.80 on real data |
| Phase 4 | Graph/network features (AWS Neptune), cross-customer signal detection | Phase 3 stable in production |
| Phase 5 | ISO 20022 MX payload integration, GIFT City corridor expansion | Regulatory approval |

## 19.2 Phase 2 Key Activities

| Activity | Description |
|---|---|
| Real data access | Work with DBS/JP Morgan to access masked real transaction data (India data residency � ap-south-1) |
| Production labels | Obtain confirmed fraud labels from DBS partner system or historical STR database |
| PII masking pipeline | Tokenise customer and beneficiary identifiers before model ingestion |
| Model retraining | Retrain XGBoost/LightGBM champion on real data; compare vs. synthetic-trained model |
| Threshold recalibration | Recalibrate GREEN/AMBER/RED thresholds using real fraud rate (may differ significantly from 0.5%) |
| STR filing integration | Build STR workflow: RED-tier transaction ? analyst review ? STR draft ? FIU-IND submission |
| Regulatory positioning memo (final) | Update PoC regulatory memo with real data processing context for RBI |

## 19.3 Phase 3 Key Activities

| Activity | Description |
|---|---|
| Real-time scoring API | ECS/Fargate scoring service: ISO 20022 MT103/MX payload ? score in < 500ms |
| Analyst dashboard | Executive, Operational, and Fraud Analyst views (Streamlit or internal tool) |
| MLOps monitoring | EvidentlyAI data drift detection; automated retraining trigger; Prometheus/Grafana metrics |
| Model registry | MLflow production registry; A/B champion/challenger deployment |
| Feedback loop | Analyst override decisions feed back into retraining pipeline |

## 19.4 Phase 4�5 Key Activities

| Activity | Description |
|---|---|
| Graph features (Phase 4) | AWS Neptune for beneficiary network analysis; cross-customer transaction graph; hub detection for smurfing |
| GIFT City expansion (Phase 5) | IFSC corridor-specific risk rules; GIFT City transaction flow modelling |
| ISO 20022 MX integration (Phase 5) | Full XML payload parsing; pacs.008 / pain.001 message types for feature extraction |

---

# Section 20: Final Recommendation

## 20.2 Recommended Priorities

| Priority | Item | Rationale |
|---|---|---|
| P1 | Resolve OI-01, OI-03, OI-07 with CrossBorderPay before Week 1 starts | These three open items are hard blockers. Training cannot begin until OI-01 (label usability) is confirmed. Scope cannot be locked until OI-03 (existing system) is answered. Threshold cannot be set until OI-07 is confirmed. |
| P1 | Pass the label leakage gate in Week 1 before model training begins | A leakage AUC > 0.90 on 3 features invalidates all downstream results. This gate must be explicit and documented. |
| P1 | Compare XGBoost and LightGBM head-to-head on validation AUC-PR in Week 3 | Both are champion candidates per PoC design. Champion selection must be data-driven and documented in the model card. |
| P2 | Produce SHAP explanations for 100% of scored transactions (not just flagged ones) | PMLA audit compliance requires every decision to be traceable. A partial SHAP run is not acceptable. LIME cross-validation on a sample provides methodological confidence. |
| P2 | Submit the regulatory positioning memo as part of the Week 5 demo package | DBS and JP Morgan will scrutinise regulatory alignment before Phase 2 real-data access is granted. The memo must be complete before Phase 2 kickoff. |
| P3 | Plan Phase 2 data access strategy in parallel with PoC Week 5 | The gap between Phase 1 completion and Phase 2 data access approval can be 4�8 weeks. Starting the data access process in Week 5 prevents this becoming a critical path blocker. |

---

*End of Document*

**Cross-Border Transaction Risk Scoring Engine � PoC Plan V5.0**
**cross-border payments company | Bengaluru | **
**Confidential � For CrossBorderPay Internal Use Only**
