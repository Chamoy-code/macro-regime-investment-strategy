# Macro-Regime Investment Strategy

> Academic project completed at **EMLYON Business School** as part of the R programming curriculum.

**Period covered:** January 2000, August 2025

---

## Overview

This project develops a dynamic "Smart Beta" asset allocation strategy positioned between active and passive management. It identifies the current macroeconomic regime each month and selects stocks accordingly, combining regime detection, sector rotation analysis, and machine learning-based factor ranking.

The strategy is structured in three main stages:

1. **Macro regime detection**, classify each month as Expansion, Inflation, Recovery, or Crisis using unsupervised K-means clustering on macroeconomic indicators.
2. **Sector rotation analysis**, use a Relative Rotation Graph (RRG) to identify which sectors outperform in each regime.
3. **Factor importance ranking**, use a Random Forest model to determine which fundamental factors best predict future returns in each regime, then backtest the resulting portfolio strategy.

---

## Repository Contents

| File | Description |
|---|---|
| `2526_Projet_R.Rmd` | Main R Markdown source file containing all analysis, code, and commentary |
| `2526_Projet_R.html` | Rendered HTML output with all interactive charts |
| `spx_macro.RData` | Input dataset — S&P 500 stock data combined with macroeconomic indicators |

---

## Methodology

### 1. Data Preparation

- Filters to stocks with a complete history across the full period (balanced panel).
- Computes two additional microeconomic indicators: 12-month price momentum and 1-year revenue growth.
- Computes market-wide benchmarks: average volatility and average monthly return.
- Smooths GDP and unemployment over a 3-month rolling window to reduce monthly noise before regime classification.

### 2. Macro Regime Detection (Walk-Forward K-Means)

Each month's regime is assigned using only past data — no look-ahead bias. For every month from January 2003 onward:

- Train a K-means model (4 clusters) on all data prior to that month.
- Scale the test month's indicators using only past means and standard deviations.
- Assign the month to its nearest cluster centroid.
- Label the four clusters by rule: **Crisis** = highest VIX, **Inflation** = highest inflation among the rest, **Expansion** = highest GDP growth among the rest, **Recovery** = remainder.

The first three years (2000–2002) serve as a burn-in buffer and are assigned the Expansion label.

### 3. Relative Rotation Graph (RRG)

Inspired by Julius de Kempenaer's RRG framework (2004). For each sector and regime:

- Computes the sector's market-cap weight relative to its historical average (Relative Strength).
- Computes 12-month momentum of that Relative Strength.
- Plots the average position of each sector per regime to identify leading sectors.

### 4. Factor Importance - Random Forest

A Random Forest regressor (`fwd_return ~ fundamentals`) is trained separately for each regime to measure `%IncMSE` — how much prediction error increases when each variable is permuted. This identifies the most predictive fundamental factor per regime.

### 5. Backtest

Monthly rebalancing over 2000–2025:

- Uses the previous month's regime signal to avoid look-ahead.
- Selects the top 4 sectors by average momentum for the detected regime.
- Picks the 30 best stocks from those sectors ranked by the regime-specific factor:

| Regime | Selection factor |
|---|---|
| Crisis | 12-month momentum |
| Inflation | 12-month momentum |
| Recovery | Net profit margin |
| Expansion | Market capitalisation |

- Weights each stock by `score / volatility` to penalise unstable positions.
- Compares cumulative portfolio value against an equal-weighted market benchmark.

---

## Requirements

**R packages:**

```r
library(tidyverse)
library(lubridate)
library(plotly)
library(kableExtra)
library(DT)
library(randomForest)
```

Install all at once:

```r
install.packages(c("tidyverse", "lubridate", "plotly", "kableExtra", "DT", "randomForest"))
```

**R version:** 4.0 or later recommended.

---

## Running the Project

Open `2526_Projet_R.Rmd` in RStudio and click **Knit**, or run from the console:

```r
rmarkdown::render("2526_Projet_R.Rmd")
```

The `spx_macro.RData` file must be in the same working directory as the `.Rmd` file.

The walk-forward loop runs over roughly 270 months and may take several minutes to complete depending on hardware.

---

## Key Results

- The macro regime classifier correctly identifies major historical events: the COVID-19 crisis (early 2020) and the start of the Ukraine war (2022). The 2008 subprime crisis appears late in the chronology, likely due to survivorship bias in the dataset.
- The RRG analysis shows Health Care as the leading defensive sector in Crisis regimes, Real Estate leading in Recovery and Inflation, and Professional Services / Finance leading in Expansion.
- The Random Forest highlights different dominant factors per regime.
- The backtest shows the strategy **underperforms the equal-weighted benchmark** over the full period — partly attributable to survivorship bias (the universe contains only stocks that survived the entire 25-year period) and the absence of transaction costs in the simulation.

---

## Limitations

- **Survivorship bias:** only stocks present for the entire 2000–2025 period are included. Bankrupt or delisted companies are excluded, which inflates both strategy and benchmark performance.
- **No transaction costs:** monthly rebalancing costs are not modelled.
- **Buffer label:** the 2000–2002 burn-in period is labelled Expansion even though it includes the dot-com crash.
- The backtest does not account for liquidity constraints or position size limits.
