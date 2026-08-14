# Multisport Training Analytics

A small end-to-end data engineering project built on Databricks. Raw sports-watch
exports are anonymized, loaded into a relational schema with PySpark and Delta
Lake, queried with SQL, and summarized in a dashboard.

Source data: personal training sessions (running, cycling, swimming, strength
training, and more) exported from Polar Flow (a sports watch platform), used and
published with consent. Only the code and aggregate visualizations are shared
here; no row-level data is included.

## Pipeline

**1. `multisport_training_data_anonymization.ipynb`**
Reads raw session export files from a Unity Catalog volume. For each session:
replaces the device ID, session ID, and per-exercise IDs with deterministic,
salted SHA-256 hashes (namespaced so different ID types cannot be cross-matched);
drops every field not on an explicit allow-list, keeping only the heart rate,
speed, cadence, altitude, distance, and temperature streams plus a small set of
physiological thresholds (VO2 max, resting heart rate, and similar); cleans
invalid or missing values; and maps numeric sport codes to readable names via a
verified lookup table, flagging any unrecognized codes for manual review.
Writes the anonymized sessions back to a separate volume, with a safeguard that
refuses to overwrite existing output if a run produces nothing.

**2. `multisport_training_database.ipynb`**
PySpark notebook that turns the anonymized files into a small relational schema
in Unity Catalog, written as Delta tables:
- `users`: one row per anonymized user, with session count and activity span.
- `sessions`: one row per session-exercise (so a multi-discipline session, such
  as a triathlon, produces one row per leg), with the athlete's physiological
  thresholds at the time.
- `samples_<sport>` (one table per sport, e.g. `samples_running`,
  `samples_cycling`): one row per second of exercise, with the sensor streams
  pivoted into columns.

Every table has an explicit typed schema. The notebook ends with SQL checks:
table listings, per-sport session and hour totals, row counts, and a join
between `sessions` and a samples table to confirm the schema holds together.

**3. `multisport_training_exploratory_analysis.ipynb`**
Pure SQL notebook querying the tables above: per-session maximum heart rate, a
minute-by-minute heart rate profile for the single highest-heart-rate session,
and a data-quality check comparing the watch's reported speed against speed
calculated from raw distance deltas, filtering out clearly bad GPS readings
before computing a heart-rate-per-speed efficiency metric grouped by
temperature. Uses window functions (`LAG`) and safe division (`try_divide`).

**4. Dashboard: Training Data Analysis (2 pages)**
Built on top of the tables above.
- *Total*: sessions and hours per sport, first/last session dates, overall
  duration/distance/heart-rate summary stats, and a weekly training-hours trend.
- *Cycling*: duration/distance/heart-rate/altitude-gain stats, the single
  highest-altitude-gain session, and an altitude profile chart for that session.

## Notes

- All identifiers are anonymized before leaving the raw data stage; no names,
  device serials, or original session IDs appear anywhere downstream.
- Notebook outputs (query result tables) have been cleared before publishing.
  Only code, comments, and the dashboard screenshots are shared.
- The dashboard screenshots still show the project's original working title;
  the notebooks and this README use the current name.
