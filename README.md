# 7Learnings Data Scientist Case Study – Retail Demand Forecasting

Daily product-level forecast of `sales_before_returns` for market `DE`, channel `myshop.de`,
for **7–20 September 2026** (3,920 products × 14 days), using only information available on
6 September 2026, plus an investigation of the planner's question about the
*Focus article / Tops* cohort.

## Contents

| File | Description |
|---|---|
| `Coding Challenge.ipynb` | Full solution, executed with outputs: data analysis, features, baseline vs model, investigation SQL, memo |
| `predictions.csv` | Required output: `product_id, date, predicted_sales` for the exact pairs in `sample_prediction.csv` |
| `requirements.txt` | Same dependencies as the original challenge (no extra packages) |

## How to run

1. Download and decompress the database next to the notebook (see the challenge README):
   `7ldata.duckdb` (not included in this repository because of its size).
2. Create an environment and install the dependencies:
   ```bash
   python -m venv venv
   source venv/bin/activate          # Windows: venv\Scripts\Activate.ps1
   pip install -r requirements.txt
   ```
3. Open `Coding Challenge.ipynb` and run all cells top to bottom (about 10 minutes).
   The last section writes `predictions.csv` and checks row count, missing values and negatives.

## Approach

1. **Data analysis (SQL / DuckDB).** Found and handled placeholder dates (1970/2099),
   placeholder quantities (10,000 / 99,999 – about 5.3M units), negative quantities, missing and
   test product ids, 27k exact duplicate rows, and ids differing only in case or spaces
   (recovered with `lower(trim())`). 801 products had a stock-out from 25 Nov 2025 to 23 May 2026;
   those days are treated as missing demand, not zero demand. Result: a clean product × day table
   with explicit zeros.
2. **Features.** Seven features known at the forecast origin: recent product averages
   (7/28/91 days), same-weekday average of the last four known weeks, same weekday one year
   earlier, weekday and horizon. Traffic (a mirror of sales, unknown in the future), stock and the
   promotion plan (no history, no variation across products) are analysed but not used as features.
3. **Model and validation.** Direct multi-horizon strategy: one row per product, origin and
   horizon, with all features computed up to the origin, so no prediction is fed back and nothing
   inside the forecast window is used. LightGBM (L1 objective) trained on 76 weekly origins.
   Validation on 24 Aug – 6 Sep 2026 (origin 23 Aug) plus two earlier 14-day windows.
4. **Investigation.** Executable SQL verifying the cohort drop (−6.5%), testing three explanations
   (shop-wide/seasonal movement, fewer strong-voucher days, availability/data quality) and a
   historical test of whether two-week drops persist.

## Key results

| Validation (24 Aug – 6 Sep) | WAPE | MAE | RMSE | Bias |
|---|---|---|---|---|
| Seasonal naive baseline | 0.218 | 0.864 | 1.482 | +0.099 |
| LightGBM | **0.201** | **0.796** | **1.326** | **+0.089** |

- LightGBM beats the baseline on the official window and on average over three windows
  (WAPE 0.138 vs 0.144).
- The cohort drop is real and not caused by stock. Historically, group-level two-week drops
  tend to persist (closest outcome in 83% of 40 cohort cases and 74% of 3,273 category cases),
  so the memo recommends planning the cohort at the recent level (~1,100 units) instead of the
  model's ~1,176. `predictions.csv` keeps the model's product-level forecasts.

## Technical choices and trade-offs

- **SQL for heavy aggregation, pandas/numpy for features and modelling.** DuckDB aggregates the
  2.7M-row tables quickly without loading them into memory.
- **Small, transparent feature set** over a larger one: extra features (catalog attributes,
  shop-level averages, weekday profiles) were tested and made validation worse.
- **Weekly origins** (Sundays) to match the real forecast origin and avoid near-duplicate rows.
- **Group adjustment kept out of the CSV**: the persistence evidence is at group level, and a
  uniform cut would penalise products that did not drop.

## Unfinished work and next steps

Disclosed in the *Unfinished work and next steps* cell of the notebook.

## AI usage

Disclosed in the *Candidate AI Disclosure* cell of the notebook.
