# Henry Hub Natural Gas Spot Price Forecasting
### Regime-Stratified Machine Learning with Supply-Side Feature Importance Analysis

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Research%20Paper%20In%20Progress-orange)]()

---

## Overview

This repository contains the full data pipeline, feature engineering, 
modeling, and interpretability analysis for a weekly Henry Hub natural 
gas spot price forecasting study. The project benchmarks classical machine 
learning models against published hybrid architectures, demonstrating that 
Bayesian-tuned gradient boosting with engineered supply-side and momentum 
features achieves competitive or superior predictive accuracy — without 
the computational overhead and opacity of CNN-LSTM hybrid designs.

**Key result:** A volatile-window-tuned XGBoost model achieves R²=0.94 
on the stable evaluation window (2015–2019) and R²=0.94 on the volatile 
evaluation window (2019–2024), surpassing the Du et al. (2025) hybrid 
benchmark of R²=0.90 on weekly granularity data.

---

## Research Questions

1. Can Bayesian-optimized classical ML models outperform published hybrid 
   architectures (CNN-LSTM, CDO-optimized SVR) on weekly natural gas price 
   forecasting without specialized hardware?

2. Which supply-side and momentum features drive predictive power across 
   structurally distinct market regimes (stable vs. volatile)?

3. Does model complexity — via stacking ensembles — improve upon 
   well-tuned individual models, or does refinement outperform 
   architectural complexity?

---

---

## Data Sources

| Feature Group | Source | Frequency | Period |
|---|---|---|---|
| Henry Hub Spot Price | FRED / EIA | Weekly | 2002–2024 |
| Natural Gas Storage Level | EIA | Weekly | 2002–2024 |
| Storage vs. 5-Year Average | EIA (derived) | Weekly | 2002–2024 |
| Heating/Cooling Degree Days | NOAA CPC | Weekly | 2002–2024 |
| WTI Crude Oil Price | FRED | Weekly | 2002–2024 |
| Rig Count | Baker Hughes | Weekly | 2002–2024 |

> **Note:** Raw data is not included in this repository due to licensing 
> constraints. See `data/README.md` for instructions on accessing and 
> downloading each source.

---

## Feature Engineering

Beyond raw data, the following derived features are constructed:

**Momentum and Velocity Derivatives**
- Rate of change over 4-week and 8-week windows for storage, rig count, 
  WTI price, and Henry Hub price
- Acceleration terms: second-order rate of change capturing whether trends 
  are speeding up or decelerating

**Rolling Window Statistics**
- 4-week, 8-week, and 26-week rolling mean, standard deviation, min, and 
  max for price and storage series
- Rolling 8-week price volatility
- 52-week storage z-score capturing deviation from seasonal norms

**Regime Definition**
- Stable regime: 2015–2019 (shale-era low-volatility, mean CV < 0.15)
- Volatile regime: 2019–2024 (COVID disruption, February 2021 polar vortex 
  z=7.58, post-Ukraine price surge)

---

## Modeling Pipeline

Raw Data

↓

Feature Engineering (momentum, rolling stats, z-scores)

↓

Feature Selection (LASSO + RF consensus on autocorrelation-residualized target)

↓

Log Transform + RobustScaler (fit on training window only)

↓

┌─────────────────────────────────────────────────────────┐
│ 
Ridge Regression (baseline — no lagged price features) │
│  Random Forest (500–691 trees, Bayesian-tuned)          │
│  XGBoost (volatile-targeted Bayesian optimization)      │
│  Enhanced Stack (RF + XGB + LGBM + CatBoost → RidgeCV) │
└─────────────────────────────────────────────────────────┘

↓

Regime-Stratified Evaluation

↓

SHAP Interpretability Analysis

↓

Diebold-Mariano Significance Testing

---

## Results Summary

| Model | Stable R² | Volatile R² | MAE (Volatile) |
|---|---|---|---|
| Ridge (baseline, no lags) | -5.80 | -0.04 | 1.71 |
| Random Forest (V2) | 0.943 | 0.879 | 0.299 |
| XGBoost (volatile-tuned, V2) | **0.964** | **0.939** | **0.206** |
| Enhanced Stack (4→RidgeCV) | 0.824 | 0.894 | 0.351 |
| Du et al. FS-CDO-CNN-LSTM&SVR† | — | 0.902 | 0.358 |

† Du et al. (2025) results on monthly data for reference.  
All results on weekly data unless otherwise noted.

**Diebold-Mariano significance:** XGBoost significantly outperforms Ridge 
(p<0.001) and Random Forest (p=0.019) on the stable window. Performance 
differences on the volatile window do not reach significance between 
ensemble models, consistent with correlated error structures during 
extreme market events.

---

## Interpretability

SHAP analysis on the trimmed XGBoost model (zero-contribution features 
removed) identifies the following hierarchy of predictive importance:

**Both regimes:** `lag_4` > `price_roc_4w` > `price_vol_8w`

**Supply-side contributors:** `storage_vs_5yr`, `storage_zscore`, 
`rig_roc_4w` — all with confirmed nonzero SHAP values

**Key finding:** Removing all price-level rolling features reduces volatile 
window R² from 0.939 to 0.854, confirming that supply-side and momentum 
features provide meaningful signal beyond price autocorrelation.

---

## Computational Efficiency

All models trained on standard CPU hardware without GPU acceleration.  
Full pipeline runtime: < 30 minutes including Bayesian hyperparameter 
optimization (400 trials per model).

Contrast with Du et al.'s FS-CDO-CNN-LSTM&SVR architecture which requires 
iterative population-based CDO optimization combined with GPU-accelerated 
CNN-LSTM backpropagation — implicitly GPU-hours scale with GB-level memory 
requirements.

---

## Installation

```bash
git clone https://github.com/yourusername/henry-hub-ng-forecasting.git
cd henry-hub-ng-forecasting
pip install -r requirements.txt
```

**Core dependencies:**

pandas>=2.0
numpy>=1.24
scikit-learn>=1.3
xgboost>=2.0
lightgbm>=4.0
catboost>=1.2
optuna>=3.0
shap>=0.44
statsmodels>=0.14
matplotlib>=3.7
seaborn>=0.12

---

## Citation

If you use this code or findings in your research, please cite:

```bibtex
@article{wilson2025henryhub,
  title   = {Regime-Stratified Classical Machine Learning for Weekly 
             Henry Hub Natural Gas Spot Price Forecasting},
  author  = {Wilson, Brandon},
  year    = {2025},
  note    = {Manuscript in preparation}
}
```

---

## Reference

Du, P., Zhang, X., Du, J., & Wang, J. (2025). Multivariate natural gas 
price forecasting model with feature selection, machine learning and 
Chernobyl disaster optimizer. *Petroleum Science*, 22, 4823–4837. 
https://doi.org/10.1016/j.petsci.2025.09.034

---

## License

MIT License. See `LICENSE` for details.

---

## Contact

Brandon Wilson — Graduate Researcher, University of Maryland  
Department of Data Science and Machine Learning
