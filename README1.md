# Air Quality Prediction using an End-to-End Data Pipeline and Explainable Machine Learning

Predicting AQI for data-scarce Kathmandu, and using SHAP to explain what drives AQI spikes (traffic, weather, seasonal burning) - not just forecasting the numbers.

## Project Overview

This project builds a full pipeline: data ingestion → cleaning → feature engineering → model training → SHAP explainability → dashboard. Data is pulled from [OpenAQ](https://openaq.org), using Kathmandu Metropolitan City's AirGradient Node-S sensor stations.

## Commit Convention

This project follows [Conventional Commits](https://www.conventionalcommits.org/):

| Prefix | Use for |
|---|---|
| `feat:` | New functionality (new script, new pipeline stage, new feature) |
| `fix:` | Bug fixes |
| `chore:` | Setup/maintenance (dependencies, folder structure, config) |
| `docs:` | README or documentation-only changes |
| `refactor:` | Restructuring code without changing behavior |
| `data:` | Dataset-related changes (schema changes, new data source) |
| `test:` | Adding/updating tests |


## Ingestion Pipeline Logic

The basic idea is:

OpenAQ
   │
   ▼
Monitoring Location
   │
   ▼
Find available sensors
   │
   ▼
Download measurements
   │
   │  (one year at a time)
   ▼
Combine all measurements
   │
   ▼
Reshape into a table
   │
   ▼
Save as CSV

## Data Sources - OpenAQ Location IDs

Stations used:

| Station | Location ID |
|---|---|
| Teku | 6093551 |
| Mid Baneshwor | 6142022 |

Provider: Green Decision Labs and Research, AirGradient sensors.

Confirmed data coverage (checked via /v3/locations/{id} metadata):

Station	First reading	Last reading
Teku	2025-10-19	2026-09-04
Mid Baneshwor	2025-11-25	2026-09-04

Backfill uses the overlapping range: 2025-11-25 to 2026-09-04.

To check a station's coverage before pulling:

bash
curl -H "X-API-Key: your_key_here" https://api.openaq.org/v3/locations/<LOCATION_ID>

Look at datetimeFirst / datetimeLast in the response.

## Project Structure.......................................................

```
air-quality-etl-ml-pipeline/
├── air_venv/              # virtual environment (not committed)
├── .env                   # API keys (not committed)
├── data/
│   ├── raw/                # raw pulls from OpenAQ + weather (not committed)
│   │   ├── teku.csv
│   │   ├── mid_baneshwor.csv
│   │   ├── teku_weather.csv
│   │   └── mid_baneshwor_weather.csv
│   └── processed/             # cleaned/merged/feature data (not committed)
│       ├── kathmandu_merged.csv
│       └── kathmandu_features.csv
├── models/                    # saved models, results JSONs, SHAP outputs
│   ├── baseline_results.json
│   ├── xgboost_delta_model.pkl
│   ├── main_model_results.json
│   ├── xgboost_tuned_final.pkl
│   ├── tuning_results.json
│   ├── shap_values.npy
│   └── shap_feature_importance.csv
├── notebooks/                 # run in this order
│   ├── check_data.ipynb            # 1. raw data checks, gap analysis
│   ├── clean_data.ipynb            # 2. reindexing, interpolation, merging
│   ├── acf_pacf_analysis.ipynb     # 3. autocorrelation, lag selection
│   ├── feature_engineering.ipynb   # 4. time, lag, rolling, weather features + target
│   ├── eda.ipynb                   # 5. exploratory analysis on the feature set
│   ├── baseline_model.ipynb        # 6. RF / Linear Regression vs persistence
│   ├── delta_target_test.ipynb     # 7. RF on the delta (change) target
│   ├── main_model.ipynb            # 8. XGBoost delta + model x target ablation
│   ├── hyperparameter_tuning.ipynb # 9. RandomizedSearchCV on XGBoost
│   └── shap_analysis.ipynb         # 10. SHAP explainability
├── src/
│   └── ingestion/
│       ├── openaq_pipeline.py
│       └── weather_pipeline.py
├── requirements.txt
├──README.md
└──.gitignore
```

## Setup

1. Create and activate a virtual environment, then install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. Get a free API key from [explore.openaq.org](https://explore.openaq.org) (sign up → account → API keys).

3. Create a `.env` file in the project root:
   ```
   OPENAQ_API_KEY=your_actual_key_here
   ```
   No quotes around the key.

## Running the Pipeline

Pull historical data for one station:

```bash
python src/ingestion/openaq_pipeline.py --location_id <ID> --start [start_date] --end [end_date] --out data/raw/<station_name>.csv
```

For Teku:
```bash
python src/ingestion/openaq_pipeline.py --location_id 6093551 --start 2025-11-25 --end 2026-09-04 --out data/raw/teku.csv
```

Run once per station, changing `--location_id` and `--out` each time.

For Mid Baneshwor:
```bash
python src/ingestion/openaq_pipeline.py --location_id 6142022 --start 2025-11-25 --end 2026-09-04 --out data/raw/mid_baneshwor.csv
```
## Checking Raw Data (notebooks/check_data.ipynb)

1. Loads the raw CSVs from data/raw/.
2. Prints shape and date range (.min() / .max() on datetime_utc).
3. Reindexes to a full hourly timeline per station to expose true gaps (pd.date_range(freq="h") + .reindex()).
4. Plots raw coverage before and after reindexing.
5. Calculates completeness % (len(df) / expected_hours).
6. Groups consecutive missing hours into individual gaps and measures each gap's length (.groupby() on a boolean-change cumsum).
7. Prints gap length stats (.describe(), top 10 longest gaps) and gaps-per-month counts.

## Data Cleaning (notebooks/clean_data.ipynb)

1. Reindexes each station to a full hourly timeline (makes missing hours explicit NaN rows).
2. Adds a was_missing flag column before interpolating.
3. Interpolates gaps ≤ 3 hours (limit=3, limit_area="inside"). Leaves gaps > 3 hours as NaN.
4. Adds a station column and concatenates both stations into one file (not averaged).
5. Output: data/processed/kathmandu_merged.csv

Raw completeness found:

Station	Completeness	Longest gap
Teku	63.9%	391 hrs
Mid Baneshwor	57.1%	1,318 hrs

## ACF / PACF Analysis (notebooks/acf_pacf_analysis.ipynb)

1. Loads data/processed/kathmandu_merged.csv and splits the PM2.5 series per station.
2. Fills gaps with linear interpolation (for this analysis only, not saved).
3. Plots ACF and PACF up to 72 hours to check for daily and multi-day patterns.
4. Extracts the top 10 PACF lags numerically (PACF isolates each lag's direct effect, so it is used for lag selection).

Top PACF lags found:

Station	Top lags (by |PACF|)
Teku	1 (0.93), 8, 7, 9, 19, 20, 26, 23, 6, 22
Mid Baneshwor	1 (0.95), 2, 8, 9, 25, 7, 18, 22, 6, 19

The very high lag-1 correlation means next-hour PM2.5 is mostly explained by the current hour, which is why naive persistence is a strong baseline. Lags 1, 2, 6-9 were carried into feature engineering, plus 24h and 48h for the daily cycle.

## Feature Engineering (notebooks/feature_engineering.ipynb)

Target: target_pm25_next_1h - PM2.5 one hour ahead, via groupby("station")["pm25"].shift(-1).

Dropped: pm1 (correlation with pm25 = 0.994, redundant).

Time features: hour, day_of_week, month, is_weekend (Saturday only, Nepal's weekend).

Cyclical encoding: hour_sin/cos, month_sin/cos.

Lag features: pm25_lag_1h/2h/6h/7h/8h/9h/24h/48h (chosen from PACF), via groupby("station")["pm25"].shift(lag).

Rolling features: pm25_rolling_mean_6h/24h, pm25_rolling_std_6h/24h, via groupby("station")["pm25"].transform(...), min_periods=1.

Weather features: WS10M (wind speed), WD10M (wind direction), PRECTOTCORR (precipitation) from data/raw/<station>_weather.csv. -999 fill values replaced with NaN, timestamps made timezone-naive UTC, merged on the floored hour + station (row count asserted unchanged). Wind direction also encoded as wind_dir_sin/cos.

Station encoding: one-hot via pd.get_dummies.

Target NaNs: rows dropped (5,232 rows, 38.5%; 8,343 remain). Other NaNs left as-is (XGBoost handles them natively; Random Forest and Linear Regression use a median SimpleImputer fitted on each training fold).

Final feature set: 32 features.

Output: data/processed/kathmandu_features.csv

## Exploratory Data Analysis (notebooks/eda.ipynb)

1. PM2.5 distribution overall and per station.
2. Diurnal, day-of-week and monthly patterns.
3. Full time series per station.
4. Correlation matrix (PM2.5, target, weather, lags, rolling means, time).
5. PM2.5 vs temperature / wind speed / humidity scatter plots.
6. Missingness by column, and was_missing rate by hour and month.
7. Weather distributions, wind rose, weather seasonality and per-station weather comparison.

Key observations:

- PM2.5 is clearly higher Dec-Feb and drops through the monsoon (Jun-Sep), consistent with a seasonal burning / winter inversion signal.
- Teku has higher median PM2.5 (55.4) than Mid Baneshwor (50.5).
- Diurnal peaks are at hours 2 and 15 UTC, which is about 07:45 and 20:45 Nepal time (UTC+05:45), i.e. morning and evening rush hours. All hour features are in UTC.

## Validation Setup (all model notebooks)

- TimeSeriesSplit with 5 folds (expanding window). Each fold trains on all earlier data and tests on the next ~1,390 rows, so the model never sees the future.
- Test windows: Fold 1 Dec 24 - Jan 30, Fold 2 Jan 30 - Apr 4, Fold 3 Apr 4 - Jun 6, Fold 4 Jun 6 - Jul 30, Fold 5 Jul 30 - Sep 3.
- Metrics: RMSE, MAE, R² on absolute next-hour PM2.5, averaged across folds.
- Reference baseline: naive persistence (prediction = current PM2.5). Skill = 1 - RMSE_model / RMSE_persistence; above 0 means the model beats persistence.

## Baseline Models (notebooks/baseline_model.ipynb)

Compares four approaches on the same folds, all predicting absolute next-hour PM2.5:

| Model | RMSE | MAE | R² |
|---|---|---|---|
| Naive persistence | 14.51 | 9.40 | 0.726 |
| Random Forest (full features) | 16.34 | 10.72 | 0.676 |
| Random Forest (pm25 only) | 18.02 | - | 0.567 |
| Linear Regression (full features) | 19.75 | - | 0.537 |

None of the models beat persistence (RF: -17.0% RMSE vs persistence). This motivated reframing the target.

Output: models/baseline_results.json

## Delta Target Test (notebooks/delta_target_test.ipynb)

1. Reframes the target as the change: y_delta = pm25(t+1) - pm25(t).
2. Trains a Random Forest on y_delta (rows with NaN delta dropped from training).
3. Reconstructs the absolute prediction as current pm25 + predicted delta and scores it against the true next-hour value.
4. Compares against persistence on the same folds and plots feature importances for the delta model.

Result: Delta RF RMSE 15.68, R² 0.669. Improves on the absolute RF (16.34) but still -8.1% vs persistence.

## Main Model (notebooks/main_model.ipynb)

1. Trains XGBoost on the delta target (n_estimators=100, learning_rate=0.05, max_depth=3). XGBoost handles NaNs natively, no imputation.
2. Runs an ablation of model (RF vs XGBoost) x target (absolute vs delta) + persistence, with the same training rows and the same scoring rows for every approach.
3. Plots RMSE per fold, predicted vs actual, and feature importances.

Ablation results:

| Model | RMSE | MAE | R² | Mean skill | Folds beating persistence |
|---|---|---|---|---|---|
| XGBoost (delta) | 13.54 | 8.53 | 0.772 | +0.032 | 4/5 |
| XGBoost (absolute) | 16.20 | 11.12 | 0.634 | -0.208 | 3/5 |
| Random Forest (delta) | 15.68 | 10.73 | 0.669 | -0.170 | 2/5 |
| Random Forest (absolute) | 16.23 | 10.65 | 0.688 | -0.147 | 2/5 |
| Naive persistence | 14.51 | 9.40 | 0.726 | 0 | - |

XGBoost on the delta target is the only approach that beats persistence on average (+6.7% RMSE). The delta target lowers XGBoost RMSE by 16.4% vs the absolute target. Fold 3 (Apr-Jun) is the hardest; every model loses to persistence there.

Output: models/xgboost_delta_model.pkl, models/main_model_results.json

## Hyperparameter Tuning (notebooks/hyperparameter_tuning.ipynb)

1. RandomizedSearchCV (n_iter=50, 250 fits) over n_estimators, max_depth, learning_rate, subsample, colsample_bytree, min_child_weight, reg_alpha, reg_lambda, using the same TimeSeriesSplit and neg RMSE on the delta target.
2. Re-evaluates the best parameters per fold on absolute PM2.5.
3. Trains the final model on all data and saves it.

Best parameters: n_estimators=200, max_depth=4, learning_rate=0.01, subsample=0.6, colsample_bytree=0.6, min_child_weight=3, reg_alpha=0.1, reg_lambda=1.5.

| Model | RMSE | R² |
|---|---|---|
| Tuned XGBoost (delta) | 12.87 | 0.780 |
| Main model XGBoost (delta, 100/0.05/3) | 13.54 | 0.772 |
| Naive persistence | 14.52 | 0.725 |

Tuned model: +11.3% RMSE vs persistence, about 5.6% better than the main model.

Output: models/xgboost_tuned_final.pkl, models/tuning_results.json

## SHAP Explainability (notebooks/shap_analysis.ipynb)

Uses shap.TreeExplainer (exact SHAP values for tree models) on the final tuned XGBoost. Since the model predicts the delta, SHAP values explain what drives PM2.5 to rise or fall in the next hour, which directly targets spikes.

1. Global importance (mean |SHAP| bar plot) and beeswarm plot.
2. Dependence plots for the top 4 features.
3. Local force plots for the largest actual increase and largest actual drop.
4. Seasonal comparison: winter (Dec-Feb) vs monsoon (Jun-Aug).
5. Station comparison: Teku vs Mid Baneshwor.

Top features by mean |SHAP|, per station:

| Rank | Feature | Teku | Mid Baneshwor |
|---|---|---|---|
| 1 | hour_sin | 1.93 | 1.84 |
| 2 | pm25_lag_9h | 1.45 | 1.45 |
| 3 | hour_cos | 1.11 | 1.04 |
| 4 | um003 | 1.03 | 1.04 |
| 5 | pm25_lag_1h | 1.01 | 0.94 |
| 6 | pm25 | 0.88 | 0.84 |
| 7 | pm25_lag_8h | 0.46 | 0.46 |
| 8 | pm25_rolling_mean_6h | 0.41 | 0.40 |
| 9 | WS10M | 0.40 | 0.40 |
| 10 | temperature | 0.33 | 0.36 |
| 11 | month_cos | 0.32 | 0.30 |
| 12 | hour | 0.26 | 0.24 |

Key findings:

- Time of day (hour_sin/cos) is the strongest driver of hourly PM2.5 change, consistent with the rush-hour diurnal cycle.
- pm25_lag_9h ranks second, matching the 7-9h PACF peaks (roughly the gap between the morning and evening peaks).
- um003 (count of particles >0.3 µm) is a strong leading signal for PM2.5 change.
- Winter SHAP magnitudes are about 1.5-3x monsoon for most features; wind speed matters about 3.7x more in winter (0.71 vs 0.19).
- Teku and Mid Baneshwor have almost identical driver rankings.
- The model under-predicts extreme spikes: for the largest increase (2026-04-02 22:15 UTC, 67.6 → 258.0, +190.4) it predicted only +2.5. For the largest drop (2026-01-23, -142.1) it predicted -55.8.

Output: models/shap_values.npy, models/shap_feature_importance.csv

