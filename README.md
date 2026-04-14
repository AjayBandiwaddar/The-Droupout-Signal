# Dropout Signal

Early warning system that detects student dropout risk weeks before 
it appears in grades — built on Databricks.

## What it does
- Assigns every student a continuous risk score from 0 to 1
- Detects deteriorating trends in grades, attendance, and engagement
- Explains the top 3 factors driving each student's score
- Audits predictions for fairness across gender and socioeconomic groups
- Outputs an intervention table: who needs help, why, and what to do

## What makes this different
Most systems flag dropout after grades have already fallen.
This system detects the trajectory — weeks before it shows up.

## Built with
- Databricks (everything runs here)
- Delta Lake
- XGBoost
- SHAP
- MLflow

## Running this project

### Step 1 — Import into Databricks
Go to Databricks → Repos → Add Repo
Paste this GitHub URL and clone it.

### Step 2 — Set up your cluster
- Databricks Runtime: 13.3 LTS ML
- Python 3.10+

### Step 3 — Install dependencies
Run in any notebook cell:
%pip install -r requirements.txt

### Step 4 — Run notebooks in order
01_data_ingestion → 02_cleaning_and_features → 
03_model_training → 04_fairness_audit → 
05_explainability_and_output

