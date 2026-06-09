# Basel II PD Scorecard — End-to-End Credit Risk Model on LendingClub Data

![Python](https://img.shields.io/badge/Python-3.9+-blue?logo=python)
![Domain](https://img.shields.io/badge/Domain-Credit%20Risk-darkgreen)
![Framework](https://img.shields.io/badge/Framework-Basel%20II%20Aligned-orange)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

---

## Overview

This repository contains an end-to-end **retail credit risk modeling pipeline** built on publicly available LendingClub consumer loan data. The project delivers a **Basel II–aligned Probability of Default (PD) scorecard**, supplemented by Loss Given Default (LGD) and Exposure at Default (EAD) estimates, culminating in an Expected Loss (EL) calculation at the account level.

The pipeline is structured as a sequence of **five** (check again) modular Jupyter notebooks, each corresponding to a distinct phase of the model development lifecycle as practiced in regulated financial institutions.

---

## Business Objective

In retail credit risk, the central modeling objective is to estimate:

> **PD = P(Default within a defined horizon | Borrower characteristics at origination)**

A deployable credit model must satisfy two conditions that are often in tension: **predictive accuracy** and **causal integrity** — that is, it must learn only from information available at the point of underwriting, not from realized loan outcomes. This project enforces that discipline throughout every stage of development.

---

## Project Structure

```
lending-club-pd-scorecard/
│
├── notebooks/
│   ├── 01_data_preparation.ipynb       # Target definition, leakage removal, time splits
│   ├── 02_feature_engineering.ipynb    # WOE/IV binning, monotonicity, multicollinearity
│   ├── 03_modeling.ipynb               # Logistic regression PD, score scaling, LGD/EAD/EL
│   └── 04_evaluation.ipynb             # AUC/Gini, KS, decile analysis, OOT stability
│
├── data/
│   └── README.md                       # Instructions to download LendingClub dataset
│
├── reports/
│   └── figures/                        # Saved plots (ROC, Gini, calibration, PSI)
│
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Pipeline Summary

### Stage 1 — Data Preparation

The raw LendingClub dataset contains post-origination variables that would directly leak target information into the model (e.g., `recoveries`, `total_pymnt`, `out_prncp`). Stage 1 systematically removes these, defines the binary target (Good = 1 / Bad = 0) aligned with scorecard convention, and constructs a temporally ordered three-way split.

Key decisions:
- **Target**: `Charged Off` and `Default` statuses classified as Bad (0); `Fully Paid` as Good (1)
- **Leakage removal**: All variables reflecting post-disbursement cashflows, repayment behavior, or settlement outcomes are excluded
- **Time-based split**: Train (pre-2014) / Test (2014) / Out-of-Time (2015+) — random splits are not used because credit risk exhibits temporal autocorrelation
- **Missing value philosophy**: Economically driven imputation — `999` for time-since variables (signals no observed event), `-1` for thin-file structural absences, rather than generic median fill which would destroy WOE signal
- **Winsorization**: Applied at the 1st/99th percentile on skewed numeric variables to stabilize logistic regression coefficients

### Stage 2 — Feature Engineering (WOE / IV)

Weight of Evidence (WOE) transformation converts raw features into a monotonic, log-odds–scaled representation, which is both interpretable and directly compatible with logistic regression. Information Value (IV) is used as the primary feature selection criterion.

Key decisions:
- **Fine classing → Coarse classing**: Initial granular bins are manually reviewed and merged to enforce monotonicity and ensure each bin contains ≥5% of the population (regulatory minimum for stability)
- **IV thresholds**: Variables with IV < 0.02 are excluded (weak); IV > 0.50 are flagged for potential leakage
- **Multicollinearity**: Variance Inflation Factor (VIF) analysis is applied post-WOE to identify and remove redundant predictors
- **Reject inference**: Acknowledged as a limitation — the dataset covers only accepted applicants; the model therefore conditions on lender approval, which may introduce selection bias in deployment

### Stage 3 — Modeling (PD Scorecard + LGD/EAD/EL)

The PD model is a **logistic regression estimated on WOE-transformed features**. This architecture is preferred in regulated retail banking because it produces interpretable coefficients, is auditable by risk governance teams, and aligns with Basel II internal ratings-based (IRB) requirements.

Key decisions:
- **Variable selection**: Stepwise elimination based on p-values (threshold: 0.05); coefficients must align with the directional expectation from WOE analysis
- **Score scaling**: Logit output is transformed into a scorecard using the standard PDO (Points to Double the Odds) methodology — scores are calibrated such that a 20-point increase halves the odds of default
- **LGD model**: OLS regression on the defaulted sub-portfolio; LGD is bounded to [0, 1] via post-prediction clipping; beta regression is noted as a theoretically superior alternative
- **EAD**: Estimated as outstanding principal at time of default, with Credit Conversion Factor (CCF) applied for revolving exposures
- **Expected Loss**: Computed as EL = PD × LGD × EAD at account level, enabling portfolio-level loss forecasting

### Stage 4 — Evaluation & Monitoring

Model performance is assessed across all three temporal splits (Train / Test / OOT) to verify generalization and detect temporal decay.

Key metrics:
- **Gini coefficient** and **AUC** (primary discriminatory power metrics)
- **KS statistic** (separation between Good and Bad score distributions)
- **Decile analysis**: Population ranked into 10 score bands; default rate must decrease monotonically from decile 1 to 10
- **Calibration**: Predicted PD is compared against observed default rates by score band to assess reliability of probability estimates
- **Stability monitoring**: Population Stability Index (PSI) and Characteristic Stability Index (CSI) are used to detect distribution drift between train and OOT — PSI > 0.25 triggers model redevelopment review

---

## Key Technical Decisions

| Design Choice | Rationale |
|---|---|
| Time-based split (not random) | Credit data has temporal structure; random splits overstate generalization |
| WOE over raw features | Handles non-linearity, missing values, and produces monotonic log-odds input |
| Economic missing value logic | Preserves predictive signal for WOE bins rather than masking it with medians |
| Winsorization over deletion | Retains sample size while reducing leverage of extreme observations on coefficients |
| PDO score scaling | Produces an interpretable scorecard aligned with industry cut-off convention |
| OOT validation | Simulates production conditions; in-sample or random test results are insufficient for regulatory credibility |

---

## Limitations

This project is built as a portfolio demonstration and intentionally documents its own limitations in the spirit of rigorous model documentation:

- **Point-in-Time (PIT) vs. Through-the-Cycle (TTC)**: The model is calibrated to a specific historical period; TTC adjustment would be required for Basel II regulatory capital purposes
- **No macroeconomic conditioning**: GDP growth, unemployment rate, and credit cycle indicators are absent — a production-grade IRB model would incorporate these to satisfy IFRS 9 scenario requirements
- **Reject inference not applied**: The portfolio reflects only approved loans; the model may underestimate risk for borrower segments historically rejected by the lender
- **LGD bounded estimation**: OLS does not naturally bound predictions to [0, 1]; post-prediction clipping is a pragmatic workaround, not a statistically rigorous solution
- **Static snapshot**: The dataset reflects origination-time characteristics only; behavioral scoring using dynamic repayment trajectories would substantially improve discriminatory power

---

## Data

This project uses the **LendingClub Loan Data** available on Kaggle:

> [https://www.kaggle.com/datasets/wordsforthewise/lending-club](https://www.kaggle.com/datasets/wordsforthewise/lending-club)

Download `loan.csv` and place it in the `data/` directory. The file is excluded from this repository via `.gitignore` due to size.

---

## Requirements

```
pandas>=1.3
numpy>=1.21
scikit-learn>=1.0
statsmodels>=0.13
matplotlib>=3.4
seaborn>=0.11
scipy>=1.7
```

Install with:

```bash
pip install -r requirements.txt
```

---

## Regulatory Context

This model is developed in alignment with the **Basel II Internal Ratings-Based (IRB) Approach** for retail credit exposures. The IRB framework requires banks to estimate their own PD, LGD, and EAD inputs, subject to minimum standards for data quality, model validation, and documentation. This project demonstrates awareness of those standards even in a non-production context.

---

## Author

**Harsh**
M.A. Economics — Jawaharlal Nehru University | B.Tech ECE — JIIT Noida
Specialization: Credit Risk Analytics, Quantitative Risk Modeling

---

*This project is for portfolio and educational purposes. LendingClub data is used under Kaggle's terms of service.*
