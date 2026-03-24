# Sanford Health Data Analytics Competition

## Project Summary

This project was completed for a data analytics competition hosted by Sanford Health. The goal was to forecast Emergency Department (ED) demand using historical encounter data and generate submission-ready predictions for future dates.

The final workflow builds a daily forecasting pipeline for:

- `ED Enc` (Emergency Department encounters)
- `ED Enc Admitted` (Emergency Department encounters resulting in admission)

The model produces forecasts by site and then converts those predictions into the required competition submission format using historical hourly patterns.

---

## Problem Statement

Sanford Health provided historical ED volume data across multiple sites. The objective was to predict future ED activity accurately enough to support planning and operational decision-making.

This project focused on:

- forecasting daily ED encounters
- estimating daily admitted encounters
- preserving site-level trends
- converting daily forecasts into hourly block-level submission outputs

---

## Dataset

The source dataset included fields such as:

- `Site`
- `Date`
- `Hour`
- `ED Enc`
- `ED Enc Admitted`

The raw data was aggregated to a daily level per site before modeling.

---

## Methodology

### 1. Data Preparation

The data was grouped by site and date to create a daily time series. For each site-day combination, the pipeline summed:

- `ED Enc`
- `ED Enc Admitted`

To create a complete time series, missing dates were filled for each site.

Missing values were handled as follows:

- `ED Enc` was imputed using the median by day of week
- remaining missing values were filled using the site median
- `ED Enc Admitted` was estimated using the site-level admission rate
- a `WasMissing` indicator was added to preserve missingness information

### 2. Feature Engineering

The model used a set of time-series features to capture short-term and seasonal patterns.

#### Calendar features
- day of week
- day of year
- month

#### Cyclical features
- sine and cosine transforms for day of week
- sine and cosine transforms for day of year
- sine and cosine transforms for month

#### Lag features
- 1-day lag
- 2-day lag
- 3-day lag
- 7-day lag
- 14-day lag
- 28-day lag

#### Rolling statistics
Calculated over 7, 14, and 28 day windows:
- rolling mean
- rolling standard deviation
- rolling minimum
- rolling maximum

#### Difference features
- 1-day difference
- 7-day difference

The `Site` column was label encoded, and the target variable was log-transformed using `log1p()` before training.

### 3. Train/Test Split

A time-based split was used instead of a random split to avoid leakage.

- Training set: dates before `2024-02-24`
- Test set: dates on or after `2024-02-24`

This setup better reflects real forecasting conditions by evaluating the model on future unseen data.

### 4. Model

The final model was a `HistGradientBoostingRegressor`.

This model was chosen because it works well with structured tabular data, handles nonlinear relationships, and performs strongly with engineered lag and rolling-window features.

#### Model parameters
- `learning_rate = 0.05`
- `max_depth = 6`
- `max_leaf_nodes = 31`
- `min_samples_leaf = 30`
- `l2_regularization = 0.5`
- `random_state = 42`

---

## Results

The model was evaluated on the held-out test set using standard regression metrics.

| Metric | Value |
|--------|-------|
| MAE | 1.728 |
| RMSE | 2.641 |
| R² | 0.996 |
| MAPE | 1.43% |

These results indicate that the model captured historical ED demand patterns very closely during the test period.

---

## Forecasting Workflow

After training, the model was used to recursively forecast daily ED encounters for each site for:

- `2025-09-01` through `2025-10-31`

To keep forecasts realistic, predictions were constrained using site-specific bounds derived from the historical data distribution.

---

## Submission Generation

The competition required forecasts in time blocks rather than only daily totals.

To produce the final submission format:

1. Daily site-level forecasts were generated.
2. Daily encounter totals were distributed into 6-hour blocks:
   - `00:00`
   - `06:00`
   - `12:00`
   - `18:00`
3. Historical proportions by site, day of week, and hour block were used to allocate daily totals.
4. Admitted encounters were estimated using historical admission rates at the same grouping level.
5. Fallback defaults were applied where historical combinations were missing.

Final output file:

```bash
DSU_Forecast_Submission_M1_nonLinear_TrainTestSplit.csv
