# Cross-Border Transaction Risk Scoring Engine
## Foundational Fraud Risk Scoring Capability — Proof of Concept Plan
### Version 3.0 | June 2026

> **Confidential — For CEO & CTO Review Only**

---

## Document Control

| Field | Detail |
|---|---|
| Document Title | Transaction Risk Scoring Engine — Proof of Concept Plan |
| Version | 3.0 |
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

CrossBorderPay operates as a cross-border A2A payment platform serving Indian businesses — exporters, importers, IT services firms, EdTech companies, SaaS providers, and SMEs. Every transaction carries regulatory risk under FEMA and PMLA, reputational risk with banking partners DBS Bank and JP Morgan, and financial risk from fraud typologies specific to the Indian cross-border payments corridor.

CrossBorderPay is building its fraud scoring capability from the ground up. No existing ML-based or validated rule-based fraud detection system has been confirmed (Open Item OI-03). The PoC will establish a foundational capability — not replace an existing one.

> **Core Problem:** CrossBorderPay does not currently have a calibrated, explainable, machine learning–based fraud detection capability. Cross-border payments are exposed to complex, multi-signal fraud patterns — TBML, round-tripping, smurfing, account takeover — that require adaptive ML scoring rather than static rules. The PoC will prove this capability is technically viable.

## 1.2 Current Challenges

| Challenge | Business Impact |
|---|---|
| No ML-based fraud detection capability | Complex fraud patterns go undetected without an adaptive model |
| No calibrated risk score | Analysts cannot prioritise review queues by risk severity |
| No explainability for flagged transactions | Inability to respond to client disputes or regulatory queries |
| No behavioural baseline per customer | Unusual-for-this-customer transactions treated same as unusual-for-all |
| No regulatory-aligned STR filing workflow | Manual, inconsistent, potentially non-compliant suspicious transaction reporting |
| No fraud score audit log | Cannot demonstrate AML controls to DBS, JP Morgan, or RBI examiners |

## 1.3 Proposed Solution

A Machine Learning–based Transaction Risk Scoring Engine that:

- Assigns every cross-border transaction a calibrated risk score from 0 to 100
- Produces the top 5 contributing SHAP features driving each score (Feature Name, Feature Value, Contribution Score)
- Classifies transactions into three action tiers: Auto-Pass (GREEN), Analyst Review (AMBER), Auto-Hold (RED)
- Learns behavioural baselines per customer, business type, and corridor
- Runs as a batch scoring pipeline (CSV in → CSV out) on the approved V3.1 synthetic dataset
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

1. Validate and profile the existing approved synthetic dataset (V3.1 — 50,800 transactions) through EDA and data quality checks
2. Engineer 37 features across 5 categories: customer, account, transaction, behavioural, risk/regulatory
3. Build and compare three ML models (Logistic Regression, XGBoost, Isolation Forest) against a rule-based baseline
4. Produce a calibrated risk score with top-5 contributing SHAP features for every transaction
5. Build a batch scoring pipeline (CSV in → scored CSV out) demonstrable on the V3.1 dataset
6. Produce a regulatory positioning memo confirming alignment with RBI/PMLA/FEMA obligations
7. Deliver a PoC demonstration to CrossBorderPay CEO and CTO at Week 5

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
| SHAP explanations are interpretable by a non-technical fraud analyst | Confirmed in demo |
| Model card is complete and honest about dataset constraints | Signed off |
| Regulatory positioning memo completed | Signed off |

## 1.7 Executive Benefits

### For the CEO
- Demonstrates the platform's risk management sophistication to DBS Bank and JP Morgan
- Creates a productisable fraud scoring capability — competitive differentiator
- Reduces fraud-driven NOSTRO position losses
- Strengthens the RBI Innovation Hub and NPCI relationships

### For the CTO
- Establishes the foundational ML pipeline: data validation → feature engineering → model → explainability → scoring
- Produces a reproducible, version-controlled ML codebase that can be extended to a production API (Phase 3)
- Creates audit-ready output suitable for PMLA compliance review
- Delivers the MLOps foundation for continuous model improvement

### For DBS and JP Morgan Partners
- Provides a shared risk intelligence narrative that strengthens the correspondent banking relationship
- Demonstrates that the platform's transaction screening is developing toward bank-grade fraud controls

---

# Section 2: Business Context

## 2.1 CrossBorderPay Business Model

CrossBorderPay () is a cross-border Account-to-Account (A2A) payment platform incorporated as cross-border payments company (, Bengaluru). The platform enables Indian businesses to send and receive international payments via SWIFT, Fedwire, CHIPS, SEPA, and CHAPS rails. NOSTRO pre-funded accounts with DBS Bank and JP Morgan enable fast FX settlement. The company is RBI Innovation Hub incubated and NPCI recognised.

## 2.2 Cross-Border Payment Flow

```
Payment Flow (Current State → Target State with ML Scoring)

Indian Business Sender → CrossBorderPay Platform → Risk Scoring Engine → Decision:
  • Score < 40 (GREEN):  Auto-Pass → Payment Processing → NOSTRO (DBS/JP Morgan)
                          → SWIFT/Fedwire/CHIPS/SEPA/CHAPS → Overseas Beneficiary
  • Score 40–79 (AMBER): Analyst Review Queue → Approve → Payment Processing
                          / Reject → STR Filing with FIU-IND
  • Score ≥ 80 (RED):    Auto-Hold → Analyst Review Queue → Investigation
```

## 2.3 Customer Segments

| Segment | Share | Primary Corridor | Typical Amount | Fraud Risk | Key Fraud Typology |
|---|---|---|---|---|---|
| Exporters | 25% | India → USA, UK, UAE | USD 10K–200K | Medium | TBML over-invoicing, phantom shipments |
| Importers | 20% | USA, China, UAE → India | USD 5K–200K | High | Under-invoicing, advance fee fraud |
| IT Services | 20% | India → USA, UK, Singapore | USD 20K–200K | Low–Medium | Salary diversion, ATO |
| EdTech | 15% | India → Global | USD 500–50K | Medium | Refund fraud, fake enrollments |
| SaaS | 10% | India → Global | USD 1K–100K | Low | Purpose code misdeclaration |
| SME | 10% | India → UAE, UK | USD 1K–50K | High | Hawala mirror, structuring |

## 2.4 Risk Landscape

**FEMA:** Every outward remittance must be for a declared purpose under LRS or specific business purpose codes. Misdeclaration is a FEMA violation. Fraud that exploits purpose code flexibility is endemic.

**PMLA:** the platform's AD banking partners (DBS, JP Morgan) are Reporting Entities under PMLA. Suspicious transactions must be reported to FIU-IND via STR within prescribed timelines.

**RBI Master Directions:** Cross-border payments are subject to transaction limits, documentation requirements, and reporting thresholds that fraudsters actively exploit through structuring.

## 2.5 Fraud Typologies Covered in This PoC

| # | Typology | Description | Detection Signal |
|---|---|---|---|
| 1 | TBML — Over-invoicing | Export invoice value inflated to extract more FX than goods are worth | Invoice amount vs. GST filing, corridor × business type anomaly |
| 2 | TBML — Under-invoicing | Import invoice understated; balance settled via hawala | Amount significantly below peer group median for corridor |
| 3 | Round-tripping | Funds leave India as business payment, return as FDI via offshore SPV | Corridor + purpose code combination |
| 4 | Hawala mirror transactions | Legitimate-looking outward remittance mirrors an informal inward collection | Velocity + beneficiary patterns across unrelated customers |
| 5 | Account Takeover (ATO) | Compromised credentials lead to fraudulent outward remittances | New beneficiary, amount deviation, off-hours submission |
| 6 | Smurfing / Structuring | Large payment broken into sub-threshold transactions | Velocity burst + amount clustering near reporting thresholds |
| 7 | Dormant Account Reactivation | Long-dormant account suddenly active with high-value transaction | Days since last transaction, amount × dormancy interaction |
| 8 | First-Party Fraud (Purpose Code Abuse) | Business misdeclares purpose code to bypass scrutiny | Purpose code deviation from historical pattern |
| 9 | Salary Diversion | IT services company routes client payment to director's personal offshore account | Beneficiary entity type mismatch |
| 10 | EdTech Refund Fraud | Fake student enrollments generate inward remittances refunded to different accounts | Refund velocity, beneficiary change after inward receipt |

## 2.6 Assumptions

| # | Assumption | Validation Required |
|---|---|---|
| A1 | Synthetic dataset V3.1 is sufficient for PoC model training and evaluation | Confirmed — no real transaction data access at PoC stage |
| A2 | PoC validated against held-out test set from the same V3.1 generator with temporal split | Architectural — no real fraud cases available |
| A3 | CrossBorderPay may have an existing rule-based fraud detection approach — this is unconfirmed. A synthetic rule baseline is used as the PoC comparator regardless. | Must be confirmed with CrossBorderPay (Open Item OI-03) |
| A4 | Payment rail distribution: SWIFT 60%, Fedwire 15%, CHIPS 10%, SEPA 10%, CHAPS 5% | Must be confirmed with CrossBorderPay (Open Item OI-13) |
| A5 | Approved fraud rate for the V3.1 dataset is 0.5% flat across all business types (254 fraud cases from 50,800 transactions) | Approved — governs all model training and evaluation |
| A6 | Explainability outputs (top-5 SHAP features) are for internal analyst use only at PoC stage | Not for client-facing use without regulatory review |

## 2.7 Out of Scope Items

| Item | Reason | Future Phase |
|---|---|---|
| Real transaction data | Not available at PoC stage; requires data sharing agreement | Phase 2 |
| OFAC/SDN sanctions list integration | Third-party data feed and legal review required | Phase 3 |
| FIU-IND STR filing integration | Requires regulatory workflow design and AD bank coordination | Phase 3 |
| Real-time feature store | Infrastructure complexity beyond PoC scope | Phase 3–4 |
| Production MLOps pipeline | Architecture documented; production pipeline built in Phase 3 | Phase 3 |
| Graph database (Neo4j / AWS Neptune) | Network features require graph infrastructure; simulated features removed from scope | Phase 4–5 |
| Network / graph fraud features | Graph analytics deferred; 5-category feature set used instead | Phase 4–5 |
| Real-time API scoring endpoint | Batch scoring only in Phase 1; real-time API is Phase 3–4 deliverable | Phase 3–4 |
| Analyst dashboard | Deferred; dashboard design documented in Section 11 for Phase 3 execution | Phase 3 |
| Business cost model (rupee value) | FP costs not validated at current transaction volumes; deferred to Phase 2 with real data | Phase 2 |
| Client-facing risk score disclosure | Requires RBI/legal review of AI decision disclosure obligations | Phase 3 |

---

# Section 3: PoC Scope

## 3.1 In Scope

| # | Item | Description |
|---|---|---|
| 1 | Approved Synthetic Dataset V3.1 | 50,800 transactions · 500 customers · 780 accounts · 0.5% flat fraud rate · 254 fraud cases · 10 fraud typologies. Beneficiary pool embedded as columns in transactions.csv. Dataset already generated and on disk. |
| 2 | Feature Engineering Pipeline | 37 features across 5 categories: customer, account, transaction, behavioural, risk/regulatory |
| 3 | Rule-Based Baseline System | Minimum viable rule set used as comparison benchmark |
| 4 | ML Model Training (3 models) | Logistic Regression (baseline ML), XGBoost (champion), Isolation Forest (anomaly layer) |
| 5 | Model Evaluation Framework | AUC-PR, Recall, Precision, F1, FDR, FPR |
| 6 | Calibrated Risk Score (0–100) | Probability-calibrated score mapped to 0–100 scale with three action tiers |
| 7 | Top-5 SHAP Features per Transaction | SHAP TreeExplainer — top 5 features by absolute contribution. Output: Feature Name, Feature Value, Contribution Score only. |
| 8 | Batch Scoring Pipeline | Python script: CSV in → feature engineering → XGBoost score + SHAP → scored CSV out |
| 9 | Docker Containerisation | Reproducible container build with pinned dependencies |
| 10 | Data Card | Full documentation of V3.1 dataset assumptions and generation methodology |
| 11 | Regulatory Positioning Memo | One-page RBI/PMLA/FEMA alignment statement |
| 12 | PoC Summary Presentation | 10-slide deck for CEO/CTO demo |

## 3.2 Feature Classification (MoSCoW)

| Feature | Priority | Rationale |
|---|---|---|
| Transaction risk score (0–100) | **Must Have** | Core deliverable |
| Top-5 SHAP features per transaction | **Must Have** | PMLA audit requirement + analyst explainability |
| Three-tier action classification (GREEN/AMBER/RED) | **Must Have** | Operational workflow requirement |
| Behavioural deviation features | **Must Have** | Primary ML signal |
| Rule-based baseline comparison | **Must Have** | Cannot prove ML value without it |
| Batch scoring pipeline (CSV in → CSV out) | **Must Have** | Demo requirement |
| Isolation Forest anomaly detection | **Should Have** | Catches fraud types not in training labels |
| Adversarial test cases (3 scenarios) | **Should Have** | Production readiness signal |
| Regulatory positioning memo | **Should Have** | CEO/board requirement |
| Network / graph features | **Won't Have** (Phase 4–5) | Graph infrastructure not available in PoC |
| Analyst review queue dashboard | **Won't Have** (Phase 3) | Deferred — documented in Section 11 |
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
Customer Features (7) | Account Features (5) | Transaction Features (8)
| Behavioural Features (11) | Risk/Regulatory Features (6)
[Total: 37 features across 5 categories]
[Network/graph features: Deferred to Phase 4–5 — see Section 6.5]

Layer 4 — Model Training Layer
Logistic Regression (Baseline ML) | XGBoost (Champion, scale_pos_weight≈199)
| Isolation Forest (Anomaly) | Rule-Based (Comparator)

Layer 5 — Model Registry
MLflow Experiment Tracking → Model Artifacts (Pickle + ONNX)
→ Model Card Documentation → Version Control

Layer 6 — Batch Scoring Pipeline
Feature Matrix → XGBoost Inference → Score Calibration (0–100)
→ SHAP TreeExplainer (Top-5 Features) → Audit Logger
→ scored_transactions.csv

Layer 7 — Dashboard (Phase 3 — Not in PoC Scope)
[Deferred — see Section 11]

Layer 8 — Sandbox Deployment Layer
Docker Container (Batch Pipeline) | Prometheus Metrics (optional) | Logging (JSONL)
```

## 4.3 Batch Scoring Pipeline Flow

```
Batch Scoring Sequence

1. Load V3.1 dataset (transactions.csv + customers.csv + accounts.csv)
2. Run data validation checks (schema + null rates)
3. Compute 37 features (feature engineering pipeline)
4. Load champion XGBoost model (xgb_model.pkl)
5. XGBoost: Predict(feature_matrix) → Raw probability per transaction
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
                                → SHAP Explainer → Rule Override Engine

Decision Workflow:     Auto-Pass Queue | Analyst Review Queue
                       | Auto-Hold Queue | STR Filing → FIU-IND

Dashboard (Phase 3):   Executive View | Operational View | Analyst View (Streamlit)

MLOps Monitoring:      MLflow Registry | Prometheus/Grafana | EvidentlyAI Drift
                       | Retraining Pipeline Trigger

All infrastructure deployed in AWS ap-south-1 (Mumbai) — India data residency.
```

---

# Section 5: Data Strategy

## 5.1 Dataset Structure (V3.1 — Current Approved Dataset)

> **DATA GOVERNANCE:** The values below are the approved, source-of-truth dataset parameters. No parameter may be increased, rebalanced, or modified without explicit CrossBorderPay approval.

| Dataset | Records | Key Fields | Format | Status |
|---|---|---|---|---|
| customers.csv | 500 | customer_id, business_type, gst_number, kyc_status, annual_turnover_inr, city, incorporation_date, risk_tier | CSV | On disk |
| accounts.csv | 780 | account_id, customer_id, account_type (Current/EEFC/OD), currency, opened_date, status, od_limit | CSV | On disk |
| transactions.csv | 50,800 | transaction_id, customer_id, account_id, amount_usd, currency, corridor, purpose_code, payment_rail, beneficiary_id, submission_timestamp, status, is_fraud, fraud_typology + beneficiary pool columns | CSV | On disk |

> **Note:** There is no standalone beneficiaries.csv in V3.1. Beneficiary data (beneficiary_id, beneficiary_country, beneficiary_bank_bic, beneficiary_entity_type, first_seen_date) is embedded as columns in transactions.csv. A standalone CrossBorderPay_beneficiaries.csv is a future deliverable requiring explicit CrossBorderPay approval before generation.

> **Note:** There is no separate fraud_labels.csv. The `is_fraud` label and `fraud_typology` field are columns within transactions.csv.

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
| Fraud typologies | 10 (as defined in Section 2.5) |
| Time span | **Pending Confirmation from CrossBorderPay** — verify via `df['submission_timestamp'].min()` / `.max()` on loaded data |

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

## 6.5 Network Features — Deferred to Phase 4–5

> **Network features have been removed from Phase 1 PoC scope.** Graph analytics require dedicated graph database infrastructure (AWS Neptune) that is not available in the PoC environment. The full network feature set is a Phase 4–5 deliverable.
>
> **Features deferred:** shared_beneficiary_graph_score, fraud_ring_signal, community_detection_score, hawala_mirror_flag, bic_network_score, lei_network_flag, network_centrality_score.
>
> See Section 19 (Future Roadmap) for Phase 4–5 network feature implementation plan.

## 6.6 Risk and Regulatory Features

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
  [Logistic Regression | XGBoost | Isolation Forest | Rule-Based Baseline]
→ Model Comparison Report → Champion Selection
→ Recommended: XGBoost (primary) + Isolation Forest (secondary signal)
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

### Model 3: XGBoost (Champion)

| Attribute | Detail |
|---|---|
| Purpose | Primary production model candidate — best balance of performance, speed, and explainability |
| Advantages | State-of-the-art on tabular fraud data; handles imbalanced classes via scale_pos_weight; fast inference; excellent SHAP support |
| Limitations | More hyperparameters than LR; requires careful tuning to avoid overfitting |
| Explainability | SHAP TreeExplainer — fast, exact SHAP values; top-5 features per transaction |
| Expected Performance | Recall ~82%, Precision ~74%, AUC-PR ~0.79 |
| Role in PoC | Champion model; primary scoring model in batch pipeline |
| Class Imbalance | **scale_pos_weight = n_negative / n_positive = 50,546 / 254 ≈ 199** |
| Hyperparameters | n_estimators, max_depth, learning_rate, scale_pos_weight=199, subsample, colsample_bytree, min_child_weight |

### Model 4: Isolation Forest (Anomaly Detection Layer)

| Attribute | Detail |
|---|---|
| Purpose | Unsupervised anomaly detection — catches novel fraud patterns not in training labels |
| Advantages | Does not require fraud labels; catches emerging fraud typologies; complements supervised model |
| Limitations | Cannot be calibrated to a probability; false positive rate harder to control |
| Expected Performance | Recall ~55% on labelled fraud; catches ~20% of fraud not detected by XGBoost |
| Role in PoC | Secondary signal — elevates XGBoost score when Isolation Forest also flags the transaction |

## 7.3 Champion Model Selection Strategy

**Recommended:** XGBoost (primary) + Isolation Forest (secondary signal)

1. Train all three ML models and rule baseline on training set
2. Evaluate all on validation set using AUC-PR as primary metric
3. Confirm XGBoost as champion if AUC-PR ≥ 5% better than Logistic Regression on validation set
4. Run Isolation Forest as parallel layer; use anomaly score as additional feature in final scoring
5. Deploy champion model; document all results in model comparison report

**Threshold Selection:** Do NOT use default 0.5 threshold. Use F1-maximising threshold on validation set. Document precision-recall curve at multiple thresholds. Recommend operational threshold to CrossBorderPay for final approval (Open Item OI-08).

## 7.4 Class Imbalance Handling

| Technique | Applied To | Rationale |
|---|---|---|
| scale_pos_weight ≈ 199 in XGBoost | Training | Native XGBoost imbalance handling; critical at 0.5% fraud rate |
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

## 8.2 Business Cost Model

> **Not in scope for Phase 1 PoC.**
>
> Assumptions about the cost of blocking a legitimate transaction (e.g., USD 250 per event) are not validated at current CrossBorderPay transaction volumes and have not been confirmed by the business. A business cost model will be developed in Phase 2 using real transaction data and confirmed FP/FN cost inputs from CrossBorderPay leadership. See Section 19 Future Roadmap and Open Item OI-08.

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

| Metric | Rule Baseline | Logistic Regression | XGBoost | Target |
|---|---|---|---|---|
| AUC-PR | TBD | TBD | TBD | ≥ 0.70 |
| Recall @ 5% FPR | TBD | TBD | TBD | ≥ 75% |
| Precision @ threshold | TBD | TBD | TBD | ≥ 60% |
| F1 Score | TBD | TBD | TBD | ≥ 0.66 |
| Batch inference time (50K rows) | < 1s | < 10s | < 60s | < 5 min |
| Explainability | Rule fired | Coefficient | SHAP top-5 | Full |

*TBD values to be populated during model training in Week 3.*

---

# Section 9: Explainability Framework

## 9.1 Explainability Architecture

```
Transaction Feature Vector
→ XGBoost Model (Predict Probability)
→ Score Calibration (0–100 scale)
→ Score Tier Assignment (GREEN / AMBER / RED)
→ SHAP TreeExplainer (Compute exact SHAP values for all features)
→ Rank Features by |SHAP contribution|
→ Select Top 5 Features
→ Audit Logger (JSONL — PMLA compliance)
→ Batch Output Row (CSV)
```

## 9.2 Top-5 SHAP Output Specification

Each scored transaction outputs the top 5 features by absolute SHAP contribution.

**Format: Feature Name · Feature Value · Contribution Score only. No reason codes. No plain-language narratives.**

| Output Column | Content | Data Type |
|---|---|---|
| f1_name | Name of feature with highest \|SHAP\| contribution | String |
| f1_value | Actual feature value for this transaction | Float |
| f1_contribution | SHAP contribution (positive = pushes toward fraud; negative = pushes toward legitimate) | Float |
| f2_name | Feature with 2nd highest \|SHAP\| | String |
| f2_value | Actual value | Float |
| f2_contribution | SHAP contribution | Float |
| f3_name through f5_name | Features 3–5 by SHAP rank | String |
| f3_value through f5_value | Actual values | Float |
| f3_contribution through f5_contribution | SHAP contributions | Float |

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

> **PMLA Audit Requirement:** Every scored row — including all 5 SHAP features — is written to both the output CSV and the JSONL audit log. This satisfies the requirement to log every fraud score plus the features used per decision.

## 9.4 Explainability Use Cases

| Audience | Format | Content |
|---|---|---|
| Fraud Analyst | Top-5 features in batch output CSV | Feature name and value allow analyst to understand what drove the score |
| PMLA Audit Log | JSONL audit record per transaction | Full SHAP values + risk score + tier + model version — structured for regulatory examination |

---

# Section 10: Batch Scoring Pipeline

## 10.1 Pipeline Overview

The batch scoring pipeline replaces a real-time API for Phase 1 PoC. It processes the full V3.1 dataset (or any subset) offline and produces a scored output CSV with risk scores and SHAP explainability. Real-time API scoring is a Phase 3–4 deliverable.

| Attribute | Detail |
|---|---|
| Input | transactions.csv (V3.1 — 50,800 rows) + feature-engineered matrix |
| Output | scored_transactions.csv (50,800 rows with risk score, tier, top-5 SHAP) |
| Execution | Python script: `src/scoring/batch_scorer.py` or `notebooks/06_batch_scoring.ipynb` |
| Runtime | Target: complete full dataset in < 5 minutes on standard developer hardware |
| Model | XGBoost champion (xgb_model.pkl) + Isolation Forest (if_model.pkl) |
| Explainability | SHAP TreeExplainer (exact SHAP values) |

## 10.2 Pipeline Configuration

| Parameter | Value |
|---|---|
| Input file | `data/processed/feature_matrix.parquet` |
| Model path | `models/xgb_model.pkl` |
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
| GREEN | 0–39 | Auto-Pass — payment proceeds without analyst intervention |
| AMBER | 40–79 | Analyst Review — payment queued for human decision |
| RED | 80–100 | Auto-Hold — payment held pending analyst investigation |

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

# Section 11: Dashboard Design

## 11.1 Dashboard Status

> **PHASE 3 — NOT IN SCOPE FOR PHASE 1 PoC**
>
> Dashboard design is deferred to Phase 3 of the ML roadmap. The Phase 1 PoC deliverable is a batch scoring pipeline with SHAP explainability output, not an interactive dashboard.
>
> The design below documents the intended Phase 3 dashboard for planning purposes. No dashboard code will be written in Phase 1.

## 11.2 Dashboard Architecture (Phase 3 Design)

```
Scoring Pipeline Audit Logs → Log Aggregator (JSONL Files) → Streamlit Dashboard App
  ├── Executive View   (Portfolio Risk — GREEN background)
  ├── Operational View (Queue Management — AMBER background)
  └── Analyst View     (Transaction Detail — RED background)
```

## 11.3 Executive View (Phase 3)

**Audience:** CEO, Head of Risk, Board presentations

| Panel | Chart Type | Content |
|---|---|---|
| Risk Score Distribution | Histogram | Distribution of all scores in period |
| Fraud Detected (₹ value) | KPI card | Estimated fraud value caught this week/month |
| False Positive Rate | KPI card | % of legitimate transactions incorrectly flagged |
| Top Fraud Corridors | Bar chart | Risk score average by corridor |
| Fraud by Business Type | Donut chart | % of fraud cases by customer segment |
| Monthly Trend — Fraud Rate | Line chart | Trend of fraud detections over time |
| Model vs. Rule Baseline | Side-by-side bar | Recall comparison — ML vs. rule system |
| Top Risk Features | Bar chart | Most frequent top-SHAP features across flagged transactions |

## 11.4 Fraud Analyst View (Phase 3)

**Audience:** Fraud Analysts, Compliance Officers

| Panel | Content |
|---|---|
| Transaction Detail | Full transaction record with all feature values |
| Risk Score Card | Score (0–100), tier, model version |
| SHAP Feature Panel | Top-5 features with name, value, and contribution score |
| Customer Risk Profile | Customer 90-day behavioural baseline for context |
| Beneficiary History | Previous transactions to this beneficiary |
| SHAP Waterfall Chart | Feature-by-feature contribution to score |
| Analyst Decision Capture | Approve / Reject / Escalate buttons with mandatory note field |

---

# Section 12: Sandbox Deployment Plan

## 12.1 Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| Batch Scoring Pipeline | Python 3.11 + XGBoost + SHAP | Core PoC deliverable |
| Model Artifacts | MLflow + local pickle/ONNX | Model versioning and artifact storage |
| Containerisation | Docker + Docker Compose | Reproducible, portable deployment |
| Logging | Python logging → JSONL files | Structured audit log (PMLA) |
| Feature Store (PoC) | Pre-computed Parquet files | Behavioural baselines computed from V3.1 |
| Notebook Environment | Jupyter | EDA, feature engineering, model training |

## 12.2 Folder Structure

```
CrossBorderPay-fraud-poc/
├── data/
│   ├── raw/                     # V3.1 CSVs (customers, accounts, transactions)
│   ├── processed/               # Cleaned, feature-engineered data (Parquet)
│   └── output/                  # scored_transactions.csv + audit_log.jsonl
├── notebooks/
│   ├── 01_data_validation.ipynb
│   ├── 02_eda.ipynb
│   ├── 03_feature_engineering.ipynb
│   ├── 04_model_training.ipynb
│   ├── 05_model_evaluation.ipynb
│   └── 06_batch_scoring.ipynb
├── src/
│   ├── data/
│   │   ├── validator.py
│   │   └── loader.py
│   ├── features/
│   │   ├── customer_features.py
│   │   ├── account_features.py
│   │   ├── transaction_features.py
│   │   ├── behavioral_features.py
│   │   └── risk_features.py
│   ├── models/
│   │   ├── baseline_rules.py
│   │   ├── logistic_regression.py
│   │   ├── xgboost_model.py
│   │   └── isolation_forest.py
│   ├── scoring/
│   │   ├── batch_scorer.py
│   │   ├── calibration.py
│   │   └── shap_explainer.py
│   └── utils/
│       └── metrics.py
├── tests/
│   ├── unit/
│   └── adversarial/
│       └── test_adversarial_cases.py
├── deployment/
│   ├── Dockerfile
│   └── docker-compose.yml
├── docs/
│   ├── data_card.md
│   ├── model_card.md
│   └── regulatory_memo.md
├── requirements.txt
├── .env.example
└── README.md
```

## 12.3 Monitoring and Logging

| Signal | Mechanism | Alert Threshold |
|---|---|---|
| Score distribution shift | Custom Python check on each pipeline run | Alert if mean score shifts > 10 points vs. previous run |
| Fraud rate in scored output | Post-run check | Alert if scored fraud rate < 0.3% (model may have failed) |
| SHAP coverage | Post-run check | Alert if any row missing SHAP columns |
| Feature null rate | Pre-scoring check | Alert if > 5% of features null for any transaction |
| Audit log completeness | Post-run row count | Alert if audit_log rows ≠ output CSV rows |

---

# Section 13: Delivery Plan

## 13.1 Plan Overview

| Field | Detail |
|---|---|
| Total Duration | 6 Weeks (Week 0 pre-work + Weeks 1–5) |
| Start Condition | All P1 Open Items resolved (Section 16) |
| End Condition | PoC demo delivered to CEO and CTO; all deliverables signed off |
| Total Estimated Effort | ~170 hours across 6 weeks |

## 13.2 Week-by-Week Execution Plan

### Week 0 — Prerequisites and Open Question Resolution

| Field | Detail |
|---|---|
| Objective | Resolve all P1 open questions before any engineering work begins |
| Activities | Schedule kickoff with CrossBorderPay CTO/Head of Risk; present Open Items Register; collect answers; confirm data access decision; confirm existence of any current fraud detection approach |
| Deliverables | Completed Open Items Register (P1 items); Project kickoff sign-off; Confirmed PoC scope |
| Dependencies | CrossBorderPay stakeholder availability; legal/compliance input on OI-04, OI-05, OI-06 |
| Risks | Open questions not resolved → delays Week 1 start |
| Acceptance Criteria | OI-01, OI-03, OI-07, OI-08, OI-09 confirmed; Start date confirmed |
| Estimated Effort | 8 hours (meetings + documentation) |

### Week 1 — Data Validation and EDA

| Field | Detail |
|---|---|
| Objective | Validate V3.1 dataset quality; understand its characteristics; establish temporal split boundaries |
| Activities | Load V3.1 CSVs; run Great Expectations validation suite; check schema + null rates; determine time span (submission_timestamp min/max); run label leakage smoke test (3-feature LR — AUC must be < 0.90); EDA: amount distributions, fraud typology breakdown, corridor patterns, business mix confirmation; define temporal split boundary dates; build rule-based baseline (5–7 rules); document V3.1 data profile |
| Deliverables | Data Validation Report; EDA Notebook (02_eda.ipynb); Confirmed time span; Temporal split boundaries; Label leakage smoke test result; Rule-Based Baseline Code; Baseline Benchmark Metrics |
| Dependencies | V3.1 files on disk (already present); Week 0 complete |
| Risks | Label leakage detected → return to dataset review before proceeding; time span shorter than expected |
| Acceptance Criteria | Dataset passes all Great Expectations checks; 254 fraud cases confirmed; LR smoke test AUC < 0.90; rule baseline recall documented; time span confirmed |
| Estimated Effort | 25 hours |

### Week 2 — Feature Engineering Pipeline

| Field | Detail |
|---|---|
| Objective | Build complete feature engineering pipeline producing 37 features from V3.1 |
| Activities | Implement all 5 feature category modules; compute rolling windows (1h, 24h, 7d, 30d, 90d); compute peer group percentiles; run EDA and correlation analysis; generate train/validation/test temporal split; verify feature null rates |
| Deliverables | Feature Engineering Pipeline (5 Python modules); Feature Matrix — train/val/test splits (Parquet); Feature Correlation Analysis |
| Dependencies | Week 1 dataset validated; temporal split boundaries confirmed |
| Risks | Rolling window features computationally slow on 50K rows; NaN handling edge cases |
| Acceptance Criteria | Pipeline produces 37 features with < 5% null rate; temporal split verified; feature modules pass unit tests |
| Estimated Effort | 35 hours |

### Week 3 — Model Training, Comparison, and Champion Selection

| Field | Detail |
|---|---|
| Objective | Train all three ML models + rule baseline; compare; select champion |
| Activities | Train Logistic Regression (class_weight='balanced'); train XGBoost (scale_pos_weight≈199, hyperparameter tuning); train Isolation Forest; compare all on validation set using AUC-PR; calibrate probability scores; tune operating threshold (F1-maximising on val set); document model comparison report; select champion (expected: XGBoost) |
| Deliverables | Trained Model Artifacts (LR + XGBoost + IF — pickle + ONNX); Model Comparison Report; Champion Model Selected; Threshold Recommendation for CrossBorderPay |
| Dependencies | Week 2 feature matrix; Week 1 baseline metrics |
| Risks | XGBoost overfitting on 178 training fraud cases; insufficient fraud cases in validation fold (51) for stable threshold selection |
| Acceptance Criteria | Champion model AUC-PR ≥ 0.70 on validation set; ML model recall ≥ 15% higher than rule baseline; all MLflow runs logged with reproducible seeds |
| Estimated Effort | 40 hours |

### Week 4 — Explainability Layer, Batch Pipeline, and Adversarial Testing

| Field | Detail |
|---|---|
| Objective | Build SHAP explainability; build batch scoring pipeline; run adversarial test cases; containerise |
| Activities | Implement SHAP TreeExplainer (exact SHAP values); build top-5 feature extraction and ranking; build batch_scorer.py; generate scored_transactions.csv from V3.1; write JSONL audit log; run 3 adversarial test cases; write Dockerfile; write docker-compose.yml; verify full pipeline runs in container |
| Deliverables | SHAP Explainability Module; batch_scorer.py; scored_transactions.csv (V3.1 full output); audit_log.jsonl; Adversarial Test Report (3 cases); Docker image; docker-compose.yml |
| Dependencies | Week 3 champion model selected and calibrated |
| Risks | SHAP computation slow on 50K rows → use approximate SHAP or batch in chunks; adversarial cases reveal threshold needs adjustment |
| Acceptance Criteria | Batch pipeline produces scored_transactions.csv with all 50,800 rows; 100% SHAP coverage; audit log row count matches output; all 3 adversarial scenarios documented; docker-compose up produces working pipeline |
| Estimated Effort | 40 hours |

### Week 5 — Documentation, Demo Preparation, and Final Presentation

| Field | Detail |
|---|---|
| Objective | Complete all documentation; prepare and deliver PoC demonstration to CrossBorderPay CEO and CTO |
| Activities | Write model card; complete data card; write regulatory positioning memo; prepare demo dataset (30 transactions with 4 scenarios); build 10-slide PoC summary presentation; rehearse demo flow (3 run-throughs); prepare Q&A document; deliver demo; collect feedback; update production roadmap |
| Deliverables | Model Card; Data Card; Regulatory Positioning Memo; Demo Dataset (30 transactions); 10-Slide PoC Presentation; Live Batch Pipeline Demo; Q&A Document; Post-Demo Feedback Report |
| Dependencies | Week 4 complete; CrossBorderPay CEO/CTO availability confirmed |
| Risks | Demo environment fails → pre-run output CSV as backup; regulatory memo requires legal input |
| Acceptance Criteria | Batch pipeline demo runs without errors; SHAP explanations reviewed by Head of Risk; CEO and CTO acknowledge PoC objectives met; next steps agreed |
| Estimated Effort | 22 hours |

---

# Section 14: Work Breakdown Structure

| WBS ID | Task | Subtask | Owner | Start Wk | End Wk | Status | Dependency | Deliverable |
|---|---|---|---|---|---|---|---|---|
| **1.0** | **PROJECT INITIATION** | | PM | 0 | 0 | Not Started | — | — |
| 1.1 | Project Initiation | Stakeholder kickoff meeting | PM | 0 | 0 | Not Started | — | Meeting minutes |
| 1.2 | Project Initiation | Open Questions resolution | PM + DS | 0 | 0 | Not Started | 1.1 | Completed OI Register |
| 1.3 | Project Initiation | PoC Plan sign-off | PM | 0 | 0 | Not Started | 1.2 | Signed PoC Plan |
| **2.0** | **DATA VALIDATION & EDA** | | DS | 1 | 1 | Not Started | 1.3 | — |
| 2.1 | Data Validation | Load V3.1 CSVs + schema check | DS | 1 | 1 | Not Started | 1.3 | Validation Report |
| 2.2 | Data Validation | Determine time span (min/max timestamp) | DS | 1 | 1 | Not Started | 2.1 | Confirmed time span |
| 2.3 | Data Validation | Great Expectations suite | DS | 1 | 1 | Not Started | 2.1 | GE Report |
| 2.4 | Data Validation | Label leakage smoke test (LR, AUC < 0.90) | DS | 1 | 1 | Not Started | 2.3 | Smoke Test Result |
| 2.5 | EDA | Distribution analysis + fraud breakdown | DS | 1 | 1 | Not Started | 2.3 | EDA Notebook |
| 2.6 | EDA | Temporal split boundary definition | DS | 1 | 1 | Not Started | 2.2, 2.5 | Split boundaries |
| 2.7 | EDA | V3.1 data profile document | DS | 1 | 1 | Not Started | 2.5 | data_card.md (draft) |
| **3.0** | **RULE-BASED BASELINE** | | DS + FR | 1 | 1 | Not Started | 2.3 | — |
| 3.1 | Baseline | Rule set design (5–7 rules) | FR | 1 | 1 | Not Started | 2.3 | Rule Specification |
| 3.2 | Baseline | Rule engine implementation | DS | 1 | 1 | Not Started | 3.1 | baseline_rules.py |
| 3.3 | Baseline | Baseline benchmark metrics | DS | 1 | 1 | Not Started | 3.2 | Baseline Metrics Report |
| **4.0** | **FEATURE ENGINEERING** | | DS | 2 | 2 | Not Started | 2.0 | — |
| 4.1 | Feature Engineering | Customer features module | DS | 2 | 2 | Not Started | 2.3 | customer_features.py |
| 4.2 | Feature Engineering | Account features module | DS | 2 | 2 | Not Started | 2.3 | account_features.py |
| 4.3 | Feature Engineering | Transaction features module | DS | 2 | 2 | Not Started | 2.3 | transaction_features.py |
| 4.4 | Feature Engineering | Behavioural features (rolling windows) | DS | 2 | 2 | Not Started | 2.3 | behavioral_features.py |
| 4.5 | Feature Engineering | Risk/regulatory features | DS + FR | 2 | 2 | Not Started | 2.3 | risk_features.py |
| 4.6 | Feature Engineering | Feature correlation analysis | DS | 2 | 2 | Not Started | 4.1–4.5 | Correlation Report |
| 4.7 | Feature Engineering | Train/val/test temporal split | DS | 2 | 2 | Not Started | 4.6 | Split datasets (Parquet) |
| **5.0** | **MODEL TRAINING** | | DS | 3 | 3 | Not Started | 4.0 | — |
| 5.1 | Model Training | Logistic Regression training | DS | 3 | 3 | Not Started | 4.7 | lr_model.pkl |
| 5.2 | Model Training | XGBoost training + tuning (scale_pos_weight≈199) | DS | 3 | 3 | Not Started | 4.7 | xgb_model.pkl |
| 5.3 | Model Training | Isolation Forest training | DS | 3 | 3 | Not Started | 4.7 | if_model.pkl |
| 5.4 | Model Training | Score calibration | DS | 3 | 3 | Not Started | 5.2 | calibration.pkl |
| 5.5 | Model Training | Threshold tuning | DS | 3 | 3 | Not Started | 5.4 | Threshold Report |
| 5.6 | Model Training | Model comparison report | DS | 3 | 3 | Not Started | 5.1–5.5 | Model Comparison Report |
| 5.7 | Model Training | Champion model selection | DS + CTO | 3 | 3 | Not Started | 5.6 | Decision Memo |
| **6.0** | **EXPLAINABILITY** | | DS | 4 | 4 | Not Started | 5.7 | — |
| 6.1 | Explainability | SHAP TreeExplainer integration | DS | 4 | 4 | Not Started | 5.7 | shap_explainer.py |
| 6.2 | Explainability | Top-5 feature extraction and ranking | DS | 4 | 4 | Not Started | 6.1 | Extraction module |
| 6.3 | Explainability | PMLA audit log writer | DS | 4 | 4 | Not Started | 6.2 | audit_log.jsonl |
| **7.0** | **BATCH SCORING PIPELINE** | | DS | 4 | 4 | Not Started | 5.7 | — |
| 7.1 | Batch Pipeline | batch_scorer.py implementation | DS | 4 | 4 | Not Started | 5.7, 6.1 | batch_scorer.py |
| 7.2 | Batch Pipeline | Full V3.1 scoring run | DS | 4 | 4 | Not Started | 7.1 | scored_transactions.csv |
| 7.3 | Batch Pipeline | Output validation (row count, SHAP coverage) | DS | 4 | 4 | Not Started | 7.2 | Validation Report |
| **8.0** | **ADVERSARIAL TESTING** | | DS + FR | 4 | 4 | Not Started | 7.0 | — |
| 8.1 | Adversarial | Legitimate large export test | DS + FR | 4 | 4 | Not Started | 7.0 | Adversarial Report |
| 8.2 | Adversarial | Dormant ATO reactivation test | DS + FR | 4 | 4 | Not Started | 7.0 | Adversarial Report |
| 8.3 | Adversarial | Smurfing sequence detection test | DS + FR | 4 | 4 | Not Started | 7.0 | Adversarial Report |
| **9.0** | **CONTAINERISATION** | | DS | 4 | 4 | Not Started | 7.0 | — |
| 9.1 | Docker | Dockerfile (batch pipeline) | DS | 4 | 4 | Not Started | 7.0 | Dockerfile |
| 9.2 | Docker | docker-compose.yml | DS | 4 | 4 | Not Started | 9.1 | docker-compose.yml |
| **10.0** | **DOCUMENTATION** | | DS + PM + FR | 5 | 5 | Not Started | 9.0 | — |
| 10.1 | Documentation | Model card (complete) | DS | 5 | 5 | Not Started | 5.7 | model_card.md |
| 10.2 | Documentation | Data card (complete) | DS | 5 | 5 | Not Started | 2.7 | data_card.md |
| 10.3 | Documentation | Regulatory positioning memo | PM + FR | 5 | 5 | Not Started | CrossBorderPay legal | regulatory_memo.md |
| 10.4 | Testing | End-to-end integration test | DS | 5 | 5 | Not Started | 9.0 | Integration Test Report |
| **11.0** | **DEMO AND PRESENTATION** | | PM + DS | 5 | 5 | Not Started | 10.0 | — |
| 11.1 | Demo | Demo dataset preparation (30 transactions, 4 scenarios) | DS | 5 | 5 | Not Started | 7.2 | Demo Dataset |
| 11.2 | Demo | 10-slide presentation build | PM | 5 | 5 | Not Started | 10.0 | Presentation |
| 11.3 | Demo | Demo rehearsal (×3) | All | 5 | 5 | Not Started | 11.1 | Rehearsal Sign-off |
| 11.4 | Demo | Q&A document preparation | All | 5 | 5 | Not Started | 11.2 | Q&A Document |
| 11.5 | Demo | Live PoC Demo to CEO/CTO | All | 5 | 5 | Not Started | 11.3 | Demo Recording |
| 11.6 | Demo | Post-demo feedback and next steps | PM | 5 | 5 | Not Started | 11.5 | Feedback Report |

*Owner Codes: DS = Data Scientist, FR = Fraud Risk SME, PM = Product/Program Manager, CTO = CrossBorderPay CTO*

---

# Section 15: RAID Log

## 15.1 Risks

| ID | Risk | Likelihood | Impact | Severity | Mitigation | Owner |
|---|---|---|---|---|---|---|
| R-01 | Label leakage from V3.1 synthetic data produces artificially inflated metrics | Medium | Critical | Critical | Run LR smoke test immediately after loading V3.1; AUC > 0.90 on 3-feature LR triggers dataset review before any model training | DS |
| R-02 | V3.1 has only 254 fraud cases — small positive class may result in poor model performance (178 training fraud cases) | High | High | High | scale_pos_weight≈199 for XGBoost; class_weight='balanced' for LR; SMOTE if AUC-PR < 0.65 after threshold tuning; if performance insufficient, escalate to Phase 2 dataset expansion | DS |
| R-03 | Open questions not resolved before Week 1, delaying start | Medium | High | High | Schedule Week 0 kickoff; escalate if no CrossBorderPay response within 5 business days | PM |
| R-04 | XGBoost overfits 178 training fraud cases | Medium | High | High | Stratified k-fold CV, subsample/colsample regularisation, early stopping, min_child_weight tuning | DS |
| R-05 | SHAP computation over 50,800 rows exceeds acceptable batch runtime | Low | Medium | Medium | Switch to approximate SHAP (model.predict with shap_approximate=True); batch in chunks of 5,000 rows | DS |
| R-06 | Regulatory memo cannot be completed without CrossBorderPay legal input | High | Medium | High | Draft memo with known facts; clearly flag sections requiring legal confirmation before production use | PM + FR |
| R-07 | Demo environment fails during live CEO/CTO presentation | Medium | Critical | High | Pre-run scored_transactions.csv as backup; show static output if pipeline fails live; rehearse 3× | DS |
| R-08 | Adversarial test cases reveal model incorrectly blocks legitimate transactions | Medium | High | High | Adversarial tests are diagnostic; if model fails, document and adjust threshold recommendation | DS + FR |
| R-09 | the platform's actual fraud typologies differ from V3.1 synthetic design | High | High | High | Anchor design to FIU-IND/FATF published typologies; validate with Head of Risk in Week 0 (OI-07) | FR |
| R-10 | Docker build failures on target environment | Low | Medium | Medium | Test builds on Linux and Windows; document minimum requirements | DS |
| R-11 | Scope creep — CrossBorderPay requests features beyond agreed scope | Medium | Medium | Medium | Redirect all scope additions to Future Roadmap with formal change control | PM |
| R-12 | V3.1 time span is shorter than expected, leaving insufficient transactions for stable temporal split | Medium | High | High | Verify time span in Week 1 (OI-16); if too short, evaluate whether split can still produce valid holdout | DS |

## 15.2 Assumptions

| ID | Assumption | Basis | Validation Required By | Impact if Wrong |
|---|---|---|---|---|
| A-01 | V3.1 synthetic dataset is sufficient for PoC evaluation | No real data access at PoC stage; V3.1 already approved and on disk | Confirmed | Minimal — V3.1 is the agreed baseline |
| A-02 | XGBoost is the preferred primary model (performance priority) | Committee recommendation; confirmed in project context | Week 3 | Switch to Logistic Regression if explainability constraints dominate |
| A-03 | Batch scoring pipeline will complete 50,800 transactions in < 5 minutes on standard developer hardware | Estimated based on XGBoost batch inference speed | Week 4 | If slower, optimise with chunk-based processing or approximate SHAP |
| A-04 | SWIFT is the dominant payment rail (60%+) | India cross-border payments context | Week 0 (confirm OI-13) | Adjust V3.1 rail distribution assumptions in model features |
| A-05 | Approved fraud rate is 0.5% flat across all business types — producing 254 fraud cases from 50,800 transactions | Formally approved and governs all training and evaluation | Confirmed | No change permitted without CrossBorderPay approval |
| A-06 | CrossBorderPay may have no existing fraud detection system — PoC builds capability from scratch regardless | Open Item OI-03 | Week 0 | If an existing system is confirmed, use it as the rule baseline comparator instead of the synthetic one |

## 15.3 Issues

| ID | Issue | Raised | Severity | Resolution | Owner |
|---|---|---|---|---|---|
| I-01 | No confirmed real fraud cases available for validation | Week 0 | High | Synthetic-only PoC; flag as limitation in model card explicitly | DS |
| I-02 | the platform's current fraud detection baseline metrics unknown | Week 0 | High | Estimate using industry benchmarks if not provided; confirm in Open Items | FR |
| I-03 | Regulatory memo requires legal sign-off which may not be available in 6 weeks | Week 5 | Medium | Draft memo; label clearly as "pending legal review" | PM |

## 15.4 Dependencies

| ID | Dependency | External? | Owner | Due |
|---|---|---|---|---|
| D-01 | CrossBorderPay Open Items P1 responses | Yes — CrossBorderPay | PM | Week 0 |
| D-02 | Python 3.11 environment with agreed library versions | No | DS | Week 1 |
| D-03 | MLflow instance available for experiment tracking | No | DS | Week 1 |
| D-04 | FATF country risk scores (public data) | Partially external | DS | Week 1 |
| D-05 | CrossBorderPay CTO/CEO availability for Week 5 demo | Yes — CrossBorderPay | PM | Week 3 |
| D-06 | CrossBorderPay legal/compliance review for regulatory memo | Yes — CrossBorderPay | PM + FR | Week 4 |
| D-07 | CrossBorderPay confirmation of operating threshold (FP vs FN tolerance) | Yes — CrossBorderPay | PM | Week 3 |

---

# Section 16: Open Items Register

| ID | Open Question | Owner | Priority | Status | Decision Required By | Impact if Unanswered |
|---|---|---|---|---|---|---|
| OI-01 | Does CrossBorderPay have any historical confirmed fraud cases (even 20–30 anonymised examples) that can be used to validate whether synthetic fraud patterns are realistic? | CrossBorderPay Head of Risk | P1 — Critical | Open | Week 0 | PoC proceeds on synthetic-only basis; model card explicitly flags this as a limitation; validation is circular |
| OI-02 | Is there any access to DBS or JP Morgan's existing transaction risk signals, even in anonymised or aggregated form? | CrossBorderPay CTO | P1 — Critical | Open | Week 0 | PoC proceeds without partner data; significantly reduces commercial credibility with bank partners |
| OI-03 | Does CrossBorderPay have a current fraud detection system (rule-based or otherwise)? If yes, what are its documented precision and recall? This determines whether the PoC is building a new capability or augmenting an existing one. | CrossBorderPay Head of Risk | P1 — Critical | Open | Week 0 | Cannot confirm whether PoC is greenfield or augmentation; rule baseline for comparison will be synthetic regardless |
| OI-04 | Has the platform's legal team reviewed whether an ML-based risk score constitutes a "credit decision" or risk decision triggering any RBI disclosure obligation? | CrossBorderPay Legal / Compliance | P1 — Critical | Open | Week 4 | Regulatory memo cannot be completed; PoC must include explicit disclaimer that legal review is pending |
| OI-05 | What is the current STR filing process? What triggers it? Who approves it? Which AD bank files it — CrossBorderPay or DBS/JP Morgan? | CrossBorderPay Compliance | P1 — Critical | Open | Week 1 | Explainability audit log cannot include correct regulatory workflow notes |
| OI-06 | Under PMLA, does CrossBorderPay (or its AD partners) have specific obligations that constrain how automated fraud decisions can be used? | CrossBorderPay Legal | P2 — High | Open | Week 2 | Regulatory memo will need strong caveats; output format may need modification |
| OI-07 | What is the single fraud typology that has cost CrossBorderPay (or its clients) the most in the last 12 months? | CrossBorderPay Head of Risk / CEO | P1 — Critical | Open | Week 1 | Fraud scenario priority in V3.1 is based on FATF defaults, not CrossBorderPay actuals; may not match primary risk |
| OI-08 | What is the platform's false positive tolerance? What is the estimated cost of blocking a legitimate transaction? | CrossBorderPay CEO / PM | P1 — Critical | Open | Week 3 | Operating threshold recommendation will be directional only; cannot calibrate to a validated business cost |
| OI-09 | Is the ultimate goal to build a new fraud scoring capability from scratch, or to augment an existing detection system? This determines whether recall or precision is the primary metric to optimise. | CrossBorderPay CTO / Head of Risk | P1 — Critical | Open | Week 0 | Cannot select correct operating threshold or frame model without knowing the objective |
| OI-10 | Which client segment is the highest fraud risk priority? Which segment would benefit most from this PoC? | CrossBorderPay CEO | P2 — High | Open | Week 1 | Fraud scenario weighting in V3.1 may not match the platform's highest-priority segment |
| OI-11 | What is the expected transaction volume per day in production? What is the latency SLA for a payment scoring decision? | CrossBorderPay CTO | P2 — High | Open | Week 4 | Production API infrastructure (Phase 3) cannot be sized correctly |
| OI-12 | Does CrossBorderPay have an existing data warehouse or data lake? Or does the scoring engine integrate directly with the payment processing system? | CrossBorderPay CTO | P2 — High | Open | Week 4 | Phase 3 API design and feature store architecture may need to change |
| OI-13 | What is the actual payment rail distribution in the platform's transaction volume? (% SWIFT, Fedwire, CHIPS, SEPA, CHAPS) | CrossBorderPay Ops | P3 — Medium | Open | Week 1 | V3.1 uses estimated distribution; model may learn wrong rail patterns |
| OI-14 | Is there a preference for a specific ML framework or infrastructure (AWS SageMaker, on-premise)? | CrossBorderPay CTO | P3 — Medium | Open | Week 1 | PoC uses local MLflow; production MLOps architecture may need to change |
| OI-15 | Who will maintain the scoring model post-production? Is there an existing data science team or will this be the founding ML capability? | CrossBorderPay CTO | P3 — Medium | Open | Week 5 | Production roadmap assumptions may need revision; model governance section may need expansion |
| OI-16 | What is the exact date range (start date and end date) of the V3.1 synthetic dataset? This is needed to define temporal split boundary dates for train/validation/test. | CrossBorderPay / DS (verify from data) | P1 — Critical | Open | Week 1 | Temporal split boundary dates cannot be confirmed; train/val/test split ratios used instead of calendar-based boundaries |

---

# Section 17: Weekly Status Report Templates

## 17.1 CEO View — Weekly Status Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CrossBorderPay FRAUD DETECTION POC — CEO WEEKLY UPDATE
Week [X] of 5  |  Report Date: [DATE]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

OVERALL STATUS:    [ GREEN / AMBER / RED ]
OVERALL PROGRESS:  [XX]% Complete

─────────────────────────────────────────────────────────────
THIS WEEK IN ONE SENTENCE
[What changed that matters to the business this week]
─────────────────────────────────────────────────────────────

KEY ACCOMPLISHMENTS (Business Language)
• [What was built/proven — phrased in business outcome terms]
• [e.g., "Model now detects smurfing patterns with 83% accuracy"]
• [e.g., "Batch pipeline scored all 50,800 transactions in under 3 minutes"]

NEXT WEEK FOCUS
• [Next milestone — what will be ready to show]

DECISIONS NEEDED FROM CrossBorderPay
• [Open Item ID] — [Question] — Needed by [Date]
• [Open Item ID] — [Question] — Needed by [Date]

RISKS TO TIMELINE
• [Risk] — [Status: Mitigated / Active] — [Mitigation action]

DEMO DATE CONFIRMATION
Planned: [DATE]  |  Confirmed: [YES / NO — action if NO]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 17.2 CTO View — Weekly Status Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CrossBorderPay FRAUD DETECTION POC — CTO TECHNICAL UPDATE
Week [X] of 5  |  Report Date: [DATE]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TECHNICAL STATUS:    [ GREEN / AMBER / RED ]
TECHNICAL PROGRESS:  [XX]% of WBS tasks complete

─────────────────────────────────────────────────────────────
COMPLETED THIS WEEK
• [Technical task] — [Status: DONE / PARTIAL]
• [e.g., "XGBoost model trained — AUC-PR: 0.78 on validation set"]
• [e.g., "Batch pipeline scored 50,800 rows in 2m 47s — SHAP coverage 100%"]

CURRENT MODEL METRICS (Updated Weekly)
┌───────────────────┬───────────────┬────────────────────┬────────┐
│ Metric            │ Rule Baseline │ XGBoost (Latest)   │ Target │
├───────────────────┼───────────────┼────────────────────┼────────┤
│ AUC-PR            │ [X]           │ [X]                │ 0.70+  │
│ Recall @ 5% FPR   │ [X]           │ [X]                │ 75%+   │
│ Precision         │ [X]           │ [X]                │ 60%+   │
│ F1 Score          │ [X]           │ [X]                │ 0.66+  │
│ Batch runtime     │ N/A           │ [X] min            │ < 5min │
└───────────────────┴───────────────┴────────────────────┴────────┘

NEXT WEEK TECHNICAL PLAN
• [Task with WBS ID]
• [Task with WBS ID]

BLOCKERS
• [Technical blocker] — [Resolution path]

OPEN ITEMS REQUIRING CTO INPUT
• [OI-ID] — [Question]

CODE REPOSITORY STATUS
Branch: [branch name]  |  Tests Passing: [X/Y]  |  Docker Build: [PASS/FAIL]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 17.3 Project Team View — Weekly Status Report

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CrossBorderPay FRAUD POC — TEAM WEEKLY STANDUP REPORT
Week [X] of 5  |  Report Date: [DATE]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

WBS TASKS COMPLETED THIS WEEK
• [WBS ID] [Task] — DONE
• [WBS ID] [Task] — DONE

WBS TASKS IN PROGRESS
• [WBS ID] [Task] — [X]% complete — owner: [name]

PLANNED FOR NEXT WEEK
• [WBS ID] [Task] — owner: [name] — due: [date]

BLOCKERS (Action Required)
• [Blocker] — blocked since [date] — needs: [who / what]

OPEN ITEMS PENDING CrossBorderPay RESPONSE
• [OI-ID] — [Question] — escalate if no response by [date]

RAID UPDATES
• [New risk or resolved risk]
• [New assumption or changed assumption]

EFFORT LOG (This Week)
┌─────────────┬──────┬──────────────────┬───────────────┬──────────────────┐
│ Team Member │ Role │ Hours This Week  │ Hours to Date │ Remaining Budget │
├─────────────┼──────┼──────────────────┼───────────────┼──────────────────┤
│ [Name]      │ DS   │ [X]h             │ [X]h          │ [X]h             │
│ [Name]      │ PM   │ [X]h             │ [X]h          │ [X]h             │
└─────────────┴──────┴──────────────────┴───────────────┴──────────────────┘
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# Section 18: Demo Plan

## 18.1 Demo Overview

| Field | Detail |
|---|---|
| Audience | CrossBorderPay CEO (Vikas), CTO, Head of Risk (if available) |
| Duration | 60 minutes (40 min demo + 20 min Q&A) |
| Format | Live batch pipeline run + slide deck |
| Backup | Pre-run scored_transactions.csv on standby; static CSV walkthrough if live run fails |
| Demo Environment | Docker Compose stack confirmed live 48h before demo; scored_transactions.csv pre-generated |

## 18.2 Demo Flow

| Segment | Duration | Owner | Content |
|---|---|---|---|
| 1. Business Problem | 5 min | PM | Why ML is needed; what the capability gap is; regulatory stakes |
| 2. What We Built | 5 min | PM | Architecture diagram; 4 components in plain language |
| 3. The Data | 3 min | DS | V3.1 dataset — 50,800 transactions, 10 fraud typologies, why synthetic is valid for PoC |
| 4. Model Performance | 7 min | DS | Model comparison table; rule baseline vs. XGBoost recall improvement |
| 5. Live Batch Pipeline Demo | 10 min | DS | Run batch_scorer.py live on demo dataset (30 transactions); show scored_transactions.csv output; highlight scores and SHAP columns |
| 6. SHAP Deep Dive | 5 min | DS | Walk through one RED transaction's top-5 SHAP features; explain what each feature means |
| 7. Adversarial Cases | 5 min | DS | Show one legitimate large export scoring GREEN; one smurfing pattern scoring RED |
| 8. Production Roadmap | 5 min | PM | 5 phases; what Phase 2 needs from CrossBorderPay |
| 9. Q&A | 15 min | All | See anticipated questions below |

## 18.3 Demo Scenarios (Batch Pipeline)

The demo dataset contains 30 pre-selected transactions from the V3.1 dataset covering these scenarios. The batch pipeline is run live; scored_transactions.csv is shown.

### Scenario 1 — Clear Fraud (Expected: RED, Score 85–95)

```
Transaction: DEMO-001
Customer:    CUST-SME-042
Amount:      USD 98,500
Corridor:    IND-UAE
Purpose:     P0101
Rail:        SWIFT
Beneficiary: BEN-NEW-003 (first seen 2 days ago)
Time:        02:14 local

Expected output row:
transaction_id: DEMO-001
risk_score:     91
risk_tier:      RED
f1_name:        days_since_beneficiary_first_seen  f1_value: 2.0   f1_contribution: 0.34
f2_name:        amount_zscore_90d                  f2_value: 6.1   f2_contribution: 0.28
f3_name:        submission_hour                    f3_value: 2.0   f3_contribution: 0.19
f4_name:        corridor_fatf_score                f4_value: 8.5   f4_contribution: 0.12
f5_name:        txn_count_24h                      f5_value: 4.0   f5_contribution: 0.09
```

### Scenario 2 — Legitimate Transaction (Expected: GREEN, Score < 30)

```
Transaction: DEMO-002
Customer:    CUST-IT-017
Amount:      USD 42,000
Corridor:    IND-USA
Purpose:     P0802
Rail:        SWIFT
Beneficiary: BEN-EST-091 (established, 18 months old)
Time:        10:30 local

Expected output row:
risk_score:  22   risk_tier: GREEN
Demonstrates: Model does NOT over-flag legitimate established-pattern transactions
```

### Scenario 3 — Dormant Account Reactivation (Expected: RED, Score 78–88)

```
Transaction: DEMO-003
Customer:    CUST-SME-091 (last transaction 127 days ago)
Amount:      USD 175,000
Corridor:    IND-SG

Expected output row:
risk_score:  82   risk_tier: RED
f1_name:     days_since_last_transaction    f1_value: 127.0   f1_contribution: 0.31
f2_name:     amount_zscore_90d             f2_value: 9.1     f2_contribution: 0.24
Demonstrates: Dormant reactivation + large amount combination detected
```

### Scenario 4 — Smurfing Sequence (Expected: Escalating AMBER → RED)

```
Three transactions for same customer, USD 24,800 each, 8 minutes apart.

Transaction 1: risk_score ~55 (AMBER) — velocity_acceleration: moderate
Transaction 2: risk_score ~68 (AMBER) — txn_count_1h rising
Transaction 3: risk_score ~83 (RED)   — is_near_threshold=1, txn_count_1h peak

Demonstrates: Batch pipeline reveals escalating pattern across a sequence of
individually-innocent-looking transactions
```

## 18.4 Questions the CEO May Ask — Suggested Responses

| Question | Suggested Response |
|---|---|
| "What happens when the model is wrong and blocks a legitimate transaction?" | "Every AMBER and RED transaction goes to an analyst queue — not automatic rejection. The SHAP features tell the analyst exactly what to verify. No transaction is auto-rejected without human review at current threshold settings." |
| "Will RBI have a problem with this?" | "We've drafted a regulatory positioning memo framing this as a risk scoring tool supporting human decision-making, not an automated rejection engine. The STR filing obligation remains with DBS and JP Morgan. Two items require your legal team to confirm before production." |
| "Can we show this to DBS or JP Morgan?" | "Yes — the PoC is designed to be demonstrable to bank partners. The SHAP audit log addresses what correspondent banks want to see: structured audit trails and documented rationale." |
| "When can this replace our current system?" | "The roadmap shows Phase 3 production rollout at 6–9 months from PoC approval — conditional on real data access in Phase 2. Any existing system runs in parallel through Phases 2 and 3 as a fallback." |
| "What does the model look at to flag a transaction?" | "For every transaction, it looks at 37 signals — things like how much this customer normally transfers, whether the beneficiary is new, whether the amount is close to a reporting threshold, the time of day, and the risk profile of the destination country. It then shows you the top 5 signals that drove the decision." |

## 18.5 Questions the CTO May Ask — Suggested Responses

| Question | Suggested Response |
|---|---|
| "How fast is the batch pipeline?" | "In testing, the full 50,800 transactions complete in under 5 minutes on a standard developer laptop. For production real-time scoring, Phase 3 introduces the FastAPI endpoint — the model inference is under 50ms per transaction." |
| "How do we retrain the model when fraud patterns change?" | "The PoC establishes the training pipeline and MLflow experiment tracking. Retraining is a Phase 3 deliverable. Score distribution shift in the monitoring check is the trigger signal." |
| "What happens if the model goes down in production?" | "The Phase 3 production design includes a circuit breaker — if the ML model is unavailable, payments are allowed through and flagged for offline review. The circuit breaker is a documented Phase 3 requirement, not in PoC scope." |
| "Is the code production-grade or just notebook code?" | "Feature engineering and scoring logic are implemented as Python modules with unit tests — not raw notebook code. The batch pipeline is structured as a proper Python package with Docker build and schema validation. Full DevSecOps hardening is a Phase 3 gate." |
| "What security review was done?" | "The sandbox pipeline has no hardcoded credentials — all paths are in .env excluded from version control. A full penetration test is a mandatory gate before Phase 3 production deployment." |

---

# Section 19: Future Roadmap

```
5-Phase Journey: Synthetic PoC → Advanced Fraud Intelligence

Phase 1 (Weeks 0–5):   Synthetic Data PoC (current)
Phase 2 (Months 3–6):  Real Data Pilot
Phase 3 (Months 6–12): Production Rollout
Phase 4 (Months 12–18): Real-Time Scoring at Scale
Phase 5 (Months 18–30): Advanced Fraud Intelligence
```

## Phase 1 — Synthetic Data PoC (Weeks 0–5)

**Goal:** Demonstrate that ML-based transaction risk scoring is technically viable for the platform's cross-border payments context using the approved V3.1 synthetic dataset.

**Phase 1 Exit Criteria:** Batch scoring pipeline demo accepted by CEO and CTO; production roadmap approved; Phase 2 scope agreed.

## Phase 2 — Real Data Pilot (Months 3–6)

**Goal:** Validate synthetic model performance on real CrossBorderPay transactions; establish ground truth for model improvement.

**Key Activities:**
- Data sharing agreement with CrossBorderPay (legal prerequisite)
- Anonymisation and PII stripping of historical transactions
- Real data ingestion and feature engineering
- Model retraining on real data
- Blind validation: score historical transactions; compare model flags with known outcomes
- Analyst feedback loop: capture human decisions as training labels
- OFAC/UN sanctions list integration (first real regulatory data feed)
- **Dataset Scaling (pending CrossBorderPay approval):** Expansion to 150,000+ transactions; stratified fraud rates by business type (0.3%–2.0% — actual rates to be confirmed by CrossBorderPay Head of Risk); standalone beneficiaries.csv with 8,000+ beneficiaries; noise injection rules. These parameters MUST NOT be implemented in Phase 1 and MUST NOT be assumed without explicit CrossBorderPay sign-off.

**Phase 2 Success Gate:** Model performance on real data within 10% of synthetic PoC metrics.

## Phase 3 — Production Rollout (Months 6–12)

**Goal:** Deploy scoring engine in parallel with any existing detection approach; begin capturing production value.

**Key Activities:**
- Production-grade FastAPI scoring endpoint on AWS ap-south-1 (Mumbai — India data residency)
- Real-time feature store (Redis for velocity features, DynamoDB for behavioural baselines)
- Integration with CrossBorderPay payment processing system (API hook pre-settlement)
- STR filing workflow integration with AD banking partners
- Full security review and penetration test
- A/B test: ML scoring vs. baseline on live traffic (shadow mode first)
- Model monitoring and drift detection
- Retraining pipeline with CrossBorderPay approval gate
- **Dashboard build:** Executive View, Operational View, Fraud Analyst View (Streamlit) — see Section 11 design

## Phase 4 — Real-Time Scoring at Scale (Months 12–18)

**Goal:** True real-time sub-100ms scoring integrated into the payment authorisation flow.

**Key Activities:**
- Graph database (AWS Neptune) for network fraud detection
- Real-time Kafka event stream for transaction ingestion
- Production feature store with < 10ms lookup latency
- Model ensemble (XGBoost + Isolation Forest + network model)
- Automated model retraining on rolling 90-day transaction window
- GIFT City product extension — separate model for GIFT City IFSC transactions
- Shared fraud intelligence with DBS/JP Morgan
- **Network feature implementation:** shared_beneficiary_graph_score, fraud_ring_signal, community_detection_score, hawala_mirror_flag, bic_network_score, lei_network_flag, network_centrality_score

## Phase 5 — Advanced Fraud Intelligence (Months 18–30)

**Goal:** Build CrossBorderPay into a fraud intelligence platform, not just a scoring engine.

**Key Activities:**
- Graph Neural Network (GNN) for network fraud detection (hawala rings, shell company clusters)
- LLM-based SWIFT message analysis (MT103/pacs.008 field anomaly detection)
- Federated learning with AD bank partners (shared model without sharing raw data)
- Fraud intelligence API (productised as a paid feature for enterprise clients)
- Real-time STR pre-filing system (model generates draft STR for analyst approval)
- FATF mutual evaluation support
- Behavioural biometrics for portal login (ATO prevention at authentication layer)

---

# Section 20: Final Recommendation

## 20.1 Recommended Execution Strategy

> **Approved to proceed to planning — Conditional.**
>
> Conditions:
> 1. P1 Open Items OI-01, OI-03, OI-07, OI-08, OI-09 answered before Week 1.
> 2. V3.1 dataset validated (label leakage test passes, time span confirmed) before any model training.
> 3. Rule-based baseline constructed and benchmarked before any ML model is trained.
> 4. **Data Governance Rules are in force for all sessions:** No dataset parameter (transaction count, account count, fraud rate, business mix, fraud typologies) may be modified, augmented, or scaled without explicit CrossBorderPay approval. Dataset expansion recommendations belong exclusively in Section 19 (Future Roadmap — Phase 2).
> 5. Explainability is constrained to top-5 SHAP features per transaction — no reason codes, no plain-language narratives in Phase 1.

## 20.2 Recommended Priorities

| Priority | Action | Why |
|---|---|---|
| 1 | Resolve P1 Open Items (OI-01, OI-03, OI-07, OI-08, OI-09) before Week 1 | These five answers determine the model design, threshold strategy, and fraud scenario priority |
| 2 | Build the rule-based baseline before any ML model | Without it, no ML result can be interpreted as improvement |
| 3 | Validate V3.1 with label leakage test before training | Contaminated data produces useless results regardless of model quality |
| 4 | Prioritise TBML and smurfing as the two primary fraud typologies | Highest business impact for Indian cross-border context; strongest ML differentiation from rules |
| 5 | Document the 254-fraud-case constraint in the model card honestly before the demo | CEO will ask about confidence; the honest answer preserves credibility and frames Phase 2 clearly |
| 6 | Run adversarial test cases before finalising threshold recommendation | Threshold must survive realistic hard cases, not just clean test set performance |
| 7 | Brief CrossBorderPay legal on regulatory memo no later than Week 4 | Surprises here can delay or derail the production roadmap conversation |

## 20.3 Critical Success Factors

| # | Factor | Why Critical |
|---|---|---|
| 1 | Brutally honest model card | Credibility with CTO and bank partners depends on transparency about the 254-case constraint and synthetic data limitations — overselling kills trust |
| 2 | Rule-based baseline must be fair | If the baseline is deliberately weak, the ML improvement is meaningless. The baseline should be the strongest reasonable rule system. |
| 3 | Business language throughout | Every technical output must have a plain-language equivalent for the CEO. AUC-PR means nothing. "Catches 83% of fraud" means everything. |
| 4 | Batch pipeline demo must run flawlessly | Technical credibility is established or destroyed in the demo. Invest heavily in rehearsal and backup output CSV. |
| 5 | SHAP features must tell a story | The demo must show that the 5 features selected actually make intuitive sense to a fraud analyst — not just a data scientist. |
| 6 | Adversarial tests must be documented honestly | If the model fails a test case, document it and show the plan to fix it. Hiding failures destroys credibility when they surface in production. |
| 7 | Regulatory positioning must be grounded | Do not overstate RBI compliance. The memo must be factual and caveated where legal review is pending. |

## 20.4 Top 10 Decisions CrossBorderPay Must Make Before Production Implementation

| # | Decision | Who Decides | Stakes |
|---|---|---|---|
| 1 | **False positive tolerance:** What % of legitimate transactions are we willing to hold for review to catch more fraud? | CEO + Head of Risk | Determines operating threshold; affects every downstream metric |
| 2 | **Replace or augment:** Is the ML model replacing any existing detection approach, or running as a parallel layer? | CTO + Head of Risk | Determines API integration architecture and transition risk |
| 3 | **Real-time vs. batch:** Does every transaction need a score before settlement, or is post-settlement review acceptable? | CTO + CEO | Determines Phase 3 infrastructure cost by an order of magnitude |
| 4 | **Analyst workflow:** Who reviews AMBER transactions? What is the SLA? What if no analyst is available? | Head of Risk + Ops | Without this answer, the queue fills and the model's value is unrealised |
| 5 | **STR filing ownership:** When the model flags a transaction, who files the STR — CrossBorderPay or the AD bank? | Legal + Compliance | Determines where regulatory obligation lands and what workflow integration is needed |
| 6 | **Data residency:** All transaction data used for ML training and inference must reside in AWS ap-south-1 (Mumbai). Hard constraint in all production architecture decisions. | CTO | RBI data residency requirement; non-negotiable |
| 7 | **Model retraining governance:** Who approves a new model version before it goes to production? | CTO + Head of Risk | Without a governance gate, model updates can introduce bias or degrade performance without accountability |
| 8 | **Client communication:** When a legitimate client's transaction is held for review, what do we tell them? | CEO + PM | Client relationship risk; must be defined before live deployment |
| 9 | **DBS/JP Morgan integration:** Will the scoring output be shared with partner banks for their own AML processes? | CEO + CTO | If yes, the API becomes a commercial product — legal, commercial, and technical complexity increases substantially |
| 10 | **ML ownership post-production:** Who owns the model in production? Is there a dedicated data scientist? | CEO + CTO | Without ownership, the model degrades as fraud evolves and nobody has the mandate to retrain it |

## 20.5 Final Verdict

> This PoC, executed as documented here, will produce an artifact that meets a high bar: technically credible, commercially articulate, regulatorily aware, and production-informed. It demonstrates that the person who built it understands not just how to train a model but why fraud works the way it does, what the regulatory obligations are, and what it takes to get from a Jupyter notebook to a system a bank would trust.
>
> **Three things will determine whether the PoC converts to a paid engagement:**
>
> 1. The batch pipeline runs end-to-end without errors in the demo — no excuses
> 2. The SHAP features for a RED transaction make the Head of Risk lean forward and say "yes, that's exactly what I'd want to know"
> 3. The model card is honest about the 254-case training constraint and frames Phase 2 dataset expansion as the clear logical next step
>
> Everything else in this document is infrastructure for those three moments.

---

*End of Document — Cross-Border Transaction Risk Scoring Engine PoC Plan V3.0*
*Prepared by Joint Delivery Team | June 2026 | For CEO and CTO Review Only*
* | cross-border payments company | Bengaluru*
