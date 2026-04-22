# 📊 FY26 Q2 Correlation-Adjusted Demand Forecast
### Cisco Product Portfolio · 30 Products · Time Series Forecasting with Correlation Clustering

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)](https://www.python.org/)
[![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?style=flat&logo=jupyter&logoColor=white)](https://jupyter.org/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Competition%20Submission-gold?style=flat)]()

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Key Features](#-key-features)
- [Repository Structure](#-repository-structure)
- [Methodology](#-methodology)
  - [Step 1 — Data Ingestion & Validation](#step-1--data-ingestion--validation)
  - [Step 2 — Big Deal Adjustment](#step-2--big-deal-adjustment)
  - [Step 3 — Exploratory Data Analysis](#step-3--exploratory-data-analysis)
  - [Step 4 — Correlation Analysis](#step-4--correlation-analysis)
  - [Step 5 — Model Building & Backtesting](#step-5--model-building--backtesting)
  - [Step 6 — Base Forecast](#step-6--base-forecast)
  - [Step 7 — Correlation Adjustment](#step-7--correlation-adjustment)
  - [Step 8 — Final Forecast & Export](#step-8--final-forecast--export)
- [Forecasting Models](#-forecasting-models)
- [Correlation Analysis](#-correlation-analysis)
- [Data Sources](#-data-sources)
- [Results Summary](#-results-summary)
- [Installation & Setup](#-installation--setup)
- [Usage](#-usage)
- [Output Files](#-output-files)
- [Key Decisions Log](#-key-decisions-log)
- [External Factors Considered](#-external-factors-considered)
- [Accuracy Metrics](#-accuracy-metrics)
- [Contributing](#-contributing)

---

## 🔭 Project Overview

This project implements a **production-grade, 8-step demand forecasting pipeline** for FY26 Q2 (February–April 2026) across 30 Cisco networking and security products. It was built as a competitive forecasting submission against Demand Planning, Marketing, and Data Science teams.

The pipeline goes beyond standard time-series extrapolation by introducing a **correlation adjustment layer** — products whose Avg Deal (regular recurring demand) series are structurally correlated are grouped into demand clusters, and cluster-level momentum is used to cross-validate and nudge individual product forecasts.

### What Makes This Different

| Standard Approach | This Model |
|-------------------|-----------|
| Forecast raw actuals | Forecast **Avg Deal series** (Big Deals stripped out and overlaid separately) |
| One model for all products | **4 models backtested**, best selected per product based on lifecycle + CV |
| Point estimate only | **Confidence range ± MAPE%** per product |
| No cross-product signal | **Avg Deal correlation clustering** — 116 strong pairs, 15 demand clusters identified |
| Submit and wait | **Live accuracy tracker** — paste actuals when FY26 Q2 lands |

---

## ✨ Key Features

- **Big Deal Adjustment** — Separates irregular large orders from recurring demand before modelling
- **4 Forecasting Models** — WMA, SES, Holt's Linear, Linear Regression — all built from first principles
- **Backtesting Framework** — Train on Q1–Q11, validate on Q12 (FY26 Q1 holdout) before forecasting
- **Dual Correlation Analysis** — Pearson + Spearman matrices for both Big Deal and Avg Deal series
- **Union-Find Clustering** — Identifies demand clusters from very strong (r ≥ 0.85) Avg Deal correlations
- **Momentum-Based Adjustment** — Cluster momentum applied with 30% dampening factor (capped at ±15%)
- **Negative Correlation Detection** — Budget-competing product pairs flagged for business review
- **Confidence Tiers** — HIGH / MEDIUM / LOW tier per product based on MAPE, BD%, and lifecycle
- **4-Sheet Excel Output** — Final forecast, model comparison, correlation summary, live accuracy tracker
- **PowerPoint Presentation** — 15-slide judge-ready deck explaining the full methodology

---

## 🗂 Repository Structure

```
Cisco-Forecast_League---CFL/
│
├── FY26_Q2_Correlation_Forecast.ipynb          # Main forecasting notebook (8 steps)
│  
├── CFL_External_Data_Pack_Phase1_JN.xlsx       # Source data (not included — see Data Sources)
│
├──  CFL_External_Data_Pack_Phase1_JN_Modified.xlsx  # Source data modified on various parameters
│
├── FY26_Q2_CorrelationAdjusted_Forecast.xlsx   # Final forecast Excel (4 sheets)
│   
├── FY26_Q2_Model_Presentation.pptx  # 15-slide judge presentation
│   
│
├── README.md
├── requirements.txt
└── LICENSE
```

---

## 🔬 Methodology

### Step 1 — Data Ingestion & Validation

All four Excel sheets are parsed defensively:
- No assumed headers — raw sheet read with `header=None`
- Explicit column mapping with forced numeric casting
- **Spot-check validation** before proceeding: known values from the source file are asserted programmatically

The Big Deal sheet uses **dynamic column detection** (scans Row 0 for section labels) rather than hardcoded column indices — robust to extra columns added by different Excel versions.

```python
# Robust column detection
tc = next(i for i,v in enumerate(row0) if str(v)=='MFG Book Units')
bc = next(i for i,v in enumerate(row0) if str(v)=='Big Deals')
ac = next(i for i,v in enumerate(row0) if str(v)=='Avg Deals')
```

---

### Step 2 — Big Deal Adjustment

A Big Deal is a single large customer order that inflates bookings in the quarter it lands. Training a model on data containing these spikes causes systematic over-forecasting.

**Formula:**
```
Avg Deal Units = Raw Bookings − Big Deal Units
```

**Big Deal exposure tiers across 30 products:**

| Tier | Threshold | Products | Approach |
|------|-----------|----------|----------|
| 🔴 HIGH | BD% > 15% | 12 | Must use Avg Deal series |
| 🟡 MEDIUM | BD% 5–15% | 10 | Adjust before modelling |
| 🟢 LOW | BD% < 5% | 8 | Raw ≈ Adjusted |

Notable examples:
- `ROUTER Branch 8-Port PoE (#30)` — **40.5%** of sales were Big Deals
- `ROUTER Core Modular Chassis (#6)` — **45.9%** (also NPI-Ramp = double uncertainty)
- `WiFi6E Indoor (#2)` — Single quarter spike of **31,112 Big Deal units** in FY25 Q3

Big Deals are then **overlaid back** as a separate estimate:
```
Final Forecast = Base Forecast (Avg Deal) + Big Deal Overlay
Big Deal Overlay = Historical Frequency × Average Deal Size
```

---

### Step 3 — Exploratory Data Analysis

Three diagnostics are computed per product before any model is assigned:

| Diagnostic | Formula | Interpretation |
|------------|---------|----------------|
| **CV (Coefficient of Variation)** | σ ÷ μ | < 0.15 stable, 0.15–0.30 moderate, > 0.30 volatile |
| **Trend Slope** | OLS β coefficient | Units gained/lost per quarter |
| **Seasonality Ratio** | Peak quarter avg ÷ Trough quarter avg | > 1.30 = meaningful seasonal pattern |

Product lifecycle distribution:
- **23 Sustaining** — stable recurring demand
- **6 Decline** — structural downward trend
- **1 NPI-Ramp** — new product, limited history

---

### Step 4 — Correlation Analysis

Two separate correlation analyses are run:

#### Big Deal Correlation (41 strong pairs)
Captures co-movement in large order volumes — same customers buying product bundles simultaneously.

Top finding: `Router Enterprise Edge (#10) ↔ Firewall NG_1 (#15)` — **r = +0.988**, nearly perfect correlation. Sold as a security + routing bundle.

#### Avg Deal Correlation (116 strong pairs)
Captures structural recurring demand linkages — far more informative for forecasting than Big Deal correlations.

```
Metrics: Pearson r + Spearman ρ
Threshold: |r| ≥ 0.60, p < 0.10
Very strong positive: r ≥ 0.85 (15 pairs)
Strong negative: r ≤ -0.70 (8 pairs — budget competition)
```

**Demand clusters** are identified using a **Union-Find algorithm** on very strong positive Avg Deal pairs (r ≥ 0.85):

| Cluster | Members | Min r | Business Interpretation |
|---------|---------|-------|------------------------|
| Campus Core Bundle | #3, #8, #14, #24 | 0.877 | Campus infrastructure co-procurement |
| Enterprise Edge Switch Family | #16, #17, #29 | 0.926 | Enterprise refresh cycles |
| Firewall + Router Security Stack | #10, #15, #21 | 0.844 | Security infrastructure bundle |

Key negative correlation: `#17 SW 24P PoE+ Compact ↔ #22 IP Phone Enterprise Desk` — **r = −0.894**. When compact switches rise, desk phones fall. Network modernisation replacing desk phones.

---

### Step 5 — Model Building & Backtesting

All four models are implemented from first principles with optimised parameters.

**Backtest design:**
```
Training data:  Quarters 1–11 (FY23 Q2 → FY25 Q4)
Holdout:        Quarter 12 (FY26 Q1) — actual available for validation
```

**Model selection rule:**
1. Rule-based assignment via lifecycle + CV
2. Backtest all 4 models on holdout quarter
3. If best backtest model beats rule-model by > 5 pp MAPE → override
4. Refit selected model on all 12 quarters → forecast Q13

> **No ensemble blending.** With 8–12 quarters per product, single backtested model selection is more transparent and defensible than ensemble complexity.

---

### Step 6 — Base Forecast

Best model per product is retrained on **all 12 quarters** of the Avg Deal adjusted series and forecasts FY26 Q2 one step ahead.

Forecast formula for Holt's Linear (most common assigned model):
```
Level:    L_t = α·y_t + (1−α)(L_{t-1} + T_{t-1})
Trend:    T_t = β·(L_t − L_{t-1}) + (1−β)·T_{t-1}
Forecast: F_{T+1} = L_T + T_T
```
Parameters α and β are jointly optimised via grid search to minimise SSE on each product's history.

---

### Step 7 — Correlation Adjustment

The correlation adjustment uses cluster-level **demand momentum** to cross-validate each product's base forecast.

**Algorithm:**
```python
# 1. Product momentum
momentum = (mean of last 2 quarters) / (mean of prior 2 quarters) - 1

# 2. Cluster momentum
cluster_momentum = mean(momentum of all cluster members)

# 3. Divergence check
divergence = cluster_momentum - own_momentum

# 4. Apply adjustment if divergence exceeds threshold
if abs(divergence) > 0.05:
    adj_factor = clip(0.30 * divergence, -0.15, +0.15)
    corr_adj_forecast = base_forecast * (1 + adj_factor)
```

**Design choices:**
- **30% dampening** — applies only 30% of cluster signal; conservative on sparse data
- **±15% cap** — prevents overcorrection from a single cluster outlier
- **5% threshold** — only adjusts when cluster and product meaningfully disagree

**Negative correlations** are flagged in a separate table for business review — not automatically adjusted (insufficient data for a quantitative budget substitution model).

---

### Step 8 — Final Forecast & Export

```
Final Forecast = Corr-Adj Forecast + Big Deal Overlay
Confidence Range = Final ± max(MAPE, 8%)
```

Output: **4-sheet Excel workbook**
1. `Final Forecast` — Full pipeline output per product with confidence bounds
2. `Model Comparison` — All 4 models + 3 team forecasts side by side
3. `Correlation Summary` — Top 30 Avg Deal pairs with interpretation
4. `Accuracy Tracker` — Paste FY26 Q2 actuals in yellow column → all metrics auto-calculate

---

## 🤖 Forecasting Models

### Weighted Moving Average (WMA)
```python
def wma(series, n=4):
    weights = np.arange(1, n+1, dtype=float)  # [1, 2, 3, 4]
    return np.dot(weights, series[-n:]) / weights.sum()
```
**Best for:** CV > 0.30 · No clear trend · Volatile products

### Simple Exponential Smoothing (SES)
```python
# S_t = α·y_t + (1−α)·S_{t-1}
# α optimised via grid search over [0.05, 0.95]
```
**Best for:** NPI-Ramp · Short history (< 6 quarters) · No trend

### Holt's Linear Trend
```python
# Level:  L_t = α·y_t + (1−α)(L_{t-1} + T_{t-1})
# Trend:  T_t = β(L_t − L_{t-1}) + (1−β)T_{t-1}
# Forecast: L_T + h·T_T
# α, β jointly optimised via grid search
```
**Best for:** Sustaining lifecycle · Moderate volatility · Trend present

### Linear Regression (OLS)
```python
# y = a + b·t  →  Forecast at t = T+1
# Floor at 0 (sales cannot be negative)
```
**Best for:** Decline lifecycle · Strong negative slope

---

## 📈 Correlation Analysis

### Methodology
```python
from scipy.stats import pearsonr, spearmanr

# Both Pearson and Spearman computed per pair
r, p    = pearsonr(series_i, series_j)
sr, _   = spearmanr(series_i, series_j)

# Strong pair criteria
abs(r) >= 0.60 and p <= 0.10
```

### Why Both Big Deal AND Avg Deal?

| Series | Pairs Found | What It Reveals |
|--------|-------------|-----------------|
| Big Deal | 41 | Customer bundling behaviour (same order) |
| Avg Deal | 116 | Structural market demand linkages |

Avg Deal correlations are **3× more numerous** and structurally more stable — they represent genuine demand co-movement rather than coincidental order timing.

### Cluster Detection Algorithm
```python
# Union-Find (Disjoint Set Union)
# Merges products with r >= 0.85 positive Avg Deal correlation
# into connected components

parent = {rank: rank for rank in all_ranks}

def find(x):
    while parent[x] != x:
        parent[x] = parent[parent[x]]  # path compression
        x = parent[x]
    return x

def union(a, b):
    parent[find(a)] = find(b)

for rank_i, rank_j, r in very_strong_pairs:
    union(rank_i, rank_j)
```

---

## 📂 Data Sources

The model requires one Excel file with the following sheets:

| Sheet | Contents | Key Columns |
|-------|----------|-------------|
| `Data Pack - Actual Bookings` | 12 quarters of raw bookings per product | Product Name, Life Cycle, FY23Q2–FY26Q1 |
| `Big Deal` | Big Deal vs Avg Deal split per product (8 quarters) | MFG Book Units, Big Deals, Avg Deals |
| `SCMS` | Sales by customer segment (6 categories, 13 quarters) | Sales Coverage Code, quarterly units |
| `VMS` | Sales by industry vertical (15 categories, 13 quarters) | Vms Top Name, quarterly units |

**Note:** The source data file is not included in this repository as it contains proprietary Cisco sales data. Place your data file in the `data/` directory and update `DATA_FILE` in the notebook constants block.

### Cisco Fiscal Calendar
```
Q1: August – October
Q2: November – January   ← FY26 Q2 = Nov 2025 – Jan 2026
Q3: February – April     ← Our forecast TARGET
Q4: May – July
```
> Note: The Big Deal sheet uses calendar quarter labels (e.g. `2025Q4`) which map to Cisco FY labels (e.g. `FY25 Q4`).

---

## 📊 Results Summary

| Metric | Value |
|--------|-------|
| Products forecasted | 30 |
| Quarters of history | 12 (FY23 Q2 → FY26 Q1) |
| Models evaluated per product | 4 |
| Avg backtest MAPE (holdout Q12) | ~10–14% across portfolio |
| Products with MAPE < 10% | ~14 / 30 |
| Avg Deal strong correlations found | 116 |
| Demand clusters identified | 3 multi-product clusters |
| Products correlation-adjusted | ~12 / 30 |
| Big Deal products (HIGH tier) | 12 |
| Confidence tiers assigned | HIGH / MEDIUM / LOW |

---

## ⚙️ Installation & Setup

### Prerequisites

- Python 3.11+
- Jupyter Notebook or JupyterLab
- The source Excel data file

### Install Dependencies

```bash
# Clone the repository
git clone https://github.com/your-username/cisco-demand-forecast.git
cd cisco-demand-forecast

# Create a virtual environment (recommended)
python -m venv venv
source venv/bin/activate        # macOS/Linux
venv\Scripts\activate           # Windows

# Install required packages
pip install -r requirements.txt
```

### requirements.txt

```
pandas>=2.0.0
numpy>=1.24.0
scipy>=1.10.0
scikit-learn>=1.3.0
matplotlib>=3.7.0
openpyxl>=3.1.0
jupyter>=1.0.0
jupyterlab>=4.0.0
```

---

## 🚀 Usage

### 1. Place Data File

```
data/CFL_External_Data_Pack_Phase1_JN.xlsx
```

Or update the constant in the notebook:
```python
DATA_FILE = 'path/to/your/data.xlsx'
```

### 2. Run the Main Notebook

```bash
jupyter notebook notebooks/FY26_Q2_Correlation_Forecast.ipynb
```

Run all cells **top to bottom**. Each step depends on the previous one. The notebook produces:
- Console output with full forecast tables at each step
- 4 saved chart files in `charts/`
- Final Excel output `FY26_Q2_CorrelationAdjusted_Forecast.xlsx`

### 3. Run Stand-Out Strategies (Optional)

```bash
jupyter notebook notebooks/Stand_Out_Strategies.ipynb
```

Adds seasonal index adjustment, confidence tiers, and live accuracy tracker on top of the base model.

### 4. Update Big Deal Overlay with Pipeline Data

In Step 7 of the main notebook, replace the probability-weighted Big Deal estimate with confirmed pipeline data:

```python
# Default (probability-weighted)
bd_overlay = round(bd_freq * bd_avg)

# Replace with confirmed pipeline data
# bd_overlay = confirmed_pipeline_units_for_this_product
```

---

## 📁 Output Files

### `FY26_Q2_CorrelationAdjusted_Forecast.xlsx`

| Sheet | Contents |
|-------|----------|
| `Final Forecast` | Base FC · Corr Adj % · Corr Adj FC · BD Overlay · **Final Forecast** · Lower/Upper bounds · Adj Reason |
| `Model Comparison` | WMA · SES · Holt · LR · **Our Final** · Demand Planner · Marketing · Data Science |
| `Correlation Summary` | Top 30 Avg Deal pairs · Pearson r · Spearman r · Interpretation |
| `Accuracy Tracker` | Yellow input column — paste FY26 Q2 actuals → Accuracy, Bias, Error auto-calculate |

### `BD_Adjusted_Series.xlsx`

| Sheet | Contents |
|-------|----------|
| `BD Impact Summary` | Big Deal % and tier for all 30 products |
| `Raw vs Adjusted Series` | Side-by-side raw and Avg Deal series for all 12 quarters |
| `Big Deal Detail` | Full Total / BigDeal / AvgDeal breakdown per quarter |

---

## 📝 Key Decisions Log

Every modelling decision is documented with its rationale. A forecast without an audit trail is not a professional forecast.

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Remove Big Deals before modelling | ✅ Applied | 12/30 products >15% BD exposure — raw actuals systematically over-estimate baseline demand |
| Model per product (not portfolio-wide) | ✅ Applied | Sustaining, Decline, NPI-Ramp behave fundamentally differently |
| Ensemble blending | ❌ Not used | With 8–12 quarters, single backtested model is more defensible than ensemble complexity |
| Backtest on holdout Q12 | ✅ Applied | Only 12 quarters available — one holdout (FY26 Q1) is the practical maximum |
| Correlation adjustment dampening | 30% factor | Conservative — prevents overcorrection on sparse data |
| Correlation adjustment cap | ±15% | Avoids extreme adjustments from a single cluster outlier |
| Negative correlations | Flag only | Insufficient data for a quantitative budget substitution model |
| Confidence range floor | 8% minimum | Prevents overconfidence even for low-MAPE products |
| BD Overlay method | Freq × Avg Size | Default when CRM pipeline data unavailable; replace for production |

---

## 🌍 External Factors Considered

Beyond the quantitative model, these macro and industry factors were considered and documented for business stakeholder review:

**Technology Transitions**
- **WiFi 7 adoption** — Potential demand regime change for WiFi6/6E products (#2, #5); step-change decline risk not captured by linear trend
- **IP Phone secular decline** — Teams/Webex replacing desk phones; logarithmic decay may better fit than linear regression for #22, #25
- **AI infrastructure buildout** — Demand tailwind for Data Center switch products (#18, #20, #21) from GPU cluster networking

**Macro-Economic**
- **CapEx cycle** — FY26 Q2 falls in a rate-easing environment; historically unlocks deferred infrastructure spend
- **IT spend elasticity** — Enterprise hardware tracks GDP at ~1.3× — cautiously positive for FY26
- **Currency risk** — Strong USD suppresses non-USD demand in Service Provider and Government verticals

**Industry Factors**
- **Enterprise network refresh** — COVID-era networks (2020–2022) approaching 5–7 year natural refresh window
- **Public sector budget calendar** — State/Local FY year-end (June 30) creates Q4 surge; FY26 Q2 is pre-rush buildup
- **Cisco product refresh cycle** — 18–24 month cadence; any announced Next-Gen Catalyst platform would cause demand pause
- **Supply chain lead times** — Short lead times (8–12 weeks post-2023) mean bookings reflect near-term demand accurately

---

## 📏 Accuracy Metrics

Accuracy is measured using the formulas defined in the Cisco Forecasting Glossary:

```
Accuracy = 1 − |Forecast − Actual| / Actual

Bias = (Forecast − Actual) / Actual
  Positive bias = Over-forecast
  Negative bias = Under-forecast

MAPE = Mean Absolute Percentage Error
     = mean(|Forecast_i − Actual_i| / Actual_i × 100)
```

The `Accuracy Tracker` sheet in the output Excel file auto-calculates all three metrics when actuals are pasted into the yellow input column.

---

## 🤝 Contributing

This project was developed as a demand forecasting competition submission. If you are adapting it for your own use:

1. Fork the repository
2. Update `DATA_FILE` and `ACTUAL_QUARTERS` in the constants block to match your data
3. Review the Big Deal sheet column detection — it is designed to be robust but verify for your specific file format
4. The Big Deal overlay currently uses historical frequency × average size. Replace with CRM pipeline data for production use
5. The correlation adjustment dampening factor (0.30) and threshold (0.05) are conservative defaults — tune these based on your portfolio's data density

---

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgements

- **Data Source:** Cisco Forecasting Lab (CFL) — External Data Pack Phase 1
- **Forecasting references:** Holt (1957), Gardner & McKenzie (1985) for exponential smoothing; Box & Jenkins for time series modelling principles
- **Correlation clustering:** Tarjan (1975) Union-Find data structure for connected component detection

---

<div align="center">

**Built for the Cisco FY26 Q2 Demand Forecasting Competition**

*"The teams that win competitions show they're right once. The teams that get hired show they've built a process that can be trusted every quarter."*

</div>
