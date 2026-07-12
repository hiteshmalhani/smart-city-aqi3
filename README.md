An end-to-end data pipeline that simulates a Pakistani smart-city IoT sensor
network, cross-references it against real reference data from the
[OpenAQ V3 API](https://explore.openaq.org/), and lands both sources in
Snowflake through a Bronze → Silver → Gold (medallion) architecture, feeding
a live Streamlit dashboard.

## Architecture

```
┌────────────────────┐        ┌──────────────────────┐
│  IoT Simulator       │        │  OpenAQ V3 Fetcher     │
│  10 sensors × 5 cities│      │  Pakistan locations    │
│  1 reading / 10 sec  │        │  + sensors + latest    │
└──────────┬──────────┘        └──────────┬────────────┘
           │ CSV + direct insert           │ CSV/JSON + insert
           ▼                                ▼
┌─────────────────────────────────────────────────────┐
│               Snowflake — RAW (Bronze)               │
│      RAW.IOT_READINGS        RAW.OPENAQ_RAW           │
└───────────────────────┬───────────────────────────────┘
                         │ Python ETL (Pandas/Polars)
                         │ clean, validate, tag AQI, dedupe
                         ▼
┌─────────────────────────────────────────────────────┐
│              Snowflake — CLEAN (Silver)               │
│                 CLEAN.AQI_CLEAN                        │
└───────────────────────┬───────────────────────────────┘
                         │ SQL INSERT...SELECT (per city/day)
                         ▼
┌─────────────────────────────────────────────────────┐
│            Snowflake — ANALYTICS (Gold)                │
│               ANALYTICS.CITY_DAILY                      │
└───────────────────────┬───────────────────────────────┘
                         │
                         ▼
              ┌───────────────────────┐
              │  Streamlit Dashboard   │
              └───────────────────────┘
```

Both sources load independently into Bronze. Silver joins and validates
them into one shape. Gold aggregates per city per day for dashboard KPIs.

## Project Structure

```
smart-city-aqi/
├── README.md
├── requirements.txt
├── .env.example              # copy to .env and fill in credentials
├── .gitignore
├── config.py                  # env-driven config (Snowflake, OpenAQ, sensors)
├── common/
│   ├── aqi.py                 # EPA AQI formula + category/health-risk mapping
│   └── snowflake_client.py    # shared connection + bulk-load helper
├── simulator/
│   └── iot_simulator.py       # Stage 1A — IoT sensor simulator
├── openaq/
│   └── openaq_fetcher.py      # Stage 1B — OpenAQ V3 API client
├── etl/
│   └── etl_pipeline.py        # Stage 2 — clean, validate, transform, load
├── sql/
│   ├── 01_setup_database.sql
│   ├── 02_bronze_tables.sql
│   ├── 03_silver_table.sql
│   ├── 04_gold_table.sql
│   └── 05_gold_aggregation.sql
├── dashboard/
│   └── app.py                 # Stage 4 — Streamlit dashboard
├── scripts/
│   └── run_pipeline.sh        # convenience script: run everything in order
└── data/                      # local CSV/JSON snapshots (gitignored)
```

## Setup

### 1. Install dependencies

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure credentials

```bash
cp .env.example .env
```

Edit `.env`:

```
OPENAQ_API_KEY=your_openaq_api_key_here
SNOWFLAKE_ACCOUNT=xy12345.me-central2.gcp     # from your Snowflake URL
SNOWFLAKE_USER=your_username
SNOWFLAKE_PASSWORD=your_password
SNOWFLAKE_ROLE=SYSADMIN
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_DATABASE=SMART_CITY_AQI
```

Get your OpenAQ key at https://explore.openaq.org/register (free).
Get your Snowflake account identifier from your workspace URL, e.g.
`https://app.snowflake.com/me-central2.gcp/<account>/...` →
`SNOWFLAKE_ACCOUNT=<account>.me-central2.gcp`.

> **Security note:** `.env` is gitignored. Never commit real credentials.
> If a key is ever pasted somewhere it shouldn't be (chat, screenshot,
> public repo), regenerate it immediately from the provider's dashboard.

### 3. Create the Snowflake schema

Run these in order in a Snowflake worksheet (or `snowsql -f <file>`):

```bash
snowsql -f sql/01_setup_database.sql
snowsql -f sql/02_bronze_tables.sql
snowsql -f sql/03_silver_table.sql
snowsql -f sql/04_gold_table.sql
```

(`05_gold_aggregation.sql` is executed automatically by the ETL script —
you don't need to run it manually, though you can.)

## Running Each Component

### IoT Simulator (Stage 1A)

```bash
python simulator/iot_simulator.py                 # runs indefinitely
python simulator/iot_simulator.py --minutes 30     # runs for 30 min (hackathon requirement)
python simulator/iot_simulator.py --no-snowflake   # local CSV only, no Snowflake connection needed
```

Generates one reading per sensor every 10 seconds for all 10 sensors,
applies time-of-day and anomaly-spike effects, prints UNHEALTHY/HAZARDOUS
readings to the console, and writes to `data/iot_readings.csv` +
`RAW.IOT_READINGS`.

### OpenAQ Fetcher (Stage 1B)

```bash
python openaq/openaq_fetcher.py                # locations + sensors + latest
python openaq/openaq_fetcher.py --history       # also pulls last 24h of measurements
python openaq/openaq_fetcher.py --no-snowflake  # local CSV/JSON only
```

Writes `data/openaq_locations_raw.json`, `data/openaq_raw.csv`, and loads
`RAW.OPENAQ_RAW`.

### ETL Pipeline (Stage 2)

```bash
python etl/etl_pipeline.py                  # reads local CSVs, loads Silver + refreshes Gold
python etl/etl_pipeline.py --from-snowflake # reads Bronze straight from Snowflake instead
```

Cleans and validates both sources, tags AQI category / health risk,
deduplicates, unions into `data/aqi_clean.csv`, loads `CLEAN.AQI_CLEAN`,
then runs the Gold aggregation SQL to refresh `ANALYTICS.CITY_DAILY`.

### Dashboard (Stage 4)

```bash
streamlit run dashboard/app.py
```

Shows: bar chart of average AQI per city today, line chart of AQI trend
per sensor over the last 6 hours, metric cards (highest AQI city, total
readings, % CRITICAL), and a color-coded severity table. Auto-refreshes
every 30 seconds via `st.cache_data(ttl=30)`.

### Run everything at once (demo mode)

```bash
./scripts/run_pipeline.sh 30   # simulator for 30 min, then fetch, ETL, dashboard
```

## Data Model

| Layer | Schema | Table | Purpose |
|---|---|---|---|
| Bronze | `RAW` | `IOT_READINGS` | Raw simulated sensor readings, as-is |
| Bronze | `RAW` | `OPENAQ_RAW` | Raw OpenAQ V3 API responses, as-is |
| Silver | `CLEAN` | `AQI_CLEAN` | Validated, deduplicated, unioned across both sources |
| Gold | `ANALYTICS` | `CITY_DAILY` | Daily per-city aggregates for the dashboard |

## AQI Calculation

Standard EPA piecewise-linear formula, implemented once in
`common/aqi.py` and shared by both the simulator and the ETL pipeline so
the two never disagree:

```
AQI = ((I_hi - I_lo) / (C_hi - C_lo)) × (PM2.5 - C_lo) + I_lo
```

| PM2.5 (µg/m³) | AQI | Category |
|---|---|---|
| 0.0 – 12.0 | 0–50 | GOOD |
| 12.1 – 35.4 | 51–100 | MODERATE |
| 35.5 – 55.4 | 101–150 | UNHEALTHY_FOR_SENSITIVE |
| 55.5 – 150.4 | 151–200 | UNHEALTHY |
| 150.5 – 250.4 | 201–300 | VERY_UNHEALTHY |
| 250.5 – 500.4 | 301–500 | HAZARDOUS |

## Sensor Network

| Sensor ID | City | Zone | Base Pollution |
|---|---|---|---|
| PKS_KHI_IND_01 | Karachi | Industrial | High |
| PKS_KHI_TRF_02 | Karachi | Traffic | Medium-High |
| PKS_LHR_RES_01 | Lahore | Residential | Medium |
| PKS_LHR_IND_02 | Lahore | Industrial | High |
| PKS_ISB_PRK_01 | Islamabad | Park | Low |
| PKS_ISB_TRF_02 | Islamabad | Traffic | Medium-High |
| PKS_PEW_IND_01 | Peshawar | Industrial | High |
| PKS_PEW_RES_02 | Peshawar | Residential | Medium |
| PKS_MUL_TRF_01 | Multan | Traffic | Medium-High |
| PKS_MUL_PRK_02 | Multan | Park | Low |

