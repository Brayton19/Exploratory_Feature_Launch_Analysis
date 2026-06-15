# Feature Adoption Analysis

**Author:** Brayton Noll  
**Date:** 2026

---

## Overview

This project assesses the causal impact of a product feature launch on two engagement metrics — Monthly Active Users (MAU) and Power Active Users (PAU) — using a panel dataset of approximately 1.6 million user-month observations. The analysis combines time-series causal inference with a within-user panel event study.

---

## Analysis Structure

### 1. Data Loading and Cleaning

- NaN imputation for binary feature columns (validated as safe via neighbor-zero pattern analysis)
- Outlier clipping for values outside the binary [0, 1] range
- Panel consistency checks: duplicate rows, signup date ordering, missing months

### 2. Exploratory Analysis

- Monthly user counts, adoption rates, and new sign-up trends
- Interrupted time-series test for Product B independence — **finding:** Products A and B are correlated at launch, ruling out B as a control series

### 3. Augmented Bayesian Structural Time Series (aBSTS)

A local-linear trend plus annual seasonal state-space model fitted on the pre-period only. Post-period innovations are drawn from the fitted priors, propagating the final pre-period state forward with full posterior uncertainty to yield a counterfactual.

Both MAU and PAU models share a single `run_absts()` function parameterized by `target_col` and `priors`, making it straightforward to apply the same specification to other targets or re-run with alternative priors.

**Finding:** No statistically sufficient evidence that the product launch affected aggregate MAU or PAU adoption rates. Product C launched concurrently with Product A, so their aggregate effects are not separately identifiable in this analysis.

### 4. Within-User Event Study

Panel logistic regression (implemented as a linear probability model) with user and month fixed effects. Tests whether Product A usage at month *t* predicts PAU activation (0 → 1) at *t + lead*, controlling for Products B and C and MAU status.

User fixed effects are absorbed via within-user demeaning (the Mundlak within estimator). Standard errors are clustered by user.

**Finding:** Within a given user, Product A usage is associated with approximately a 1.5 percentage-point increase in the probability of PAU activation three months later — a small absolute effect on a low base rate, but statistically significant after controlling for other features and time trends. This is a Granger-style result (predictive precedence), not a causal identification.

---

## Key Design Choices

| Choice | Rationale |
|---|---|
| aBSTS over synthetic control | No valid control unit available — Products A and C launched simultaneously, and Product B is correlated with A |
| Linear probability model over logit | Within-user demeaning cleanly absorbs fixed effects without estimating per-user dummies; more robust on rare, unbalanced outcomes |
| Non-centered parameterization for innovations | Decouples geometry of standard normals from sigma parameters, improving NUTS sampling |
| Gamma priors for `sigma_level` / `sigma_obs` | Moves density away from zero compared to HalfNormal, reducing divergences |

---

## Setup

**Requirements:** Python >= 3.12

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter lab
```

Open `feature_adoption_analysis.ipynb` and run all cells in order.

> **Note:** The aBSTS models use 4 chains × 2,000 draws × 2,000 tuning steps. Expect ~10–20 minutes per model on a modern laptop.

---

## Repository Contents

```
feature-adoption-analysis/
├── README.md
├── requirements.txt
├── .python-version
├── data.csv                          # Panel data (~1.6M rows, 9 columns)
└── feature_adoption_analysis.ipynb   # Main analysis notebook
```

### Data Schema

| Column | Type | Description |
|---|---|---|
| `client_id` | UUID string | Unique user identifier |
| `market_code` | string | Market / country code |
| `onboard_date` | date | User registration date |
| `period` | date | Month-end date of observation |
| `product_A` | binary | Product A usage that month |
| `product_B` | binary | Product B usage that month |
| `product_C` | binary | Product C usage that month |
| `PAU` | binary | Power Active User status |
| `MAU` | binary | Monthly Active User status |
