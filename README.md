# 🎬 Cinema Audience Forecasting

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python&logoColor=white)](https://www.python.org/)
[![LightGBM](https://img.shields.io/badge/LightGBM-Gradient%20Boosting-brightgreen?logo=lightgbm)](https://lightgbm.readthedocs.io/)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-ML%20Pipeline-orange?logo=scikit-learn&logoColor=white)](https://scikit-learn.org/)
[![Kaggle](https://img.shields.io/badge/Kaggle-Competition-20BEFF?logo=kaggle&logoColor=white)](https://www.kaggle.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Complete-success)]()

> **Predicting daily cinema audience counts across multiple theaters using LightGBM with time-series feature engineering, cyclic encodings, calibration, and theater-level bias correction.**

---

## 📋 Table of Contents

- [Business Problem](#-business-problem)
- [Dataset](#-dataset)
- [Project Structure](#-project-structure)
- [Workflow Pipeline](#-workflow-pipeline)
- [Feature Engineering](#-feature-engineering)
- [Models Evaluated](#-models-evaluated)
- [Results](#-results)
- [Tech Stack](#-tech-stack)
- [Setup & Usage](#-setup--usage)
- [Future Improvements](#-future-improvements)

---

## 🎯 Business Problem

Cinema chains need to forecast daily audience attendance per theater to optimize staffing, concessions inventory, and screen scheduling. Under-forecasting leads to poor customer experience; over-forecasting wastes operational resources.

This project builds a machine learning pipeline that predicts `audience_count` — the number of attendees per theater per day — using historical booking data from two booking systems (BookNow and CinePOS), theater metadata, and calendar features.

---

## 📦 Dataset

The dataset comes from the **Cinema Audience Forecasting Challenge** on Kaggle and contains six data sources:

| File | Description |
|------|-------------|
| `booknow_booking.csv` | Online ticket bookings (show datetime, tickets booked, theater ID) |
| `booknow_theaters.csv` | Theater metadata (type, area, latitude, longitude) |
| `booknow_visits.csv` | Ground truth — actual daily audience counts per theater |
| `cinePOS_booking.csv` | Point-of-sale (walk-in) ticket transactions |
| `cinePOS_theaters.csv` | CinePOS theater metadata |
| `movie_theater_id_relation.csv` | Mapping between BookNow and CinePOS theater IDs |
| `date_info.csv` | Calendar context (day-of-week, holidays, etc.) |

**Target variable:** `audience_count` — daily audience per theater

---

## 📁 Project Structure

```
cinema-audience-forecasting/
│
├── notebooks/
│   └── cinema_audience_forecasting.ipynb   # Main analysis notebook
│  
│
├── docs/
│   └── workflow.md                         # Detailed pipeline notes
│
├── README.md
├── requirements.txt
└── .gitignore
```

---

## 🔄 Workflow Pipeline

```
Raw Data Sources
      │
      ▼
Data Loading & Structure Understanding
  - booknow_booking, booknow_theaters, booknow_visits
  - cinePOS_booking, cinePOS_theaters
  - movie_theater_id_relation, date_info
      │
      ▼
Data Merging
  - Inner join bookings + theater metadata
  - Unify BookNow & CinePOS via ID mapping
  - Outer merge with visits (ground truth)
  - Left join with date_info calendar features
      │
      ▼
Exploratory Data Analysis
  - Correlation heatmap
  - Weekend vs weekday booking patterns
  - Top theater areas by ticket volume
  - Target variable distribution (right-skewed)
      │
      ▼
Feature Engineering
  - Date decomposition (month, day, week, quarter)
  - Cyclic encodings (sin/cos for day & month)
  - Lag features (lag_1, lag_2 from tickets_booked)
  - Rolling mean (roll_7: 7-day booking average)
  - Days since previous record per theater
  - Theater activity counter
  - Weekend binary flag
      │
      ▼
Time-Based Train/Validation Split (80/20)
      │
      ▼
Preprocessing Pipeline
  - LabelEncoder for categoricals (fit on full data)
  - SimpleImputer (mean) + StandardScaler for numerics
  - ColumnTransformer with passthrough for encoded cats
      │
      ▼
Model Training & Comparison
  ┌─────────────────────────────────┐
  │  Linear Regression              │
  │  Ridge Regression               │
  │  Decision Tree                  │
  │  Random Forest                  │
  │  LightGBM ← Best Model         │
  └─────────────────────────────────┘
      │
      ▼
Post-Processing (LightGBM output)
  - Isotonic calibration on validation residuals
  - Theater-level bias correction (40% shrinkage)
  - Blend with 7-day rolling baseline (90/10)
  - Clip predictions to [0, ∞), round to integer
      │
      ▼
submission.csv
```

---

## ⚙️ Feature Engineering

| Feature | Type | Description |
|---------|------|-------------|
| `month`, `day`, `weekofyear`, `quarter` | Date | Decomposed calendar fields |
| `is_weekend` | Binary | 1 if Saturday or Sunday |
| `sin_day`, `cos_day` | Cyclic | Circular encoding of day (period=31) |
| `sin_month`, `cos_month` | Cyclic | Circular encoding of month (period=12) |
| `lag_1`, `lag_2` | Lag | Previous 1 & 2 days' tickets booked per theater |
| `roll_7` | Rolling | 7-day rolling mean of tickets booked per theater |
| `days_since_prev` | Gap | Days elapsed since last recorded show per theater |
| `theater_activity` | Count | Cumulative show count per theater (activity proxy) |
| `had_booking` | Binary | Whether any booking occurred on that date |

---

## 🤖 Models Evaluated

All models were trained on an 80% time-based split and evaluated on the remaining 20% (future dates), respecting temporal ordering to prevent data leakage.

| Model | Notes |
|-------|-------|
| Linear Regression | Baseline; confirmed data non-linearity |
| Ridge Regression | L2 regularized linear model |
| Decision Tree | Shallow tree (max_depth=10) to reduce overfitting |
| Random Forest | Ensemble of trees, parallelized |
| **LightGBM** ✅ | **Final model** — gradient boosting with early stopping |

**LightGBM Configuration:**
- `objective='mae'`, `n_estimators=5000`, early stopping at 200 rounds
- `learning_rate=0.01`, `max_depth=6`, `num_leaves=63`
- `min_child_samples=200`, `subsample=0.8`, `colsample_bytree=0.7`
- L1/L2 regularization: `lambda_l1=2.0`, `lambda_l2=2.0`

---

## 📊 Results

> Results are from validation set (time-based, unseen future dates).

| Model | R² Score | RMSE | MAE |
|-------|----------|------|-----|
| Linear Regression | *(from notebook output)* | — | — |
| Ridge | *(from notebook output)* | — | — |
| Decision Tree | *(from notebook output)* | — | — |
| Random Forest | *(from notebook output)* | — | — |
| **LightGBM** | **Best** | **Lowest** | **Lowest** |

*Full numeric results are available in the notebook output cells.*

**Key Observations:**
- Linear models performed weakly, confirming the non-linear nature of the audience data
- Tree-based models significantly outperformed linear baselines
- LightGBM achieved the best R², lowest RMSE, and lowest MAE
- Post-processing (calibration + bias correction + blending) further stabilized predictions

---

## 🛠️ Tech Stack

| Category | Libraries |
|----------|-----------|
| Data Manipulation | `pandas`, `numpy` |
| Visualization | `matplotlib`, `seaborn` |
| Machine Learning | `scikit-learn`, `lightgbm` |
| Time Series | `statsmodels` (ARIMA, ADF test) |
| Pipeline | `sklearn.pipeline`, `ColumnTransformer` |
| Hyperparameter Tuning | `GridSearchCV`, `RandomizedSearchCV`, `TimeSeriesSplit` |

---

## 🚀 Setup & Usage

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/cinema-audience-forecasting.git
cd cinema-audience-forecasting
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Add the data

Download the dataset from the [Kaggle competition page](https://www.kaggle.com/) and place the CSV files under `data/raw/`.

### 4. Run the notebook

```bash
jupyter notebook notebooks/cinema_audience_forecasting.ipynb
```

---

## 🔮 Future Improvements

- **Stacking / Ensembling** — Combine LightGBM with XGBoost or CatBoost for a blended prediction
- **Target encoding** — Replace LabelEncoder on `theater_area` with target-mean encoding to capture value-level signal
- **Audience-based lag features** — Add lag_1/roll_7 on `audience_count` directly (not just tickets_booked) for stronger autoregressive signal
- **External features** — Add public holiday flags, local event data, or weather to capture unusual spikes
- **ARIMA/Prophet baseline** — Use per-theater time-series models as an additional blend component
- **Log-transform target** — Apply `log1p` on `audience_count` to handle right-skew and reduce sensitivity to spikes
- **Optuna hyperparameter tuning** — Replace RandomizedSearchCV with Bayesian optimization for LightGBM
- **MLflow tracking** — Log experiments, parameters, and metrics for reproducibility

---

## 📄 License

This project is licensed under the [MIT License](LICENSE).

---

*Built as part of the Cinema Audience Forecasting Challenge — Kaggle Competition.*
