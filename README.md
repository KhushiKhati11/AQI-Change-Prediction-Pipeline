# Air Quality Prediction using an End-to-End Data Pipeline and Explainable Machine Learning

Predicting AQI for data-scarce Kathmandu, and using SHAP to explain what drives AQI spikes (traffic, weather, seasonal burning) - not just forecasting the numbers.

## Project Overview

This project builds a full pipeline: data ingestion → cleaning → feature engineering → model training → SHAP explainability → dashboard. Data is pulled from [OpenAQ](https://openaq.org), using Kathmandu Metropolitan City's Clarity Node-S sensor stations.

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

## Project Structure

```
air-quality-etl-ml-pipeline/
├── air_venv/              # virtual environment (not committed)
├── .env                   # API keys (not committed)
├── data/
│   ├── raw/                # raw pulls from OpenAQ (not committed)
│   │   ├── teku.csv
│   │   └── mid_baneshwor.csv
│   └── processed/             # cleaned/merged/feature data (not committed)
│       ├── kathmandu_merged.csv
│       └── kathmandu_features.csv
├── notebooks/
│   ├── check_data.ipynb       # raw data checks, gap analysis
│   ├── clean_data.ipynb       # reindexing, interpolation, merging
│   └── feature_engineering.ipynb
├── src/
│   └── ingestion/
│       └── openaq_pipeline.py
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
Mid Baneshwor	57.0%	1,318 hrs

## Feature Engineering (notebooks/feature_engineering.ipynb)

Target: target_pm25_next_1h - PM2.5 one hour ahead, via groupby("station")["pm25"].shift(-1).

Time features: hour, day_of_week, month, is_weekend (Saturday only).

Lag features: pm25_lag_1h/3h/6h/24h, via groupby("station")["pm25"].shift(lag).

Rolling features: pm25_rolling_mean_6h/24h, pm25_rolling_std_6h/24h, via groupby("station")["pm25"].transform(...), min_periods=1.

Station encoding: one-hot via pd.get_dummies.

Target NaNs: rows dropped. Other NaNs left as-is (models handle them natively).

Output: data/processed/kathmandu_features.csv