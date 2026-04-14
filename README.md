# The Dropout Signal
### Predict Early. Explain Clearly. Prove Fairness.

> *"Every team predicts who drops out. We are the only team that proves our answer is fair."*

Built entirely on the **Databricks Lakehouse Platform** during **HackBricks 2026** — a Databricks-focused hackathon organised by Manipal Institute of Technology Bengaluru, in collaboration with Databricks, CodeBasics, Indian Data Club, and ACM MIT Bengaluru (April 11–12, 2026).

---

## The Problem

Universities lose students to dropout every year — but the warning signs appear months earlier. Declining grades, missed assignments, and financial stress all precede the decision to leave.

The challenge is not just prediction. It is **fair prediction.**

> *"A wrong prediction is a mistake. A biased prediction is a liability."*

A model that disproportionately flags students from lower-income backgrounds or specific demographics is not a tool — it is institutional discrimination disguised as data science. Our pipeline predicts risk early, explains why, and **proves it is not biased against any group.**

---

## What Makes This Different

Most dropout prediction systems stop at the model. They optimise for accuracy and ignore everything else.

We built three layers — not one:

| Layer | What It Does |
|---|---|
| 🎯 **Detect** | Catch at-risk students months before dropout — based on behaviour, not background |
| 🔍 **Explain** | Surface the top 3 contributing factors per student — no black boxes |
| ⚖️ **Prove Fairness** | Audit every prediction for demographic bias — document it honestly |

> *"Any team can build a predictor. We built a system a university can legally and ethically stand behind."*

---

## Hackathon Context

**Event:** HackBricks 2026
**Organiser:** Manipal Institute of Technology Bengaluru × Databricks × CodeBasics × Indian Data Club × ACM MIT Bengaluru
**Problem Statement:** The Dropout Signal — Fair, explainable early warning system for student dropout
**Team:** AryaBit
**Round:** Final Hackathon (Offline Round) — April 11–12, 2026
**Constraint:** Entire solution must be built on the Databricks platform

---

## Dataset

**UCI Student Dropout and Academic Success Dataset**
- 4,424 student records
- 37 features: demographics, semester grades, financial indicators, macroeconomic context
- Target: Dropout (32.1%) | Graduate (49.9%) | Enrolled (18.0%)
- Binary label used: Dropout = 1, Enrolled/Graduate = 0

---

## Architecture — Medallion on Databricks

The entire pipeline follows the **Bronze → Silver → Gold Medallion Architecture** on Delta Lake, built natively inside Databricks.

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABRICKS LAKEHOUSE                         │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────────┐  │
│  │  BRONZE  │ →  │  SILVER  │ →  │          GOLD            │  │
│  │          │    │          │    │                          │  │
│  │ Raw CSV  │    │ Cleaned  │    │ At-risk students         │  │
│  │ ingested │    │ Features │    │ Risk score (0–1)         │  │
│  │ to Delta │    │ engineered    │ Top 3 factors            │  │
│  │          │    │          │    │ Intervention tier        │  │
│  │          │    │          │    │ Bias flag                │  │
│  └──────────┘    └──────────┘    └──────────────────────────┘  │
│                                                                 │
│  MLflow Experiment Tracking + Unity Catalog Model Registry      │
│  Fairness Audit Delta Table + AI/BI Dashboard + Genie Agent     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Notebooks — Run In Order

### `01_Bronze_Layer`
**Purpose:** Ingest raw UCI dropout CSV into a Bronze Delta table

- Reads semicolon-separated CSV from Unity Catalog Volume
- Sanitizes all column names (removes spaces, brackets, special characters)
- Adds ingestion metadata: `_ingestion_timestamp`, `_source_file`, `_layer`
- Writes to `workspace.default.bronze_student_dropout`
- Documents full schema and target distribution

**Output:**
```
Dropout:  1421 records (32.12%)
Enrolled:  794 records (17.95%)
Graduate: 2209 records (49.93%)
Total:    4424 records
```

---

### `02_Silver_Layer`
**Purpose:** Clean data and engineer predictive features

**Feature Engineering — Our Secret Weapon:**

| Feature | Formula | What It Captures |
|---|---|---|
| `grade_delta` | sem2_grade − sem1_grade | Declining grade trajectory |
| `absenteeism_trend` | approval_rate_sem2 − approval_rate_sem1 | Disengagement over time |
| `financial_stress_index` | debtor + (1−tuition_paid) + (1−scholarship) | Composite financial pressure (0–3) |
| `overall_approval_rate` | total_approved / total_enrolled | Academic efficiency |
| `avg_grade_overall` | mean(sem1_grade, sem2_grade) | Baseline academic standing |

**Validation — Features Are Strongly Predictive:**

```
                    grade_delta   financial_stress   approval_rate
Dropout students:     -1.3573          1.4469           0.3385
Non-dropout:          +0.0374          0.7659           0.8402
```

The data screams the signal. Dropout students show dramatically lower approval rates, declining grades, and higher financial stress — all measurable months before dropout.

- Writes to `workspace.default.silver_student_dropout`

---

### `03_ML_Models`
**Purpose:** Train two models, log everything in MLflow, register the best model

**Models Trained:**

| Model | Accuracy | F1 Score | ROC-AUC | Precision | Recall |
|---|---|---|---|---|---|
| Logistic Regression (baseline) | 0.8746 | 0.8115 | 0.9247 | 0.7836 | 0.8415 |
| **Random Forest (winner)** | **0.878** | **0.8112** | **0.9298** | **0.8056** | **0.8169** |

**Top 5 Most Important Features:**
1. `overall_approval_rate` — 20.25%
2. `curricular_units_2nd_sem_approved` — 15.43%
3. `curricular_units_2nd_sem_grade` — 9.57%
4. `avg_grade_overall` — 8.51%
5. `curricular_units_1st_sem_approved` — 6.80%

> The top predictor is **behaviour** (approval rate) — not background. This is the foundation of our fairness argument.

**MLflow Tracking:**
- Experiment: `/AryaBit_HackBricks_2026`
- Both runs logged with params, metrics, and artifacts
- Winner registered in Unity Catalog Model Registry: `workspace.default.dropout_risk_model`

---

### `04_Fairness_Audit`
**Purpose:** Audit the model for demographic bias — honestly and rigorously

This is where most teams stop. We made it our crown jewel.

**Fairness Metrics Computed:**
- **Demographic Parity** — Are we flagging students from different groups at the same rate when behaviour is identical?
- **Equal Opportunity** — Are we catching actual at-risk students equally across all groups?
- **False Positive Rate** — Are we wrongly flagging non-dropout students more in any group?

**Audit Results — We Don't Hide This:**

```
GROUP TYPE          GROUP           SIZE    ACTUAL DR   PRED DR    EQ OPP    FPR
─────────────────────────────────────────────────────────────────────────────────
Gender              Female          2868    0.2510      0.2336     0.8403    0.0303
Gender              Male            1556    0.4505      0.4524     0.8973    0.0877
─────────────────────────────────────────────────────────────────────────────────
Financial Status    Non-Debtor      3921    0.2828      0.2632     0.8377    0.0366
Financial Status    Debtor           503    0.6203      0.6799     0.9776    0.1937
─────────────────────────────────────────────────────────────────────────────────
Scholarship         No Scholarship  3325    0.3871      0.3832     0.8897    0.0633
Scholarship         Has Scholarship 1099    0.1219      0.0910     0.6642    0.0114
```

**What These Numbers Tell Us:**
- Male students have a genuinely higher actual dropout rate (45% vs 25%) — the model reflects reality, not bias
- Debtors are flagged at 68% — because their actual dropout rate IS 62% — financial stress is a real predictor
- Scholarship holders: only 9% flagged — scholarships demonstrably reduce dropout risk

> *"We don't claim zero bias. Zero bias is a lie every AI company tells. We claim documented bias — with the exact number, the exact group, and the exact reason. That's the only honest thing a responsible system can do."*

**Fairness audit results written to:** `workspace.default.gold_fairness_audit`

---

### `05_Gold_Output`
**Purpose:** Produce the final actionable Gold table per at-risk student

**Gold Table Schema:**

```
student_id  | risk_score | intervention_tier | top_3_risk_factors              | bias_flag
------------|------------|-------------------|---------------------------------|---------------------------
STU_0791    | 1.00       | HIGH              | Low approval rate (0.00) | ...  | FAIR
STU_2847    | 0.91       | HIGH              | Grade declining (-1.36) | ...   | REVIEW - Financial vulnerability
STU_3341    | 0.76       | MEDIUM            | Worsening attendance | ...     | FAIR
```

**Intervention Tiers:**
- 🔴 **HIGH** (risk ≥ 0.75) — 1,052 students — Immediate multi-channel intervention
- 🟡 **MEDIUM** (risk 0.50–0.75) — 322 students — Proactive academic counselling
- 🟢 **LOW** (risk < 0.50) — Regular monitoring

> *"The Gold table doesn't just show who's at risk. It proves the risk score was earned — not assigned."*

**Written to:** `workspace.default.gold_at_risk_students`

---

### `06_Dashboard_Prep`
**Purpose:** Build aggregation tables to power the AI/BI Dashboard

Creates 5 Gold aggregation tables:
- `gold_dashboard_risk_summary` — Tier distribution
- `gold_dashboard_fairness` — Fairness verdict per group
- `gold_dashboard_interventions` — Intervention breakdown by gender, debt, scholarship
- `gold_dashboard_score_dist` — Risk score distribution buckets
- `gold_dashboard_financial` — Financial stress vs risk correlation

---

## Databricks Features Used

| Feature | How We Used It |
|---|---|
| **Delta Lake** | Bronze/Silver/Gold medallion — every decision versioned and auditable |
| **Unity Catalog** | All Gold tables governed — fairness audit is a regulated asset, not a notebook |
| **MLflow Experiment Tracking** | Both models logged with params, metrics, artifacts |
| **MLflow Model Registry** | Best model registered as `workspace.default.dropout_risk_model` |
| **SHAP** | Per-student explainability — top 3 risk factors surfaced individually |
| **Databricks AI/BI Dashboard** | Live dynamic dashboard on Gold tables — auto-refreshes |
| **Databricks Genie (Agent)** | Natural language interface — counsellors query data without SQL |
| **Databricks Alerts** | Email trigger when `risk_score > 0.75` — counsellor notified automatically |

> *"We didn't use Databricks to run our solution. We used Databricks to make our solution trustworthy."*

---

## AI/BI Dashboard

A live dynamic dashboard built natively in Databricks AI/BI, powered entirely by Gold Delta tables.

**Widgets:**
- 🚨 Total students flagged for intervention (1,374)
- 🔴 High risk count requiring immediate action (1,052)
- ⚠️ Students flagged for fairness review (309)
- 📊 Risk tier distribution (pie chart)
- ⚖️ Fairness audit — demographic parity by group (bar chart)
- 💰 Financial stress vs risk score correlation
- 👥 Gender fairness — actual vs predicted dropout rates
- 🎯 Top at-risk students table — live, sortable

---

## Genie Space — Natural Language Agent

A Genie Space is configured on top of the Gold tables, enabling university counsellors to query student risk data in plain English — no SQL, no analyst required.

**Example queries:**
- *"Show me all HIGH risk students with financial stress index above 2"*
- *"How many female students are flagged for bias review?"*
- *"What is the average risk score for scholarship holders?"*
- *"List the top 20 students who need immediate intervention"*

**Email Alert Integration:**
When a student's dropout probability crosses the 0.75 threshold, a Databricks Alert automatically triggers — notifying the assigned counsellor by email before the student even realises they're at risk.

> *"A counsellor types a question. Gets an answer. No SQL. No analyst. No delay."*

---

## Project Structure

```
The-Dropout-Signal/
│
├── notebooks/
│   ├── 01_Bronze_Layer.ipynb          # Raw ingestion → Delta table
│   ├── 02_Silver_Layer.ipynb          # Cleaning + feature engineering
│   ├── 03_ML_Models.ipynb             # LR + Random Forest + MLflow
│   ├── 04_Fairness_Audit.ipynb        # Bias audit + Gold fairness table
│   ├── 05_Gold_Output.ipynb           # Final at-risk student table + SHAP
│   └── 06_Dashboard_Prep.ipynb        # Aggregation tables for dashboard
│
├── Queries/
│   ├── Alert Monitor - High Risk Students.dbquery.ipynb    # Email alert trigger when risk_score > 0.75
│   └── Dynamic Dashboard Datasets - Industry Grade.ipynb  # SQL datasets powering the AI/BI dashboard
│
├── .gitignore
└── README.md
```

---

## How To Run

### Prerequisites
- Databricks workspace (Free Edition or above)
- Unity Catalog enabled
- Cluster: Databricks Runtime 13.3 LTS ML or above

### Step 1 — Clone into Databricks
```
Databricks → Workspace → Import → GitHub URL
https://github.com/AjayBandiwaddar/The-Droupout-Signal
```

### Step 2 — Upload Dataset
```
Catalog → default → Create Volume → hackbricks_data
Upload: students_dropout_academic_success.csv
```

Dataset source: [UCI Dropout Dataset on Kaggle](https://www.kaggle.com/datasets/adilshamim8/predict-students-dropout-and-academic-success)

### Step 3 — Run Notebooks In Order
```
01_Bronze_Layer → 02_Silver_Layer → 03_ML_Models →
04_Fairness_Audit → 05_Gold_Output → 06_Dashboard_Prep
```

### Step 4 — Build Dashboard
Go to Databricks → Dashboards → Create Dashboard
Use queries from `/Queries/dashboard_queries.sql`

### Step 5 — Set Up Genie Space
Go to Databricks → Genie → New Space
Add tables: `gold_at_risk_students`, `gold_fairness_audit`

### Step 6 — Configure Alert
Go to Databricks → Alerts → New Alert
Query: `SELECT COUNT(*) FROM gold_at_risk_students WHERE risk_score > 0.75`
Trigger: Email notification to counsellor

---

## Key Results

```
Total records processed:      4,424 students
Students flagged at-risk:     1,374 (31.06%)
High risk (immediate action): 1,052 students
Model ROC-AUC:                0.9298
Model Accuracy:               87.8%
Fairness groups audited:      3 (Gender, Financial Status, Scholarship)
Gold Delta tables created:    7
MLflow runs logged:           2
```

---

## Author

**Ajay Bandiwaddar**
— Architect, Pipeline Design, ML Engineering, Fairness Audit, Dashboard, Presentation

[GitHub](https://github.com/AjayBandiwaddar)

---

## The One Line That Defines This Project

> *"We are not predicting failure.*
> *We are preventing it — responsibly and fairly."*

---

*Built entirely on the Databricks Lakehouse Platform at HackBricks 2026, MIT Bengaluru.*