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

## 🔗 Other repositories by Rishabh Aryan

| Repository | Description | Live |
|---|---|---|
| [100-days-of-python](https://github.com/Rishabh-bgp/100-days-of-python) | My original, offline, no-API-key solutions to all 100 projects of the 100 Days of Code Python bootcamp curriculum… | — |
| [Agentic-Engineering-Ebook](https://github.com/Rishabh-bgp/Agentic-Engineering-Ebook) | Agentic Engineering is a practical notebook-book on specifying, generating, verifying, and refining multi-tenant… | — |
| [beed-ml-study](https://github.com/Rishabh-bgp/beed-ml-study) | Provenance audit and leakage-aware benchmark of seizure-type classifiers on the Bangalore EEG Epilepsy Dataset — IIT… | — |
| [Brain-Computer-Interfaces-and-Cognitive-Computing-An-Introduction](https://github.com/Rishabh-bgp/Brain-Computer-Interfaces-and-Cognitive-Computing-An-Introduction) | Brain–computer interfaces (BCIs) link neural activity to external devices, translating brain signals via algorithms… | — |
| [building-a-virtual-cell-in-python](https://github.com/Rishabh-bgp/building-a-virtual-cell-in-python) | A cell has a normal gene-activity pattern. Knock out one gene and that pattern changes. This project trains a small… | — |
| [Chronic-Kidney-Disease-Stage-Identification-in-HIV-Infected-Patients-Using-Supervised-ML](https://github.com/Rishabh-bgp/Chronic-Kidney-Disease-Stage-Identification-in-HIV-Infected-Patients-Using-Supervised-ML) | ML is transforming Chronic Kidney Disease (CKD) detection in HIV patients, catching early asymptomatic stages to… | — |
| [cobratate](https://github.com/Rishabh-bgp/cobratate) | Cobratate, an esoteric imperative programming language. Tate-flavored syntax, ordinary semantics, and a Python… | [open](https://rishabh-bgp.github.io/cobratate/) |
| [coda-synth](https://github.com/Rishabh-bgp/coda-synth) | CoDA-Synth: stylometric detection of LLM-rewritten dark-web documents with sparse TF-IDF + LinearSVC. Research lab,… | [open](https://shale-ember-delta-kite.grok.me/) |
| [IIT-ROPAR-GENAI-FOUNDATIONS](https://github.com/Rishabh-bgp/IIT-ROPAR-GENAI-FOUNDATIONS) | Free open-source Generative AI foundations ebooks (2026) covering Claude, Grok, Gemini, Cursor, OpenAI and more —… | — |
| [iris-classification](https://github.com/Rishabh-bgp/iris-classification) | Iris flower classification in Python: scikit-learn model comparison, tuning and evaluation, FastAPI /predict API,… | [open](https://rishabh-bgp.github.io/iris-classification/) |
| [kbc-game](https://github.com/Rishabh-bgp/kbc-game) | Kaun Banega Crorepati (KBC) quiz game for the terminal in Python: 16-question prize ladder, safe levels and 4… | — |
| [knn-classifier](https://github.com/Rishabh-bgp/knn-classifier) | Simple K-Nearest Neighbors classifier on the classic Iris dataset using scikit-learn. Loads data, trains the model,… | — |
| [linear-regression-model-](https://github.com/Rishabh-bgp/linear-regression-model-) | Supervised linear regression on the scikit-learn diabetes dataset. A single feature (BMI) is used to predict… | — |
| [logistic-regression](https://github.com/Rishabh-bgp/logistic-regression) | This graduate capstone applies multi-class logistic regression to the Iris dataset. It includes EDA, stratified… | — |
| [logistics-website-capstone](https://github.com/Rishabh-bgp/logistics-website-capstone) | Logistics website capstone: Kargonaut Logistics, a premium freight site mixing claymorphism, glassmorphism and… | [open](https://rishabh-bgp.github.io/logistics-website-capstone/) |
| [lovelight-leap](https://github.com/Rishabh-bgp/lovelight-leap) | 💖 Lovelight Leap: a love-themed 2D platformer in vanilla JS canvas. 4 worlds, a boss you cheer up, original… | [open](https://rishabh-bgp.github.io/lovelight-leap/) |
| [machine-learning-part-1](https://github.com/Rishabh-bgp/machine-learning-part-1) | Machine learning in Python with scikit-learn: heart disease classification and medical insurance cost regression… | — |
| [Meeting-of-Machines-Consumer-Agents-Seller-Agents-and-Hidden-Fulfilment-Risk-in-Online-Retail](https://github.com/Rishabh-bgp/Meeting-of-Machines-Consumer-Agents-Seller-Agents-and-Hidden-Fulfilment-Risk-in-Online-Retail) | Agentic commerce lets shopping and seller agents place orders without showing operational truth. This IS case argues… | — |
| [NIELIT-DATA-SCIENCE](https://github.com/Rishabh-bgp/NIELIT-DATA-SCIENCE) | Hands-on NumPy, Pandas, and Matplotlib Jupyter notebooks for the NIELIT Data Science curriculum — practical,… | — |
| [NIELIT-MACHINE-LEARNING-PROJECTS](https://github.com/Rishabh-bgp/NIELIT-MACHINE-LEARNING-PROJECTS) | Nineteen end-to-end machine learning projects: classification, regression, clustering, and recommendation, with… | — |
| [NIELIT-PYTHON-](https://github.com/Rishabh-bgp/NIELIT-PYTHON-) | Open-source Python teaching notebooks: a language reference and five capstone programs, with detailed documentation.… | — |
| [pan-cancer-classification](https://github.com/Rishabh-bgp/pan-cancer-classification) | Leakage, equivalence, conformal reliability and data efficiency on the UCI TCGA Pan-Cancer RNA-Seq dataset — IIT… | — |
| [Predicting-Hot-Events-in-the-Early-Period-through-Bayesian-Model-for-Social-Networks](https://github.com/Rishabh-bgp/Predicting-Hot-Events-in-the-Early-Period-through-Bayesian-Model-for-Social-Networks) | This notebook details the implementation and evaluation of a Bayesian perspective Early stage Event Prediction… | — |
| [prediction-as-power](https://github.com/Rishabh-bgp/prediction-as-power) | Laboratory and working paper on prediction as power in European electricity markets. A DE-like hourly simulator… | — |
| [snakes-and-ladders](https://github.com/Rishabh-bgp/snakes-and-ladders) | 🐍🪜 Original Snakes & Ladders browser game: hand-drawn SVG snakes and ladders, 3D dice, 2-4 humans or bots, sound,… | [open](https://rishabh-bgp.github.io/snakes-and-ladders/) |
| [spam-scan](https://github.com/Rishabh-bgp/spam-scan) | Explainable spam, phishing & Indian-scam detector for SMS/email (English, Hindi, Hinglish): TF-IDF + Logistic… | [open](https://spam-scan.onrender.com) |
| [vedic-math-hybrid-kt](https://github.com/Rishabh-bgp/vedic-math-hybrid-kt) | Hybrid Vedic Math AI: rule-based sutras (Urdhva-Tiryagbhyam, Nikhilam, Yavadunam) plus RF classifier that cleans… | — |
| [wind-turbine-power-model](https://github.com/Rishabh-bgp/wind-turbine-power-model) | Single-turbine power curves + wind-farm simulation. Weibull wind stats, hourly NASA-style speed data, and… | — |
| [zoo-website-capstone](https://github.com/Rishabh-bgp/zoo-website-capstone) | Capstone project: a fictional zoo website built twice — a beginner-style simple version and a cinematic advanced… | [open](https://rishabh-bgp.github.io/zoo-website-capstone/) |
