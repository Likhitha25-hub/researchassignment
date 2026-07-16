# Forecasting weekly German electricity demand

This repository models and forecasts German electricity demand from Open Power System
Data. The hourly load series is aggregated to weekly and daily means and a ladder of
forecasting models is compared on a fixed two-year (104-week) test window, from simple
benchmarks through to a neural sequence model.

## Project aim

The aim is to forecast weekly German electricity demand and to compare the accuracy,
uncertainty, interpretability and complexity of a range of forecasting approaches. The
guiding questions are:

1. How well do simple benchmark rules forecast weekly demand?
2. Does a SARIMA / SARIMAX model improve on the seasonal-naive benchmark?
3. Do temperature and holiday covariates improve forecast accuracy?
4. Do feature-based or neural models justify their extra complexity?
5. Which model is most appropriate for an operational setting?

## Data

The target is German electricity load (`DE_load_actual_entsoe_transparency`) from the Open
Power System Data 60-minute file. The series is sliced from 2015 onward on the UTC index,
converted to Europe/Berlin local time, interpolated over short gaps, converted from MW to
GW, and aggregated to weekly and daily means. A Friday-anchored week is used for the
weekly mean and partial boundary weeks (fewer than 160 hours) are dropped. The hourly GW
series is retained for the neural model.

Berlin daily temperature is used as an exogenous covariate (weekly mean temperature,
heating degree days, cooling degree days). It is fetched from the Open-Meteo archive API
and cached to `berlin_daily_temperature.csv`; if the fetch fails the notebook uses the
cache, and failing that a clearly-labelled deterministic seasonal proxy, so the run never
depends on network access. Realised temperature is not known at the forecast origin, so
the temperature-driven forecast is reported as a conditional forecast. Holiday counts are
known in advance and are valid future covariates.

## How to run

1. Install dependencies: `pip install -r requirements.txt` (or run the environment cell in
   the notebook, which also works on Colab).
2. Place `time_series_60min_singleindex.csv` in this folder, or let the data-acquisition
   cell download it from Open Power System Data.
3. Open `7. Likhita.ipynb` and run all cells top to bottom.

The notebook uses relative paths only and sets a single global seed, so it reproduces
from a fresh kernel and is Colab-ready.

## Models

- **Benchmarks** — mean, naive, seasonal-naive, drift over the 104-week horizon.
- **SARIMA** — AIC grid search over the full `p in [0,6]`, `d in [0,2]`, `q in [0,6]`
  grid (147 orders) with a `(1,1,1)[52]` seasonal part. The search screens all orders in
  parallel across CPU cores with a bounded optimiser budget, then refits the single best
  order with full default settings; progress and time-remaining lines are printed
  throughout. Residual ACF, histogram and Ljung-Box diagnostics follow, then a 104-week
  forecast with a 95% prediction interval. A `FAST` flag narrows the grid for quick
  iteration only; the submission run uses the full grid (`FAST = False`, the default).
- **SARIMAX** — the chosen SARIMA order plus the temperature exogenous block (mean temp,
  HDD, CDD), all shifted before use; reported as a conditional forecast.
- **Feature-based ML** — RandomForest (primary) and HistGradientBoosting (secondary) on
  calendar terms, weekly holiday count, temperature features and shifted target lags and
  rolling means; the 104-week forecast is produced recursively.
- **LSTM** — trained on the hourly GW series with training-only scaling, a 168-hour
  look-back, an early-stopped hyperparameter sweep over units, dropout and learning rate,
  and a genuine recursive multi-step rollout aggregated to weekly means for scoring.

## Evaluation

All models are scored on the same 104-week window using MAE, RMSE, MASE and Bias. The MASE
denominator is the in-sample seasonal-naive mean absolute error (`mean |train.diff(52)|`),
so a MASE below one beats the seasonal-naive rule; a perfect forecast scores MASE zero
(asserted in the notebook). A MASE ratio against seasonal naive is reported for every
model, together with festive-week (ISO weeks 52, 1, 2) and hottest/coldest-week regime
diagnostics. This project gives the deepest treatment to prediction-interval (95% CI)
coverage.

## Data leakage

Lag, rolling and temperature features are shifted before use so only past information
enters; the LSTM scaler is fit on training hours only; the train/test split is strictly
time-ordered and never shuffled; and no model is selected on test-set performance.

## Outputs

Running the notebook creates:

```text
outputs/forecasts/all_forecasts.csv
outputs/metrics/model_comparison.csv
outputs/metrics/regime_diagnostics.csv
outputs/figures/forecast_comparison.png
```

## Files

```text
7. Likhita.ipynb                       analysis notebook (entry point)
requirements.txt                       package list
berlin_daily_temperature.csv           cached temperature (created/used at run time)
time_series_60min_singleindex.csv      local data copy (optional; downloadable)
outputs/                               forecasts, metrics and figures (created at run time)
```
