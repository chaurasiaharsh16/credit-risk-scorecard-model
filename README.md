# Credit Risk Scorecard Model

## Overview
End-to-end credit risk modeling pipeline using LendingClub data.

Includes:
- PD model (Logistic Regression with WOE)
- Scorecard development (PDO scaling)
- LGD and EAD estimation
- Expected Loss calculation

---

## Pipeline

### Stage 1: Data Preparation
- Target definition
- Train/Test/OOT split

### Stage 2: Feature Engineering
- Fine & coarse classing
- WOE transformation
- IV filtering

### Stage 3: Modeling
- Logistic regression (PD)
- Score scaling
- Scorecard construction

### Stage 4: Evaluation
- AUC, Gini, KS

### Stage 5: Portfolio
- Expected Loss = PD × LGD × EAD

---

## Results
- AUC: XX
- Gini: XX
- KS: XX

---

## Tech Stack
- Python (pandas, numpy, sklearn, statsmodels)