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
features achieves competitive or superior predictive accuracy, without 
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

3. Does model complexity, via stacking ensembles, improve upon 
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

| Step | Stage | Description | Justification |
|:---:|---|---|---|
| 1 | **Raw Data Ingestion** | Weekly Henry Hub spot price, EIA storage, NOAA weather, Baker Hughes rig count, and WTI crude oil collected from FRED, EIA, NOAA CPC, and Baker Hughes (2002–2024, n=1,102 observations) | Multi-source dataset captures supply-side, demand-side, and macroeconomic dimensions of natural gas price formation |
| 2 | **Outlier & Missing Data Audit** | IQR and z-score outlier flagging across all features; zero missing values confirmed across all 13 raw variables | Ensures data integrity prior to feature engineering; documents extreme events (polar vortex z=7.58) for later regime analysis |
| 3 | **Feature Engineering** | Momentum and velocity derivatives (4-week and 8-week rate of change, acceleration terms); rolling window statistics (4, 8, 26-week mean, std, min, max); 52-week storage z-score; 8-week price volatility | Replicates temporal dynamics that LSTM architectures learn implicitly, making sequential information accessible to tree-based models without recurrent architecture |
| 4 | **Feature Selection** | LASSO and Random Forest importances computed on autocorrelation-residualized target; features selected by consensus of both methods | Partialling out lag₄ autocorrelation allows supply-side features to compete fairly; consensus selection reduces noise and multicollinearity |
| 5 | **Preprocessing** | Log transformation of target variable; RobustScaler fit exclusively on training window and applied via transform to evaluation windows | Log transform stabilizes heteroscedastic variance across price regimes; RobustScaler mitigates influence of outliers on scaling parameters |
| 6 | **Regime Definition** | Stable: 2015–2019 (shale-era suppression, mean CV < 0.15); Volatile: 2019–2024 (COVID disruption, February 2021 polar vortex, post-Ukraine price surge) | Empirically defined using 52-week rolling coefficient of variation; enables regime-stratified evaluation rather than single aggregate test window |
| 7 | **Model Training** | Four specifications trained on 2003–2019 window: (i) Ridge Regression baseline; (ii) Random Forest; (iii) XGBoost with volatile-targeted Bayesian optimization (400 Optuna trials); (iv) Enhanced Stacking ensemble (RF + XGBoost + LightGBM + CatBoost → RidgeCV meta-learner via OOF predictions) | Bayesian optimization with window-targeted objective directly optimizes for regime-specific generalization; manual OOF stacking preserves temporal integrity across folds |
| 8 | **Evaluation** | Regime-stratified window evaluation on held-out stable and volatile windows; five metrics reported: MAE, RMSE, NMSE, TIC, R² | Matches Du et al. (2025) metric reporting for direct comparison; regime stratification reveals performance variation masked by aggregate test splits |
| 9 | **Statistical Significance** | Diebold-Mariano pairwise tests across all model combinations on both evaluation windows | Confirms observed performance differences are statistically distinguishable from chance — a gap in prior natural gas ML forecasting literature |
| 10 | **Interpretability** | SHAP TreeExplainer on trimmed XGBoost; dual-window bar comparison (stable vs. volatile); waterfall plot for individual prediction explanation; ablation study isolating supply-side feature contribution | Provides per-prediction and aggregate feature attribution unavailable from hybrid neural architectures; supports industrial deployment where model transparency is operationally required |

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
git clone https://github.com/BrandonWilsonProjects/henry-hub-ng-forecasting.git
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
