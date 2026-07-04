# Data README

## Overview

This directory contains instructions for acquiring, downloading, and 
structuring all datasets used in the Henry Hub natural gas spot price 
forecasting pipeline. Raw data files are **not included** in this 
repository due to source licensing constraints. All datasets are 
publicly available at no cost from the sources listed below.

Following these instructions exactly will reproduce the full dataset 
used in the accompanying paper (still in works). The final merged dataset spans 
**December 28, 2002 to December 28, 2024** at weekly frequency, 
yielding **1,102 observations** after preprocessing.

---

---

## Data Sources

### 1. Henry Hub Natural Gas Spot Price

| Attribute | Detail |
|---|---|
| **Variable name** | `hh_price` |
| **Unit** | Dollars per Million British Thermal Units ($/MMBtu) |
| **Frequency** | Weekly (Friday) |
| **Period** | December 28, 2002 – December 28, 2024 |
| **Source** | U.S. Energy Information Administration via FRED |
| **FRED Series ID** | `HENRY` |
| **Direct URL** | https://fred.stlouisfed.org/series/HENRY |

**Download instructions:**

1. Navigate to https://fred.stlouisfed.org/series/HENRY
2. Click **Download** → Select **CSV**
3. Set date range: `2002-12-28` to `2024-12-28`
4. Save as `data/raw/fred/henry_hub_spot_price.csv`

**Alternatively, via FRED API (Python):**
```python
import pandas as pd
import requests

api_key = 'YOUR_FRED_API_KEY'  # register free at https://fred.stlouisfed.org/docs/api/api_key.html
url = (
    f'https://api.stlouisfed.org/fred/series/observations'
    f'?series_id=HENRY'
    f'&observation_start=2002-12-28'
    f'&observation_end=2024-12-28'
    f'&api_key={api_key}'
    f'&file_type=json'
)
response = requests.get(url).json()
df = pd.DataFrame(response['observations'])[['date', 'value']]
df.columns = ['date', 'hh_price']
df['hh_price'] = pd.to_numeric(df['hh_price'], errors='coerce')
df.to_csv('data/raw/fred/henry_hub_spot_price.csv', index=False)
```

---

### 2. WTI Crude Oil Spot Price

| Attribute | Detail |
|---|---|
| **Variable name** | `wti_price` |
| **Unit** | Dollars per Barrel ($/bbl) |
| **Frequency** | Weekly (Monday) — requires forward-fill alignment to Friday |
| **Period** | December 28, 2002 – December 28, 2024 |
| **Source** | U.S. Energy Information Administration via FRED |
| **FRED Series ID** | `DCOILWTICO` |
| **Direct URL** | https://fred.stlouisfed.org/series/DCOILWTICO |

**Download instructions:**

1. Navigate to https://fred.stlouisfed.org/series/DCOILWTICO
2. Click **Download** → Select **CSV**
3. Set date range: `2002-12-28` to `2024-12-28`
4. Save as `data/raw/fred/wti_crude_price.csv`

> **Important:** WTI is reported daily. Resample to weekly frequency 
> using Friday end-of-week values before merging. Use 
> `df.resample('W-FRI').last()` in pandas.

---

### 3. Natural Gas Working Storage Level

| Attribute | Detail |
|---|---|
| **Variable names** | `storage_bcf`, `storage_chg` |
| **Unit** | Billion Cubic Feet (Bcf) |
| **Frequency** | Weekly (Thursday report, Friday release) |
| **Period** | December 28, 2002 – December 28, 2024 |
| **Source** | U.S. Energy Information Administration |
| **EIA Series** | Natural Gas Weekly Underground Storage Report |
| **Direct URL** | https://www.eia.gov/naturalgas/storage/dashboard/ |

**Download instructions:**

1. Navigate to https://www.eia.gov/naturalgas/storage/dashboard/
2. Select **Weekly Working Gas in Underground Storage**
3. Download the full historical CSV
4. Save as `data/raw/eia/ng_storage_weekly.csv`

**Derived features constructed from this source:**
- `storage_chg` = `storage_bcf(t) - storage_bcf(t-1)` (week-over-week change in Bcf)
- `storage_vs_5yr` = `storage_bcf(t) - 5yr_avg_bcf(t)` (deviation from seasonal norm)

> **Note:** The EIA reports storage on Thursdays. Values are 
> aligned to the Friday date stamp to match the Henry Hub price 
> series before merging.

---

### 4. Natural Gas Storage 5-Year Average

| Attribute | Detail |
|---|---|
| **Variable name** | `storage_vs_5yr` (derived) |
| **Unit** | Billion Cubic Feet (Bcf) |
| **Frequency** | Weekly |
| **Period** | December 28, 2002 – December 28, 2024 |
| **Source** | U.S. Energy Information Administration |
| **Direct URL** | https://www.eia.gov/naturalgas/storage/dashboard/ |

**Download instructions:**

The 5-year average storage series is available on the same EIA 
storage dashboard as the working storage level.

1. Navigate to https://www.eia.gov/naturalgas/storage/dashboard/
2. Select **5-Year Average** overlay and download as CSV
3. Save as `data/raw/eia/ng_storage_5yr_avg.csv`

> Alternatively, the 5-year average can be computed programmatically 
> from the raw storage series using a rolling 5-year (260-week) mean 
> aligned to the same calendar week.

---

### 5. Heating and Cooling Degree Days

| Attribute | Detail |
|---|---|
| **Variable names** | `hdd`, `cdd` |
| **Unit** | Degree-days (°F-days) |
| **Frequency** | Weekly |
| **Period** | December 28, 2002 – December 28, 2024 |
| **Source** | NOAA Climate Prediction Center (CPC) |
| **Direct URL** | https://www.cpc.ncep.noaa.gov/products/analysis_monitoring/cdus/degree_days/ |

**Download instructions:**

1. Navigate to https://www.cpc.ncep.noaa.gov/products/analysis_monitoring/cdus/degree_days/
2. Select **Heating Degree Days** → **Weekly** → **National**
3. Download the full historical archive
4. Repeat for **Cooling Degree Days**
5. Save as `data/raw/noaa/hdd_cdd_weekly.csv`

> **Important:** NOAA provides HDD and CDD for multiple regional 
> aggregations. This study uses the **population-weighted national 
> average** to capture aggregate demand effects. Ensure you select 
> national totals, not regional breakdowns.

**Variable definitions:**
- `hdd` = Heating Degree Days — degrees below 65°F; proxy for heating demand
- `cdd` = Cooling Degree Days — degrees above 65°F; proxy for cooling demand

---

### 6. Baker Hughes North America Rig Count

| Attribute | Detail |
|---|---|
| **Variable names** | `rig_count`, `rig_lag4` |
| **Unit** | Number of active drilling rigs |
| **Frequency** | Weekly (Friday) |
| **Period** | December 28, 2002 – December 28, 2024 |
| **Source** | Baker Hughes |
| **Direct URL** | https://bakerhughes.com/rig-count/north-america-rotary-rig-count-pivot-table |

**Download instructions:**

1. Navigate to https://bakerhughes.com/rig-count/north-america-rotary-rig-count-pivot-table
2. Download the **North America Rotary Rig Count** Excel file
3. Filter to **Natural Gas** rigs only
4. Save as `data/raw/baker_hughes/rig_count_weekly.csv`

**Derived features:**
- `rig_lag4` = `rig_count` lagged by 4 weeks, capturing the 
  production response delay between drilling activity and supply 
  impact on spot prices

> **Note:** Baker Hughes reports on Fridays, naturally aligning 
> with the Henry Hub price series. No date adjustment is required.

---

## Data Merging and Alignment

All six source datasets are merged on a common weekly Friday date 
index. The following preprocessing steps are applied before modeling:

```python
import pandas as pd
import numpy as np

# Load all sources
hh    = pd.read_csv('data/raw/fred/henry_hub_spot_price.csv',
                     parse_dates=['date'], index_col='date')
wti   = pd.read_csv('data/raw/fred/wti_crude_price.csv',
                     parse_dates=['date'], index_col='date')
stor  = pd.read_csv('data/raw/eia/ng_storage_weekly.csv',
                     parse_dates=['date'], index_col='date')
avg5  = pd.read_csv('data/raw/eia/ng_storage_5yr_avg.csv',
                     parse_dates=['date'], index_col='date')
hdd   = pd.read_csv('data/raw/noaa/hdd_cdd_weekly.csv',
                     parse_dates=['date'], index_col='date')
rigs  = pd.read_csv('data/raw/baker_hughes/rig_count_weekly.csv',
                     parse_dates=['date'], index_col='date')

# Resample WTI to weekly Friday frequency
wti = wti.resample('W-FRI').last().ffill()

# Merge on inner join — drops weeks where any source is missing
df = (hh
      .join(wti,   how='inner')
      .join(stor,  how='inner')
      .join(avg5,  how='inner')
      .join(hdd,   how='inner')
      .join(rigs,  how='inner'))

# Derived features
df['storage_chg']    = df['storage_bcf'].diff(1)
df['storage_vs_5yr'] = df['storage_bcf'] - df['storage_5yr_avg']
df['rig_lag4']       = df['rig_count'].shift(4)

# Lagged price features
df['lag_2']  = df['hh_price'].shift(2)
df['lag_4']  = df['hh_price'].shift(4)
df['lag_52'] = df['hh_price'].shift(52)

# Drop NaN rows introduced by lagging
df = df.dropna()

# Save master dataset
df.to_csv('data/processed/ng_master_dataset.csv')

print(f"Master dataset: {len(df)} observations")
print(f"Date range: {df.index.min()} to {df.index.max()}")
print(f"Columns: {df.columns.tolist()}")
```

---

## Final Dataset Schema

| Column | Type | Unit | Description |
|---|---|---|---|
| `date` | datetime | — | Week ending Friday |
| `hh_price` | float | $/MMBtu | Henry Hub natural gas spot price (**target variable**) |
| `wti_price` | float | $/bbl | WTI crude oil spot price |
| `storage_bcf` | float | Bcf | Total working natural gas in underground storage |
| `storage_chg` | float | Bcf | Week-over-week change in storage level |
| `storage_vs_5yr` | float | Bcf | Storage level minus 5-year seasonal average |
| `storage_5yr_avg` | float | Bcf | 5-year historical average storage for same calendar week |
| `hdd` | float | °F-days | National population-weighted heating degree days |
| `cdd` | float | °F-days | National population-weighted cooling degree days |
| `rig_count` | int | count | Active North America natural gas rotary rig count |
| `rig_lag4` | float | count | Rig count lagged 4 weeks |
| `lag_2` | float | $/MMBtu | Henry Hub price lagged 2 weeks |
| `lag_4` | float | $/MMBtu | Henry Hub price lagged 4 weeks |
| `lag_52` | float | $/MMBtu | Henry Hub price lagged 52 weeks |

---

## Data Quality Summary

| Variable | # Missing | Missing Rate | # IQR Outliers | # Z>3 Outliers | Max \|Z\| | Notable Event |
|---|---|---|---|---|---|---|
| HH Spot Price | 0 | 0% | 30 | 22 | 4.42 | Post-Katrina 2005 / Early 2003 cold snap |
| HDD | 0 | 0% | 0 | 0 | 2.69 | — |
| CDD | 0 | 0% | 0 | 0 | 2.66 | — |
| Storage Level | 0 | 0% | 0 | 0 | 2.56 | — |
| Storage Chg (Wk) | 0 | 0% | 7 | 4 | 7.58 | **Polar Vortex 2021 (−338 Bcf, z=7.58)** |
| Storage vs 5yr Avg | 0 | 0% | 0 | 0 | 2.60 | — |
| Rig Count | 0 | 0% | 50 | 0 | 2.93 | Pre-shale drilling boom 2008–2010 |
| Rig Count (Lag-4) | 0 | 0% | 53 | 0 | 2.92 | Pre-shale drilling boom 2008–2010 |
| WTI Price | 0 | 0% | 1 | 2 | 3.17 | 2008 commodity supercycle peak |
| Lag-2 Price | 0 | 0% | 30 | 22 | 4.42 | Inherited from HH price spikes |
| Lag-4 Price | 0 | 0% | 30 | 22 | 4.43 | Inherited from HH price spikes |
| Lag-52 Price | 0 | 0% | 31 | 22 | 4.46 | Inherited from HH price spikes |

> The February 2021 polar vortex storage withdrawal of −338 Bcf 
> (z=7.58) represents the most extreme observation in the dataset 
> and is explicitly discussed in the regime analysis section of 
> the accompanying paper.

---

## Notable Market Events in the Dataset

| Period | Event | Price Impact |
|---|---|---|
| 2003 Q1 | Severe winter cold snap | Spike to ~$9/MMBtu |
| 2005 Q3–Q4 | Hurricanes Katrina and Rita | Spike to ~$12–14/MMBtu |
| 2008–2009 | Global financial crisis + shale boom onset | Collapse from ~$12 to ~$3/MMBtu |
| 2010–2019 | Shale era suppression | Extended low-price regime ~$2–4/MMBtu |
| 2020 Q1–Q2 | COVID-19 demand destruction | Drop to ~$1.50/MMBtu |
| 2021 Q1 | February polar vortex (z=7.58) | Spike to ~$12/MMBtu |
| 2022 Q2–Q3 | Russia-Ukraine conflict / LNG export surge | Spike to ~$8–10/MMBtu |
| 2022 Q4–2024 | Post-crisis normalization | Return to ~$2–3/MMBtu |

---

## Reproducibility Checklist

Before running the modeling pipeline, verify the following:

- [ ] All six raw data files downloaded and saved to correct paths
- [ ] Date range confirmed: 2002-12-28 to 2024-12-28
- [ ] WTI resampled to weekly Friday frequency
- [ ] EIA storage dates aligned to Friday from Thursday report date
- [ ] NOAA HDD/CDD confirmed as national population-weighted totals
- [ ] Baker Hughes filtered to natural gas rigs only
- [ ] Merged dataset has 1,102 rows and zero missing values
- [ ] `ng_master_dataset.csv` saved to `data/processed/`

---

## API Keys and Registration

| Source | API Required | Registration URL | Cost |
|---|---|---|---|
| FRED | Yes (free) | https://fred.stlouisfed.org/docs/api/api_key.html | Free |
| EIA | Yes (free) | https://www.eia.gov/opendata/register.php | Free |
| NOAA | No | — | Free |
| Baker Hughes | No | — | Free (Excel download) |

---

## Questions and Issues

If you encounter data access issues or discrepancies in observation 
counts, please open a GitHub issue with the following information:

- Source name and URL attempted
- Date range requested
- Error message or row count received
- Python version and pandas version

This enables reproducibility verification and helps maintain the 
integrity of the research pipeline.
