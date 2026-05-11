# IFRS 9 Credit Risk PD Modeling, ECL and Economic Capital Project

## Project Overview

This project builds a simplified end-to-end credit risk framework using Python. The workflow starts with a Probability of Default (PD) model, extends the predicted PDs into a simplified IFRS 9 Expected Credit Loss (ECL) calculation, and finishes with a Monte Carlo economic capital simulation using Value-at-Risk (VaR).

The project is designed as an interview-ready credit risk modeling project. It demonstrates how borrower, loan, and bureau variables can be used to estimate default risk and how those estimates can be connected to expected loss and unexpected loss concepts.

## Business Objective

The main objective is to answer three credit risk questions:

1. **PD Modeling:** What is the probability that a borrower will default?
2. **IFRS 9 ECL:** What is the expected credit loss for the loan portfolio?
3. **Economic Capital:** How much unexpected-loss capital is needed under a severe simulated loss scenario?

## Project Components

```text
Data preprocessing and feature engineering
        ↓
Logistic Regression PD model
        ↓
Model validation and class imbalance handling
        ↓
IFRS 9 staging and ECL calculation
        ↓
Monte Carlo portfolio loss simulation
        ↓
VaR and economic capital estimation
```

## Dataset

The project uses a vehicle loan default dataset with borrower, loan, and bureau-related information.

Key fields include:

- `loan_default` — target variable, where 1 means default and 0 means non-default
- `disbursed_amount` — loan amount disbursed
- `asset_cost` — asset or vehicle cost
- `ltv` — loan-to-value ratio
- `employment_type` — borrower employment category
- `date_of_birth` — borrower date of birth
- `disbursaldate` — loan disbursal date
- `perform_cns_score` — bureau score
- `pri_no_of_accts` — number of primary accounts
- `pri_active_accts` — number of active primary accounts
- `pri_overdue_accts` — number of overdue primary accounts
- `delinquent_accts_in_last_six_months` — recent delinquency indicator
- `no_of_inquiries` — number of credit inquiries

## Preprocessing and Feature Engineering

The following preprocessing and feature engineering steps were applied:

### General preprocessing

- Cleaned column names by converting them to lowercase and replacing spaces/dots with underscores.
- Filled missing `employment_type` values with `Unknown`.
- Converted `date_of_birth` and `disbursaldate` into datetime format.
- Created customer age at the time of loan disbursal.
- Converted account age variables from text format into total months.
- Dropped high-cardinality operational ID columns to reduce overfitting.
- Applied log transformation to skewed amount variables.
- Filled missing numeric values using training-set medians.
- Standardized selected amount variables for logistic regression.
- Used a stratified train-test split to preserve the default rate.

### Credit-risk feature engineering

The following credit-risk-specific features were created:

- `no_bureau_history` — flag for borrowers with bureau score equal to zero
- `has_overdue_account` — flag for borrowers with at least one overdue account
- `recent_delinquency_flag` — flag for recent delinquency in the last six months
- `high_inquiry_flag` — flag for borrowers with three or more credit inquiries
- `active_account_ratio` — active accounts divided by total accounts
- `overdue_account_ratio` — overdue accounts divided by total accounts
- `primary_utilization` — current balance divided by sanctioned amount

These variables help capture thin-file borrowers, repayment stress, recent delinquency, credit-seeking behavior, and credit utilization.

## PD Model

A logistic regression model was used as the baseline PD model because it is interpretable and produces probability outputs.

Two imbalance-handling methods were compared:

1. **Class-weighted Logistic Regression**
2. **SMOTE + Logistic Regression**

GridSearchCV was used to tune the regularization parameter `C`.

### Selected model

The selected model was:

```text
Class-weighted Logistic Regression
Best C = 10
```

This model was selected because it gave the better ROC-AUC compared with the SMOTE model.

## PD Model Results

Validation results for the selected class-weighted logistic regression model:

| Metric | Value |
|---|---:|
| Accuracy | 0.5302 |
| Precision | 0.2593 |
| Recall | 0.6269 |
| F1 Score | 0.3668 |
| ROC-AUC | 0.5883 |

Confusion matrix:

|  | Predicted Non-default | Predicted Default |
|---|---:|---:|
| Actual Non-default | 18,381 | 18,128 |
| Actual Default | 3,777 | 6,345 |

### Interpretation

The model catches about 62.7% of actual defaulters, which is useful for a credit-risk baseline. However, the precision is low, meaning many borrowers predicted as default do not actually default. The ROC-AUC of approximately 0.59 indicates modest discriminatory power, so this should be treated as a baseline model rather than a production-grade PD model.

## IFRS 9 Expected Credit Loss Framework

The predicted probabilities from the selected PD model were used as 12-month PD estimates.

The simplified IFRS 9 ECL formula used was:

```text
ECL = PD × LGD × EAD
```

### Key assumptions

| Component | Project assumption |
|---|---|
| PD | Predicted probability from logistic regression |
| EAD | Raw `disbursed_amount` used as proxy |
| LGD | Fixed assumption of 40% |
| Stage 1 | Performing loans using 12-month PD |
| Stage 2 | High-risk loans using lifetime PD |
| Stage 3 | Actual defaulted loans using PD = 100% |
| Lifetime PD | `1 - (1 - PD_12M)^3` |
| Remaining life | Simplified 3-year assumption |

### Staging logic

The simplified staging rules were:

- **Stage 1:** Performing loans with no high-risk signal
- **Stage 2:** Loans with `PD_12M >= 20%`, overdue account, or recent delinquency
- **Stage 3:** Loans where `actual_default = 1`

In a real IFRS 9 framework, Stage 2 would be based on Significant Increase in Credit Risk (SICR) relative to origination risk, not only a fixed PD threshold.

## IFRS 9 Results

| Stage | Number of Loans | Total EAD | Average 12M PD | Average IFRS9 PD | Total ECL | Actual Default Rate |
|---|---:|---:|---:|---:|---:|---:|
| Stage 1 | 51 | 1,823,054 | 0.1700 | 0.1700 | 124,394 | 0.0000 |
| Stage 2 | 36,458 | 1,962,283,210 | 0.4924 | 0.8593 | 678,819,800 | 0.0000 |
| Stage 3 | 10,122 | 567,828,295 | 0.5159 | 1.0000 | 227,131,300 | 1.0000 |

Portfolio summary:

| Metric | Value |
|---|---:|
| Total Loans | 46,631 |
| Total EAD | 2,531,934,559 |
| Total ECL | 906,075,485.37 |
| Average IFRS9 PD | 0.8891 |
| Average LGD | 0.4000 |

### IFRS 9 result interpretation

The total portfolio ECL is approximately **906.1 million**. Most of the portfolio is assigned to Stage 2 because the simplified staging rule is conservative and the class-weighted logistic regression model produces relatively high PD estimates. Stage 3 loans use PD equal to 100%, which increases the ECL for actual defaulted loans.

This IFRS 9 framework is simplified. A real company would use origination PD, current PIT PD, SICR rules, macroeconomic scenarios, proper EAD, modelled LGD, and discounting.

## Economic Capital and Monte Carlo Simulation

The economic capital section estimates unexpected credit loss using Monte Carlo simulation.

The simulation uses:

- `PD_12M` as one-year default probability
- `EAD` as exposure amount
- `LGD` as loss severity
- Uniform random numbers to simulate whether each borrower defaults
- 10,000 simulations to create a portfolio loss distribution

### Monte Carlo logic

For each simulation:

1. Generate one random number between 0 and 1 for each borrower.
2. If the random number is less than the borrower PD, the borrower defaults.
3. If the borrower defaults, loss is calculated as `LGD × EAD`.
4. Losses are summed across all borrowers to get one simulated portfolio loss.
5. This process is repeated 10,000 times.
6. The resulting simulated losses form a loss distribution.
7. VaR is taken from the 99th percentile of the loss distribution.
8. Economic capital is calculated as `VaR - Expected Loss`.

## Economic Capital Results

| Metric | Value |
|---|---:|
| Scenario | Base Scenario |
| Number of Simulations | 10,000 |
| Expected Loss - Monte Carlo | 509,515,732.13 |
| Expected Loss - Formula | 509,519,183.66 |
| VaR 99% | 515,081,688.48 |
| Economic Capital | 5,565,956.35 |

### Economic capital interpretation

The Monte Carlo expected loss is very close to the formula expected loss, which confirms that the simulation is working correctly.

The 99% VaR is approximately **515.1 million**, meaning that 99% of simulated portfolio losses were below this amount. Economic capital is approximately **5.6 million**, calculated as:

```text
Economic Capital = VaR 99% - Expected Loss
```

This economic capital figure is smaller than the IFRS 9 ECL because the two sections use different PD concepts:

- IFRS 9 ECL uses `IFRS9_PD`, where Stage 2 uses lifetime PD and Stage 3 uses 100% PD.
- Economic capital uses one-year `PD_12M` for a one-year simulation.

Also, the Monte Carlo simulation assumes independent defaults, fixed LGD, fixed EAD, and no macroeconomic default correlation. A more advanced model would include correlated defaults, downturn LGD, macroeconomic scenarios, and 99.9% VaR.

## Key Assumptions

This project uses several simplifying assumptions:

1. Logistic regression predicted probability is treated as 12-month PD.
2. LGD is fixed at 40% due to lack of recovery data.
3. EAD is proxied using original disbursed amount.
4. Stage 2 is assigned using a simplified PD/delinquency rule.
5. Lifetime PD is approximated using a 3-year survival formula.
6. Stage 3 is assigned using actual default in the validation dataset.
7. Monte Carlo assumes independent defaults.
8. Economic capital uses 99% VaR, not Basel-style 99.9% VaR.

## Limitations

This is a simplified learning project and not a production-grade IFRS 9 or Basel model.

Important limitations:

- No true recovery data, so LGD is assumed.
- No actual outstanding balance at default, so EAD is proxied.
- No origination PD, so true SICR cannot be measured.
- No macroeconomic variables are included.
- No PIT/TTC PD conversion is performed.
- No discounting of future ECL cash flows.
- No correlated default simulation.
- Logistic regression ROC-AUC is modest, so the PD model is a baseline.

## Possible Improvements

Future improvements could include:

- Add calibration curve and Brier score for PD calibration.
- Add PD decile analysis.
- Test tree-based models such as Random Forest, XGBoost, or LightGBM.
- Calibrate predicted PDs before using them for ECL.
- Use real outstanding balance for EAD.
- Build an LGD model if recovery data is available.
- Add macroeconomic scenario analysis.
- Use a correlated default simulation using a Vasicek-style one-factor model.
- Use 99.9% VaR for Basel-style capital comparison.
- Add model monitoring metrics such as PSI and stability analysis.

## How to Run the Project

1. Clone the repository.
2. Place the dataset files in the project folder.
3. Install required Python libraries.
4. Open `PD Model.ipynb`.
5. Run all cells from top to bottom.

Required packages:

```text
numpy
pandas
matplotlib
seaborn
scikit-learn
imbalanced-learn
```

Install using:

```bash
pip install numpy pandas matplotlib seaborn scikit-learn imbalanced-learn
```

## Repository Structure

```text
.
├── PD Model.ipynb
├── train.csv
├── data_dictionary.csv
└── README.md
```

## Interview Summary

This project can be explained as follows:

> I built an interpretable baseline credit risk PD model using logistic regression. I handled missing values, engineered borrower and bureau risk features, managed class imbalance using class weighting and SMOTE, and selected the class-weighted model based on ROC-AUC. I then used the predicted PDs in a simplified IFRS 9 ECL framework, assigning loans into Stage 1, Stage 2, and Stage 3 and calculating ECL using PD, LGD, and EAD. Finally, I simulated portfolio losses using Monte Carlo methods and estimated VaR-based economic capital as the unexpected-loss buffer above expected loss.

## Disclaimer

This project is for educational and interview demonstration purposes only. It simplifies several real-world IFRS 9 and Basel requirements and should not be used as a regulatory credit risk model.
