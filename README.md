# LECO Auto Recloser Health

Real-time recloser health monitoring for the LECO fleet, including V1–V42 calculation, Health Index (HI), 3D dashboard, and ML-based failure prediction.

## Overview

Every 15 minutes, this project:

1. Collects one interval of raw data per auto recloser (AR).
2. Calculates the final 42 Excel variables: V1–V42.
3. Calculates a Health Index (HI) from 0–100.
4. Classifies each asset as GOOD / FAIR / POOR / CRITICAL.
5. Stores the data in MySQL / MariaDB (or SQLite).
6. Exports an ML-ready dataset with failure labels.

The fleet is 100 NOJA reclosers across 7 LECO branches:
- Negombo
- Kelaniya
- Kotte
- Nugegoda
- Moratuwa
- Kalutara
- Galle

These branch settings are configured in `config.yaml` under `areas`.

Additional features:
- Futuristic 3D dashboard via `python main.py dashboard`
- Maintenance resets that restart the repaired part’s Health Index
- Machine-learning risk prediction next to the HI every 15 minutes
- A deterministic, physics-based simulator (`SimulatedNojaSource`) as the default data source

## Documentation

- Design: [docs/design/2026-10-06-ar-phase1-design.md](docs/design/2026-10-06-ar-phase1-design.md)
- Data dictionary: [DATA_DICTIONARY.md](DATA_DICTIONARY.md)
- ML training guide: [docs/ML_COLAB_GUIDE.md](docs/ML_COLAB_GUIDE.md)

---

## 1. Installation

Requires Python 3.11+ (tested with 3.13).

```powershell
cd "D:\AR Preventive Analyze"
python -m pip install -r requirements.txt
```

---

## 2. Database setup

### MariaDB 10.6 / MySQL 8

In HeidiSQL, connected as root, run:

```sql
CREATE DATABASE IF NOT EXISTS ar_maintenance CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'ar_gen'@'localhost' IDENTIFIED BY 'choose-a-password';
CREATE USER IF NOT EXISTS 'ar_gen'@'127.0.0.1' IDENTIFIED BY 'choose-a-password';
GRANT ALL PRIVILEGES ON ar_maintenance.* TO 'ar_gen'@'localhost';
GRANT ALL PRIVILEGES ON ar_maintenance.* TO 'ar_gen'@'127.0.0.1';
FLUSH PRIVILEGES;
```

Then copy `.env.example` to `.env` and add your password:

```env
DATABASE_URL=mysql+pymysql://ar_gen:choose-a-password@127.0.0.1:3306/ar_maintenance
```

If the password contains special characters such as `@ : / ? # %`, URL-encode them first.

If no `.env` file exists, the project falls back to SQLite at:

```text
data/ar_maintenance.sqlite
```

---

## 3. Quick start

```powershell
python main.py init-db          # create tables and indexes
python main.py seed --days 30   # create 100 LECO reclosers with 30 days of history
python main.py backfill         # raw -> V1-V42 -> HI -> daily summary
python main.py validate         # must print "Validation PASSED"
python main.py status           # latest HI for every recloser
python main.py scheduler        # keep running; updates every 15 min
python main.py dashboard        # second terminal; opens the 3D dashboard
```

### Fast trial

```powershell
python main.py seed --assets 3 --days 7
python main.py backfill
```

### Moving from SQLite to MySQL

Create `.env`, then run:

```powershell
python main.py copy-db --from sqlite:///data/ar_maintenance.sqlite
python main.py validate
```

Everything continues from there. If the configured database does not contain the seeded fleet, the getter refuses to run instead of writing orphan rows.

The live database keeps recent data only. Old history for ML training is generated separately with `generate-ml-data`, and `prune-history --keep-days 30` trims an existing database after backup.

---

## 4. Commands

| Command | Description |
|---|---|
| `init-db [--reset] [--yes]` | Create the database, tables, and indexes. `--reset` drops everything and clears simulator state. |
| `seed [--assets N] [--start YYYY-MM-DD \| --days N]` | Create the fleet and hidden degradation profiles. |
| `backfill [--end DATE] [--no-compute]` | Generate history up to now (or `--end`), committed per day, then calculate V1–V42 and HI. |
| `run-once` | Collect the newest interval, then update V1–V42 and HI. Missed intervals are filled automatically. |
| `scheduler` | Run `run-once` at :00, :15, :30 and :45 (APScheduler). |
| `status` | Show the latest HI, class, health flag, worst variables, and factor scores per recloser. |
| `dashboard [--port 8501] [--lan]` | Open the 3D recloser health map in your browser. |
| `maintenance-done --asset AR-008 --part SWITCHING [--part ...] [--at "YYYY-MM-DD HH:MM"] [--note ...]` | Record maintenance and reset the repaired part’s HI. |
| `ml-status` | Show whether a trained ML model is installed in `models/`. |
| `ml-score [--start --end]` | Score stored V1–V42 rows with the installed model. |
| `prune-history [--keep-days 30] [--backup FILE \| --no-backup]` | Delete older data after a backup, keeping only what the 30-day calculations still need. |
| `set-locations --csv file.csv` | Load real GPS positions (`asset_id,latitude,longitude`) replacing the simulated ones. |
| `validate` | Run checks for gaps, ranges, counters, fault sequence logic, alarms, repairs, coverage, and HI before failures. |
| `export-dataset [--out DIR]` | Export an ML dataset with labels, ground truth, and profiles. |
| `compute-variables [--start --end]` | Recompute V1–V42 and HI for a range. |
| `compute-hi [--start --end]` | Re-score stored variables after changing HI weights or thresholds in `config.yaml`. |
| `copy-db --from URL` | Copy all data from another database into `DATABASE_URL`. |
| `generate-ml-data [--assets 100] [--days 730] [--seed N]` | Generate an ML training dataset directly, without the database. |

Logs go to the console and to `logs/ar_gen.log` (rotating). A lock file (`state/ar_gen.lock`) prevents overlapping runs, such as scheduler + task scheduler.

---

## 5. Real-time operation every 15 minutes

### Option A: Keep `python main.py scheduler` running

Run this in a terminal or as a service.

### Option B: Windows Task Scheduler

Run this once in an elevated PowerShell:

```powershell
schtasks /Create /TN "AR data getter" /SC MINUTE /MO 15 /ST 00:00 /F `
  /TR "\"C:\Python313\python.exe\" \"D:\AR Preventive Analyze\main.py\" run-once"
```

Then in the Task Scheduler GUI:
- Set **Run whether user is logged on or not**
- Set **Do not start a new instance**

Test it with:

```powershell
schtasks /Run /TN "AR data getter"
```

Then check `logs\ar_gen.log`.

Remove the task with:

```powershell
schtasks /Delete /TN "AR data getter" /F
```

Use `where python` to find your Python path. The working directory does not matter because paths resolve from the project folder.

If the machine was off, the next run fills every missed interval, keeping the data continuous.

---

## 5b. 3D dashboard and maintenance reset

```powershell
python main.py dashboard          # opens http://localhost:8501
```

### What the page shows

- A 3D map of Sri Lanka. Each recloser is shown as a column.
- Column height = risk = `100 - HI`.
- Column colour = status, ordered by priority:

| Status | Colour | Meaning |
|---|---|---|
| OFFLINE | grey | no data for 60 min |
| FAULTY | dark red | failure waiting for repair |
| LOCKOUT | purple | switch locked out after a fault |
| CRITICAL | red | HI < 40 |
| POOR | orange | HI 40–60 |
| FAIR | yellow | HI 60–80 |
| GOOD | green | HI ≥ 80 |

Other dashboard details:
- Hovering over a column shows HI, worst variables, battery, AC, comms, and switch state.
- Thin lines join reclosers on the same feeder.
- A tile count is shown above the map for each status.
- The table below the map lists the fleet, worst first.
- A detail panel shows the chosen recloser’s recent HI trend, factor scores, and event history.
- The **Maintenance done** form allows resets after repair.

### Maintenance done = HI reset of the repaired part

From the maintenance time onward, events that occurred before the repair stop counting for that specific part. Everything else still counts.

| Part | Variables that restart |
|---|---|
| Battery | V2, V6, V7 (V3, V4, V8, V9 already follow the battery) |
| SWITCHING (OSM mechanism / contacts) | V22–V27 (V11, V21 reset with a new tank) |
| Comms | V28–V32 |
| Controller | V33, V34 |
| Any reset | V39 days since maintenance |

Do not reset:
- faults (`V12–V20`)
- AC supply (`V5, V10`)
- age and environment values

Automatic resets:
- every **CORRECTIVE** repair in `maintenance_log` resets its part

Manual resets:
- the dashboard form or `maintenance-done` adds a row to `hi_reset_log`
- recalculation starts from that moment, waiting for the collector if it is busy

**Map positions:** reclosers are placed along their feeders in the 5 areas (`asset_location`, source = `SIMULATED`). Load real positions with `python main.py set-locations --csv gps.csv`.

The map background comes from an online map service, so the browser needs internet access.

---

## 6. HeidiSQL queries

```sql
-- Latest Health Index of every recloser (worst first)
SELECT h.asset_id, h.ts, h.health_index, h.health_class, h.is_healthy, h.cap_applied, h.worst_variables
FROM ar_health_index_15min h
JOIN (SELECT asset_id, MAX(ts) ts FROM ar_health_index_15min GROUP BY asset_id) m USING (asset_id, ts)
ORDER BY h.health_index;

-- Reclosers that are NOT healthy right now
SELECT h.asset_id, h.health_index, h.health_class, h.worst_variables, h.f_battery, h.f_switching, h.f_comms
FROM ar_health_index_15min h
JOIN (SELECT asset_id, MAX(ts) ts FROM ar_health_index_15min GROUP BY asset_id) m USING (asset_id, ts)
WHERE h.is_healthy = 0;

-- Daily HI trend for one recloser
SELECT DATE(ts) AS day, ROUND(AVG(health_index), 1) AS hi, MIN(health_index) AS hi_min
FROM ar_health_index_15min WHERE asset_id = 'AR-005'
GROUP BY DATE(ts) ORDER BY day;

-- Latest V1-V42 row of one recloser
SELECT * FROM ar_variables_15min WHERE asset_id = 'AR-005' ORDER BY ts DESC LIMIT 1;

-- Battery behaviour over the last 24 h
SELECT ts, battery_voltage, battery_current, battery_charge_pct, ac_supply_on, battery_temp_c
FROM telemetry_15min WHERE asset_id = 'AR-005' AND ts > NOW() - INTERVAL 1 DAY ORDER BY ts;

-- Failures, the repair and the HI during the 7 days before each failure
SELECT f.failure_id, f.asset_id, f.ts, f.component, f.description, m.ts AS repaired_at,
       (SELECT ROUND(AVG(h.health_index), 1) FROM ar_health_index_15min h
         WHERE h.asset_id = f.asset_id AND h.ts BETWEEN f.ts - INTERVAL 7 DAY AND f.ts) AS hi_7d_before
FROM failure_log f LEFT JOIN maintenance_log m ON m.failure_id = f.failure_id
ORDER BY f.ts;

-- Faults per month and outcome
SELECT DATE_FORMAT(start_ts, '%Y-%m') AS month, outcome, COUNT(*) AS n
FROM fault_events GROUP BY month, outcome ORDER BY month;

-- A fault and its trip / reclose sequence
SELECT o.ts, o.op_type, o.source, o.ar_state, o.trip_ia, o.trip_ib, o.trip_ic, o.operating_time_ms, o.result
FROM co_operations o WHERE o.fault_id = 100 ORDER BY o.ts;

-- Asset history: what maintenance changed
SELECT ts, asset_id, component, maint_type, description, details FROM maintenance_log ORDER BY ts;
```

---

## 7. ML dataset for Phase 2

```powershell
python main.py export-dataset
```

### More training data without the database

```powershell
python main.py generate-ml-data --assets 100 --days 730
```

This simulates a separate fleet in memory (seed = config seed + 1, so it is independent of the live reclosers) and writes the same columns as `export-dataset` to `output/ml_generated/`:
- `ar_ml_dataset.csv.gz`: every 15 minutes
- `ar_ml_dataset_hourly.csv.gz`: hourly, smaller, and best for Colab
- the ground truth and profiles files

Nothing is written to the database or the live simulator state. A test checks that it produces exactly the same rows as the database route.

### Training in Google Colab (recommended)

`notebooks/AR_ML_Colab.ipynb`, with step-by-step instructions in [docs/ML_COLAB_GUIDE.md](docs/ML_COLAB_GUIDE.md).

The notebook:
- tunes LightGBM with Optuna
- calibrates the probabilities
- chooses the alarm level based on a false-alarm budget
- compares ML with the HI
- explains predictions with SHAP
- adds a model for “which part will fail”
- exports `ar_ml_model.zip`

Unzip it into `models/` and run `python main.py ml-score` (it scores all stored data; `--start YYYY-MM-DD` limits it). The collector then stores the ML risk every 15 minutes in `ar_ml_prediction_15min`.

The notebook embeds the project’s feature code (`ar_gen/ml_features.py`), so training and live scoring always build the same inputs.

The older notebook `notebooks/AR_failure_prediction.ipynb` remains for reference.

This writes the following files to `output/ml/`:
- `ar_ml_dataset.csv.gz`
- `ar_ml_ground_truth.csv.gz`
- `ar_ml_asset_profiles.csv`

Load it in a notebook:

```python
import pandas as pd

df = pd.read_csv("ar_ml_dataset.csv.gz", parse_dates=["ts"])
train = df[~df["censored_30d"] & ~df["failed_now"]]
X = train.filter(regex=r"^v\d\d_").drop(columns=["v42_weather"]).join(pd.get_dummies(train["v42_weather"]))
y = train["failure_next_30d"]
```

To avoid leakage, split train and test by asset or by time, not at random.

---

## 8. Configuration

All settings are in `config.yaml`:
- seed and fleet size
- climate, load, outages
- battery and UPS, protection, comms, controller, and maintenance rates
- V1–V42 windows
- HI weights, thresholds, caps, and classes

Environment variables override `DATABASE_URL`, `AR_CONFIG`, `AR_STATE_DIR`, `AR_OUTPUT_DIR`, and `AR_LOG_DIR`, making cloud/container runs easier.

- Changing HI settings: run `python main.py compute-hi`
- Changing variable windows: run `python main.py compute-variables`
- Changing simulation settings affects only data generated from then on; the log warns about it

For a clean dataset:

```powershell
python main.py init-db --reset
python main.py seed
python main.py backfill
```

---

## 9. How realistic the data is, and its limits

- Physics-based and deterministic.
- Battery capacity fades with temperature-weighted age and cycling; internal resistance is R0 / SOH^2.5.
- Hidden thresholds generate failures; corrective maintenance follows 1–21 days later.
- The same seed and config give identical data; crash recovery regenerates any partially written rows identically.
- Telemetry is taken at the interval end; short events between polls appear in the event logs, not in telemetry.
- Faults that start just before a poll may be written when they finish, possibly one interval later.
- Variables raw RC data cannot directly provide: V3, V12, V31, V35, and V36.
- This is simulated data. Use it to build and test the pipeline and ML workflow; validate any model on real recloser data before trusting it.

---

## 10. Tests

```powershell
python -m pytest
```

The tests cover:
- config
- physics (battery, contact wear, protection timing)
- database schema
- the HI model
- the simulator: telemetry ranges, fault ↔ operation consistency, counters vs logged operations, alarm pairing, battery failure and repair, determinism, split run vs single run
- V1–V42 (each variable, no look-ahead, incremental = full)
- end-to-end pipeline: validation passes, idempotent re-run, interval-by-interval = backfill, crash recovery, copy-db

---

## 11. Project layout

```text
main.py                     CLI entry point
config.yaml                 every tunable value
ar_gen/
  cli.py, pipeline.py       commands; collect -> store -> variables -> HI
  sources/                  RecloserSource interface + SimulatedNojaSource
  sim/                      simulator: fleet, environment, load, UPS/battery, protection, switching,
                            comms, controller, maintenance, engine
  variables.py              V1-V42 calculation
  health_index.py           HI mathematical model
  dataset.py, validate.py   ML export and validation
  db/                       schema, engine, reader, writer
tests/                      pytest suite
state/                      simulator state (hidden truth) - not in the DB
output/                     hidden profiles, ground truth, ML exports
logs/                       ar_gen.log
```

---

## Final summary

This project is a complete auto-recloser health monitoring and predictive maintenance pipeline. It simulates 100 reclosers across seven LECO branches, calculates 42 health variables and a Health Index, stores everything in a database, visualizes the fleet in 3D, supports maintenance resets, and exports ML-ready datasets for future failure prediction.
