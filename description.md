# Objective 4 — Forecast weekly orders (Regression / Time series)

**Goal:** predict the number of orders per week, 4 weeks ahead.

## Files
| File | Content |
|---|---|
| `weekly_orders.csv` | 85 weeks (2017-01-02 → 2018-08-13), columns `week_start`, `n_orders` |
| `objective4_weekly_orders_forecast.ipynb` | Cleaning checks, 15 charts, feature engineering, Linear Regression, evaluation, 4-week forecast |

## Data source
Exported from the data warehouse **TokenmafiaDW(final)**: `Fact_Orders` joined to `Dim_Date` on the purchase date (`fkdate_achat_key`), grouped by week (weeks start on Monday).
- `Fact_Orders` has one row per order and contains delivered orders only (plus 6 canceled).
- Removed: 2016 (almost no orders) and the last two weeks of August 2018 (incomplete).

## Features
`lag_1`, `rolling_mean_4` (past weeks only, via `shift(1)`), `week_of_year`, `month`, `black_friday_flag`. Target: `n_orders`.

## Results (last 12 weeks as test set)
| Model | MAE | RMSE | MAPE |
|---|---|---|---|
| Baseline (next week = last week) | 196.0 | 267.7 | 13.4% |
| Linear Regression | 222.9 | 293.9 | 14.4% |

Linear Regression does not beat the naive baseline: the test weeks (summer 2018) swing sharply and the model underpredicts on most weeks. The limit is the amount of data (~80 usable weeks, a single Black Friday), not the model choice.

## How to run (Colab)
Upload the notebook, put `weekly_orders.csv` in the Files panel, then Runtime → Run all.
