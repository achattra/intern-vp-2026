# Project Monitoring Database: Data Normalization & Exploratory Analysis

This project takes a single flat project monitoring master table (80+ columns) and decomposes it into a normalized relational schema, then performs exploratory data analysis (EDA) and predictive modeling on the resulting tables to derive insights around resource allocation, scheduling, legal processes, and finance.

## Project Overview

This project is based on a real consultant project monitoring dataset from an IT Governance, Risk, & Compliance (GRC) and cybersecurity consulting engagement. The original flat table serves as the master source of project metadata. Since several transactional tables (contribution records, time logs, legal activity logs, and plan revision history) weren't available in the raw source, synthetic data was generated to simulate these tables under realistic business rules, while preserving the schema and relational structure of the original data.

### Objectives

- Decompose the master table into a normalized relational schema (multiple entity tables linked by keys)
- Generate synthetic transactional data following realistic business rules and distributions
- Enforce referential integrity across tables (Job Code / Term as composite keys)
- Perform exploratory data analysis and derive KPIs from the normalized schema
- Engineer project-term-level features and build a classification model to predict schedule delay risk

## Repository Structure

```
├── data/
│   └── processed/      # Normalized, exported CSV/XLSX tables
├── docs/
│   ├── Normalizing.html
│   ├── Analysis.html
│   └── Prediction.html
├── Normalizing.ipynb   # Data pipeline: master table → 7 normalized tables
├── Analysis.ipynb      # EDA notebook on the normalized tables
├── Prediction.ipynb    # Feature engineering + classification modeling for schedule delay
└── README.md
```

## Data Pipeline: Normalized Schema

`Normalizing.ipynb` cleans the raw master table and decomposes it into the following 7 entity tables:

| Table | Description |
|---|---|
| **Contribution Partner** | Employee role and % contribution per project (weighted allocation) |
| **Time Contribution** | Logged time spent per employee, per project |
| **Legal Activity** | Legal document activity events with timestamps, keyed to project |
| **PMO Activity** | Project management activity events across the project lifecycle |
| **Master Baseline** | Fixed baseline completion date per Job Code / Term |
| **Plan History** | Time series of schedule revisions per Job Code / Term |
| **Finance Activity** | Financial event log per Job Code / Term — invoice, billing, payment stages |

Tables are linked via shared keys (Job Code, Term, employee ID) to preserve referential integrity across the schema.

## Exploratory Data Analysis

`Analysis.ipynb` runs univariate and bivariate EDA across each of the 7 tables — distributions, groupby aggregations, top-N rankings, and time series trends — producing 4–5 visualizations with interpretations per table. Coverage includes:

- Workload and contribution distribution across employees
- Resource allocation via contribution scores and time-based metrics
- Event frequency and monthly trend analysis for legal and PMO activity logs
- Schedule variance analysis (baseline vs. revision history)
- Finance funnel analysis — time-between-stages and completion rate

Each table section closes with a **Key Findings** summary distilling the EDA into actionable takeaways.

## Predictive Modeling: Schedule Delay Classification

`Prediction.ipynb` engineers a project-term-level feature table (joining Master Baseline, Plan History, and aggregated Contribution/Time/Legal/PMO/Finance activity) to predict whether a project term will miss its baseline schedule.

**Target:** `delayed` — binary classification (1 = final revised plan date fell after the original baseline date)

**Approach:**
- Feature engineering across all 7 normalized tables into a single project-term-level dataset
- Correlation analysis to identify signal-carrying features before modeling
- Baseline logistic regression (with class-weight balancing for the ~77/23 class split) vs. XGBoost, compared on ROC-AUC, precision/recall, and confusion matrix
- Single-feature ablation test to isolate the contribution of the strongest predictor

**Key Findings:**
- `num_revisions` (number of plan revisions recorded) is the dominant predictor of delay, with a correlation of 0.35 against delay days — by far the strongest of all engineered features
- A logistic regression using *only* `num_revisions` (AUC 0.635) performed on par with, or slightly better than, the full 6-feature logistic regression (AUC 0.630) and the full-feature XGBoost model (AUC 0.608) — indicating the other engineered features (team structure, legal/PMO/finance activity counts) add negligible predictive value, and there's no hidden non-linear signal for a higher-capacity model to exploit
- **Limitation:** `num_revisions` is partly a symptom of delay rather than a clean early predictor, since already-delayed projects tend to accumulate more revisions. This model identifies *correlates* of delay rather than providing genuine early-warning capability — a natural next step would restrict features to only what's known within the first revision or first 30 days of a term
- **Synthetic data caveat:** since delay values were generated as mock data, these findings reflect structure built into the data-generation process; the pipeline itself (feature engineering, model comparison, ablation testing) demonstrates the methodology that would be applied to real project data

## Tech Stack

- **Python** — pandas (data wrangling, groupby aggregations), matplotlib/seaborn/altair (visualization), scikit-learn (logistic regression, train/test split, evaluation metrics), XGBoost (gradient boosted classification)
- **Jupyter Notebook** — pipeline, analysis, and modeling workflow