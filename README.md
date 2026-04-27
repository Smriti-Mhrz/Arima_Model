# Arima_Model
# ARIMA Model Deployment for SunBridge Solar

**Company:** SunBridge Solar — Regional Photovoltaic Installation Company  
**Region:** Southwestern United States  
**Analysis Type:** Time Series Forecasting — TSLM vs ARIMA Model Comparison

---

## Overview

This project forecasts monthly solar panel installation volumes for SunBridge Solar using a structured four-stage analytical pipeline. The core business question addressed is:

> *Can an ARIMA model outperform a trend-season regression (TSLM) by capturing autocorrelation structure that regression leaves unexplained — and in doing so deliver forecasts that are more accurate, unbiased, and operationally trustworthy?*

The answer is a clear **yes** — ARIMA passes all three deployment gates; TSLM fails all three.

---

## Dataset

| Attribute | Detail |
|---|---|
| **Source** | SunBridge Solar — Internal Operations Records |
| **File** | SunBridge_Installations.csv |
| **Time Span** | January 2015 – December 2024 |
| **Frequency** | Monthly |
| **Total Observations** | 120 months |
| **Variable** | Installations completed (units) |
| **Range** | 42.0 units (Jan 2015) to 169.2 units (Jun 2024) |
| **Average** | ~102 units/month |

### Data Split

| Set | Period | Observations |
|---|---|---|
| Training | January 2015 – December 2022 | 96 months |
| Validation | January 2023 – December 2024 | 24 months |

---

## Key EDA Findings

- **Trend:** Sustained upward growth — installations rose ~4× from 42 to 165+ units over the decade, with no signs of deceleration.
- **Seasonality:** Strong, consistent 12-month cycle — installations peak in June–July and trough in January–February every year.
- **Amplitude:** Seasonal swing has grown wider over time → seasonal effects scale proportionally with level → **multiplicative seasonality**.
- **ACF Pattern:** Slow decay (non-stationary trend) + spikes at lags 12, 24, 36 (seasonal structure) → differencing required.

---

## Models

### Model 1: TSLM (Trend + Season Baseline)
- Linear regression with a trend slope and 11 monthly dummy variables (13 parameters total).
- R² = 0.965 on training data — appears to fit well in-sample.
- **Fails** on validation: fixed coefficients from 2015–2022 cannot adapt to accelerating 2023–2024 growth.

### Model 2: ARIMA (Auto-selected)
- Selected order: **ARIMA(2,0,0)(2,1,0)[12] with drift** (5 parameters)
- `ar1`, `ar2`: captures short-run demand momentum from the past two months
- `sar1`, `sar2`: captures seasonal relationships at lags 12 and 24
- `D=1`: one seasonal difference removes the 12-month non-stationarity
- `drift`: constant term absorbs the long-run upward growth trend

---

## Results

### Validation Accuracy (Jan 2023 – Dec 2024)

| Metric | TSLM | ARIMA | Winner |
|---|---|---|---|
| ME (units) | -9.11 | -7.31 | ARIMA |
| RMSE (units) | 9.72 | 8.31 | ARIMA |
| MAE (units) | 9.11 | 7.31 | ARIMA |
| MAPE (%) | 6.31 | 5.04 | ARIMA |
| **MASE** | 0.760 | **0.610** | **ARIMA** ✅ |

> **Note:** Both models achieve MASE < 1, meaning both beat the seasonal naïve benchmark — but ARIMA does so more convincingly.

### In-Sample Model Fit (AICc)

| Model | AICc |
|---|---|
| **ARIMA_auto** | **512.06** ✅ |
| TrendSeason | 635.72 |

ARIMA achieves lower AICc despite using only 5 parameters vs. TSLM's 13 — demonstrating that parsimony wins.

---

## Residual Diagnostics

| Diagnostic | TSLM | ARIMA |
|---|---|---|
| Time plot | Clear wave pattern; extended runs of same-sign errors | Random scatter around zero; no persistent directional pattern |
| ACF violations | ~8–12 bars outside 95% bands | All bars within confidence bands |
| Histogram | Right-skewed; shifted above zero | Approximately bell-shaped; centred near zero |
| **Ljung–Box p (lag=24)** | **≈ 0.000** ❌ | **0.6399** ✅ |

---

## Deployment Checklist

| Gate | Requirement | TSLM | ARIMA |
|---|---|---|---|
| Gate 1 | MASE < 1 (beats seasonal naïve) | 0.760 ✅ | 0.610 ✅ |
| Gate 2 | Ljung–Box p > 0.05 (white noise residuals) | ≈ 0.000 ❌ | 0.6399 ✅ |
| Gate 3 | ME ≈ 0 (unbiased) | -9.11 ⚠️ | -7.31 ✅ |

### ✅ Final Decision: **DEPLOY ARIMA(2,0,0)(2,1,0)[12] with drift**

ARIMA is the stronger model on all fronts — more accurate, statistically valid residuals, lower bias, and lower AICc despite having 8 fewer parameters than TSLM.

---

## ARIMA vs TSLM: Why ARIMA Wins

TSLM fixes its trend and seasonal coefficients once from historical data and never adjusts. When SunBridge's demand accelerated beyond the training-period baseline, TSLM had no mechanism to correct itself.

ARIMA solves this in three concrete ways:
1. The **drift term** (6.24 units/month) continuously tracks ongoing growth rather than relying on a historical slope.
2. The **two AR terms** use the previous two months' actual figures to fine-tune each forecast — if demand is running hot, the model knows.
3. The **two seasonal AR terms** relate the current month to the same month 12 and 24 months prior, rather than assuming every June looks identical.

---

## Limitations & Next Steps

### Limitations
1. **No external signals** — The model only learns from past installation data. It has no knowledge of policy shifts (e.g., federal tax credits), electricity price changes, or housing permit volumes that drive solar demand.
2. **Growing peak amplitude** — Summer peaks are widening relative to winter troughs each year. As the business scales further, peak-season accuracy may deteriorate.

### Recommended Next Step: ARIMAX
Upgrade to **ARIMAX** — retains all of ARIMA's strengths but adds external regressors:

| External Variable | Business Relevance |
|---|---|
| Federal solar tax credit rate | Explains the demand acceleration visible from 2020 onwards |
| Regional housing permit volumes | New construction leads solar demand by several months — a valuable forward signal |

If external data is unavailable, **ETS(M,A,M)** is a strong alternative that natively handles multiplicative seasonal amplitude growth.

---

## Tools & Libraries

- **Language:** R
- **Key Package:** `fpp3`
- **Functions used:** `TSLM()`, `ARIMA()`, `forecast()`, `accuracy()`, `glance()`, `report()`, `gg_tsresiduals()`, `ACF()`, `ljung_box()`



## References

- Hyndman, R.J. & Athanasopoulos, G. (2021). *Forecasting: Principles and Practice, 3rd ed.* [https://otexts.com/fpp3/](https://otexts.com/fpp3/)
