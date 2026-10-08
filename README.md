# Merchant Transaction Fraud Detection (Nigerian Payments, Synthetic Data)

## Business context
A payment processor serving merchants across Nigeria and nine other African markets loses money in two ways: fraud that gets through, and good payments that get held up for manual review. Risk operations can only review a small share of payments, and the legacy rule-based risk score used today misses a large part of the fraud.

## Goal
Build a model that scores each payment at the moment it happens, so the review team can concentrate on the riskiest 2% to 5% of payments, catch most of the fraud, and keep false alarms low.

## Tasks
1. Define what is being predicted: a per-payment fraud decision, made only with information available at payment time.
2. Audit and clean the data, and find any columns that would leak the answer.
3. Explore the data to find which behaviours separate fraud from legitimate activity.
4. Engineer features that use only a merchant's past activity.
5. Train and compare models against the legacy score.
6. Evaluate in business terms: fraud caught, false alarms and cost, not only accuracy.

## Data
A synthetic dataset built with a custom simulator: 350,000 transactions from about 35,000 merchants across a full year, with a fraud rate of about 2%. Fields cover sector, merchant category code, country, payment channel (POS, instant transfer, USSD/wallet), amount in naira, new-merchant flag, the legacy risk score, and dispute outcome.

## Work done
- **Leakage review:** `dispute_outcome` is only known weeks after a payment, so it was excluded. The legacy score was tested with and without it.
- **Exploratory analysis:** fraud rates with confidence intervals, information value for each categorical feature (with an out-of-sample check for the high-cardinality category code), time-of-day patterns, merchant behaviour, interactions, and an anomaly-detection check.
- **Point-in-time features:** transactions in the last hour, day and week, time since the merchant's previous payment, merchant age, amount compared with the merchant's own history, and cyclical time-of-day encodings.
- **Validation design:** a chronological train, validation and test split with gaps between blocks, so the model is always scored on later data than it learned from.
- **Models:** the legacy score alone, logistic regression, and LightGBM, with and without the legacy score and with and without class weights. Feature groups were removed one at a time to see what drives performance.
- **Evaluation:** PR-AUC (the right measure when fraud is about 2% of payments), precision and recall at fixed review rates, and a cost comparison with an assumed review cost per payment.

## Results
Validation block, reviewing the top 2% of payments:

| Model | PR-AUC | Precision | Recall |
|---|---|---|---|
| Legacy risk score alone | 0.32 | 36% | 32% |
| Logistic regression | 0.76 | 72% | 65% |
| **LightGBM** | **0.92** | **88%** | **79%** |

Per 10,000 payments (about 221 fraudulent), LightGBM review outcomes:

| Payments reviewed | Frauds caught | False alarms | Frauds missed |
|---|---|---|---|
| 1% | 96 | 3 | 125 |
| 2% | 175 | 24 | 46 |
| 3% | 211 | 89 | 11 |
| 5% | 221 | 279 | 0 |

**What drives the result**
- Time of day matters most: fraud rates exceed 30% between midnight and 4am, and removing the time features lowered PR-AUC by 0.07. No other feature group changed it by more than 0.01.
- New merchants are over three times riskier than established ones in every channel, and instant transfer and USSD/wallet payments carry about three times the fraud rate of POS.
- The legacy score adds nothing once these features are present (PR-AUC 0.9240 without it, 0.9236 with it), so the model can run without that dependency.
- Class weighting did not help (PR-AUC 0.904 against 0.924 unweighted), so the final model is unweighted.

**Cost view.** Choosing the score cut-off by cost (₦2,000 per review) flags about 3% of payments, catches 98% of fraud, and cuts validation-block losses from ₦125.7M with no model to ₦2.4M. On the held-out test block, every fraudulent merchant was flagged, typically on its first fraudulent payment.

## Notes
- The data is synthetic with strong built-in fraud patterns (night-time activity, bursts, short-lived fraud merchants), so absolute scores are higher than a real portfolio would give. The comparisons between approaches are the meaningful part.
- The naira figures depend on the assumed review cost and exclude customer-friction costs.
- The validation block was also used for early stopping and for choosing the cut-off, so its figures are slightly optimistic.

## Repository contents
- `fraud_detection.ipynb`: analysis, feature engineering, modelling and evaluation
