-
Perfect — then you want one strong “master prompt” that forces Copilot to build the entire pipeline correctly, end-to-end, without drifting into leakage, bad labels, or over-engineering.
Below is a single, cohesive, copy-paste prompt you can give to GitHub Copilot Chat. It’s long on purpose — this is how you control the outcome.
✅ GitHub Copilot Master Prompt
(End-to-End Propensity Modeling Pipeline)
You are a senior machine learning engineer building a production-grade, time-causal customer propensity modeling system for transaction behavior prediction.
🎯 Objective
Build an end-to-end ML pipeline that:
Ingests transaction-level data
Constructs time-causal behavioral features
Defines future outcome labels
Performs rolling train/validation/test splits
Trains and evaluates binary propensity models
The final system must be defensible, leakage-free, explainable, and production-ready.
📊 Data Assumptions
Input table (PySpark DataFrame):
Copy code

customer_id (string)
txn_date (date)
txn_amount (double)
merchant_id (string)
mcc (string)
channel (string)
One row per transaction.
All timestamps are accurate and timezone-normalized.
⏱ Time Design (STRICT – DO NOT VIOLATE)
Observation window: Q3 + Q4 2024
Label window: Q1 2025
Hard rules:
NO feature may use data from Q1 2025 or later
Labels must be derived only from Q1 2025
Percentiles or population statistics must be computed within the window they belong to
No random train/test splits — time-based splits only
🏷 Label Definitions (Binary Outcomes)
Create three independent binary labels, computed strictly from Q1 2025:
1. Dormancy Propensity
Copy code

dormant = 1 if txn_count_Q1 == 0
dormant = 0 otherwise
2. Spend Decline (At-Risk) Propensity
Copy code

at_risk = 1 if (total_spend_Q1 / total_spend_Q4) <= 0.6
at_risk = 0 otherwise
3. High Spend Propensity
Copy code

high_spender = 1 if total_spend_Q1 >= p90(total_spend_Q1)
high_spender = 0 otherwise
Notes:
Labels must be mutually non-exclusive
Percentiles for labels must be computed only within Q1
🧠 Feature Engineering (From Q3–Q4 2024 Only)
Create customer-level numeric features in the following categories:
A. Recency & Engagement
days_since_last_txn
txn_count_L30, txn_count_L90
active_days_L30, active_days_L90
recency_weighted_txn_count
B. Spend & Frequency Trends
spend_L30 / spend_L90
spend_L90 / spend_L180
txn_L30 / txn_L90
linear spend slope (last 120 days)
momentum ratio (avg last 30 / avg prev 60)
C. Volatility & Stability
daily spend std (L90)
coefficient of variation (L90)
max transaction gap (L90)
txn inter-arrival std (L90)
D. Behavioral Breadth
unique merchants (L90)
unique MCCs (L90)
merchant entropy
repeat merchant ratio
top merchant spend share
E. Long-Term Anchors
tenure_days
lifetime_spend (up to end of Q4 2024)
Hard constraints:
No hard thresholds
No label-proxy features
All features must be numeric
🧪 Train / Validation / Test Strategy
Use time-based splits only:
Train: (Q1–Q2 → Q3)
Validation: (Q2–Q3 → Q4)
Test: (Q3–Q4 → Q1)
Ensure:
No customer leakage across time windows
Identical feature definitions across splits
🤖 Model Training
For each label, train:
Baseline Logistic Regression
Primary Model: Gradient Boosted Trees (XGBoost or LightGBM)
Requirements:
Binary objective
Regularized, shallow trees
Probability outputs (not class labels)
Feature importance via SHAP
📈 Evaluation
For each model:
AUC
KS statistic
Lift @ top deciles
Calibration curve
Do NOT use accuracy.
🧱 Engineering Requirements
Use PySpark for data prep and feature engineering
Use MLflow-compatible structure
Modular functions: ingestion, features, labeling, splitting, training, evaluation
Clear inline comments explaining time-causality safeguards
✅ Final Output
Produce:
End-to-end PySpark + Python ML pipeline code
Clean feature table
Trained models per outcome
Evaluation metrics printed clearly
This code should be suitable for production review and model risk assessment.
Do not simplify the problem. Do not add shortcuts. Correctness > cleverness.
Why this prompt will actually work
It forces Copilot to respect time
It prevents label leakage and rule re-encoding
It naturally leads to multiple binary models
It produces something you can defend in a review
If you want next, I can:
Tailor this prompt for Spark + SageMaker
Make a lighter version for fast prototyping
Or split this into Copilot-friendly phases
But this prompt, as-is, is the right foundation.








Label Definitions
Class 0 — Dormant
A customer is labeled Dormant in Q1 if any of the following conditions hold:
txn_count_Q1 == 0
OR total_spend_Q1 <= P10(total_spend_Q1)
OR active_days_Q1 <= 1
Interpretation: customer is effectively inactive during Q1.
Class 2 — Heavy Spender
A customer is labeled Heavy Spender in Q1 if all of the following conditions hold:
total_spend_Q1 >= P85(total_spend_Q1)
AND txn_count_Q1 >= P75(txn_count_Q1)
AND active_days_Q1 >= P75(active_days_Q1)
AND days_since_last_txn_Q1 <= P50(days_since_last_txn_Q1)
Interpretation: customer demonstrates consistently high value and engagement within Q1.
Class 1 — At Risk
A customer is labeled At Risk if all of the following conditions hold:
Customer is not Dormant
Customer is not Heavy Spender
AND any of the following conditions hold:
total_spend_Q1 <= P40(total_spend_Q1)
OR txn_count_Q1 <= P40(txn_count_Q1)
OR active_days_Q1 <= P40(active_days_Q1)
OR days_since_last_txn_Q1 >= P60(days_since_last_txn_Q1)
Interpretation: customer remains active in Q1 but exhibits low or declining engagement intensity.
Final Label Assignment Priority
Labels are assigned in the following order:
Dormant
Heavy Spender
At Risk (default for remaining customers)
This ensures mutually exclusive and collectively exhaustive classes.
Numeric Encoding
Segment
Label
Dormant
0
At Risk
1
Heavy Spender
2
