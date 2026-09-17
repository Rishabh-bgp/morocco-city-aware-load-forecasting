# City-Aware Short-Term Load Forecasting and Peak Detection from High-Resolution Smart Meter Data in Moroccan Urban Grids

This repository presents an empirical study of short-term electricity load modelling on a high-resolution smart-meter corpus collected across Moroccan urban feeders. The work treats load both as a continuous signal to be forecast and as a discrete operating regime to be classified, and it evaluates linear baselines against tuned ensemble models under a common temporal feature set.

The motivating claim is simple: urban demand is structured by the calendar. Hour of day, day of week, and seasonality are not mere covariates; they are the principal observable drivers available when weather, tariff, and appliance-level metadata are absent. The analysis tests how far calendar features alone can carry two operational tasks—point forecasting of zone load and detection of low / medium / high demand states—on a 10-minute Moroccan dataset.

---

## Problem statement

Given a multivariate time series of zone-level active load \(L_{z,t}\) sampled at ten-minute resolution, the study addresses two complementary problems.

1. **Short-term load forecasting (regression).**  
   Predict continuous load for a designated target feeder (Zone 1) from calendar-derived features extracted at time \(t\).

2. **Peak / regime detection (classification).**  
   Map the same target series onto three quantile-defined operating states—Low, Medium, High—and recover the state from the same calendar representation.

The dual formulation mirrors utility practice: dispatch and reserve planning require a numeric forecast; demand-response and congestion flags require a regime label.

---

## Data

**Source.** M. Bensalah and A. Hair, *High-Resolution Load Dataset from Smart Meters Across Various Cities in Morocco*, UCI Machine Learning Repository, 2025.  
DOI: [10.24432/C5FC8R](https://doi.org/10.24432/C5FC8R)

| Attribute | Value |
|---|---|
| Temporal grain | 10 minutes |
| Observations | 88,890 |
| Geographic structure | Five urban zones (`zone1`–`zone5`) |
| Calendar span | From 2022-09-14 onward |
| Missingness | None observed in the published extract |
| Target used here | `zone1` (kW-scale feeder load) |

Zones 2–5 are retained in the raw table for exploratory comparison but are excluded from the feature matrix in the reported models, so that performance is attributable to time structure rather than contemporaneous load leakage from neighbouring feeders.

---

## Feature construction

The raw timestamp is decomposed into a compact calendar basis and then discarded:

| Feature | Semantics |
|---|---|
| `year` | Secular level |
| `month` | Seasonal cycle |
| `day` | Intra-month position |
| `hour` | Diurnal cycle |
| `dayofweek` | Weekly occupancy pattern |
| `dayofyear` | Fine seasonal index |

No lag features, rolling statistics, weather, or holiday calendars are introduced in the present notebook. The design is intentionally austere: it isolates how much of Moroccan urban load is recoverable from clock and calendar alone.

For classification, Zone 1 is discretised with equal-frequency tertiles:

```text
zone1_category = qcut(zone1, q=3)  →  {Low, Medium, High}
```

The resulting labels are balanced by construction.

---

## Experimental design

| Stage | Protocol |
|---|---|
| Hold-out | 80 / 20 train–test split (`random_state=42`) |
| Classification split | Stratified on the tertile label |
| Scaling | Standardisation of calendar features prior to ensemble training |
| Baselines | Ordinary least squares (regression); one-vs-rest logistic regression (classification) |
| Primary models | Random Forest regressor and classifier |
| Tuning | `GridSearchCV`, 3-fold CV |
| Confirmation | 5-fold shuffled CV on the tuned estimators |

**Search space (both forests).**

- `n_estimators` ∈ {50, 100, 150}
- `max_features` ∈ {0.6, 0.8, 1.0}
- `min_samples_leaf` ∈ {1, 2}

Scoring: \(R^2\) for regression; accuracy for classification.

---

## Results

### Regression — Zone 1 load

| Estimator | Notes | Test MAE | Test RMSE | Test \(R^2\) |
|---|---|---|---|---|
| Linear Regression | Calendar OLS baseline | — | — | weaker than RF |
| Random Forest (default, 100 trees) | Untuned | — | — | high |
| **Tuned Random Forest** | `n_estimators=150`, `max_features=0.6`, `min_samples_leaf=1` | **2.26** | **4.51** | **0.94** |

5-fold CV on the tuned regressor:

- Mean \(R^2\): **0.94** (± 0.00)
- Mean MAE: **2.27** (± 0.06)
- Mean RMSE: **4.51** (± 0.13)

The near-zero fold dispersion indicates that the calendar-to-load mapping is stable across random partitions of this corpus.

### Classification — Low / Medium / High

| Estimator | Test accuracy |
|---|---|
| Logistic Regression (OvR) | baseline |
| Random Forest (default) | high |
| **Tuned Random Forest** | **0.91** |

Best hyperparameters: `n_estimators=100`, `max_features=1.0`, `min_samples_leaf=1`.  
Grid-search CV accuracy: 0.91.  
5-fold CV accuracy: **0.91** (± 0.00).

Per-class test performance of the tuned forest (support 5,926 per class):

| Class | Precision | Recall | F1 |
|---|---|---|---|
| High | 0.93 | 0.93 | 0.93 |
| Low | 0.93 | 0.93 | 0.93 |
| Medium | 0.88 | 0.88 | 0.88 |

The Medium band is the hardest class, as expected at the two quantile boundaries.

Feature-importance plots in the notebook attribute most predictive mass to `hour` and `dayofyear`, with `dayofweek` contributing the weekly occupancy residual. That ranking is consistent with a climate and lifestyle regime in which diurnal cooling/heating and seasonal demand dominate feeder load.

---

## Interpretation

Two conclusions follow from the numbers.

1. **Calendar sufficiency.** On this Moroccan extract, a six-dimensional time encoding plus a shallow forest recovers 94% of Zone 1 variance and 91% of tertile state. That is strong evidence that the published series is highly periodic and relatively free of unstructured shocks at the 10-minute grain.

2. **Operational dual-use.** The same representation supports both a numeric forecast (for energy scheduling) and a three-state peak flag (for feeder alerts). The Medium-class F1 gap is the natural place for a subsequent threshold or cost-sensitive refinement.

The study does **not** claim a state-of-the-art multivariate forecaster. It claims a clean, reproducible measurement of how far city-level Moroccan smart-meter load can be explained without exogenous weather or lag stacks.

---

## Limitations and extensions

- Features are calendar-only; neighbouring zones are not used as contemporaneous or lagged covariates, despite the “city-aware” framing of the title.
- The train–test split is random rather than strictly causal (no rolling origin). For operational STLF this should be replaced by a temporal walk-forward protocol.
- Peak labels are in-sample tertiles of Zone 1, not an engineering peak defined by transformer rating.
- No weather, holiday, or Ramadan indicators are merged.

Natural extensions: lag and rolling-load features; multi-zone multi-output forests or gradient boosting; weather fusion; walk-forward validation; probabilistic intervals; explicit peak-over-threshold labels.

---

## Repository contents

```text
.
├── City_Aware_Short_Term_Load_Forecasting_and_Peak_Detection_from_High_Resolution_Smart_Meter_Data_in_Moroccan_Urban_Grids.ipynb
├── Data Morocco.xlsx          # obtain from UCI (not always redistributed)
└── README.md
```

The notebook is organised as a linear thesis chapter:

1. Ingest and integrity checks  
2. Calendar feature extraction  
3. Linear / logistic baselines  
4. Exploratory visualisations  
5. Random Forest regression and classification  
6. Feature importance  
7. Diagnostic plots (actual vs predicted; confusion matrix)  
8. Grid search  
9. Five-fold confirmation of the tuned models  

---

## Reproducibility

```bash
python -m pip install pandas numpy scikit-learn matplotlib seaborn openpyxl
```

Place `Data Morocco.xlsx` at the path expected by the first code cell (or edit that path). Execute the notebook top to bottom. All model seeds are fixed at 42.

---

## Citation

If you use this analysis or the underlying corpus, please cite the dataset authors:

> M. Bensalah and A. Hair. “High-Resolution Load Dataset from Smart Meters Across Various Cities in Morocco.” UCI Machine Learning Repository, 2025. https://doi.org/10.24432/C5FC8R

---

*This README describes the empirical programme encoded in the accompanying notebook. It is written as a compact methods-and-results chapter, not as a product landing page.*
