# Insider Threat Detection with Isolation Forest

Unsupervised anomaly detection over corporate access logs, using
scikit-learn's `IsolationForest` to surface potential insider-threat
activity — late-night logins, unrecognized devices, access to sensitive
systems, and logins from unusual locations.

## Repository structure

```
access-log-isolation-forest/
├── data/
│   ├── generate_dataset.py        # regenerates the synthetic dataset
│   ├── synthetic_access_logs.csv  # 5,050-row synthetic access log
│   └── flagged_anomalies.csv      # output of the model (generated)
├── notebooks/
│   └── access_log_isolation_forest.ipynb   # exploratory notebook
├── src/
│   └── isolation_forest_model.py  # runnable pipeline (script version)
├── report/
│   └── Isolation_Forest_Insider_Threat_Report.docx
├── requirements.txt
└── README.md
```

## Dataset

`data/synthetic_access_logs.csv` contains 5,050 synthetic access-log
records for 50 users over January 2023:

| Column           | Description                                   |
|------------------|------------------------------------------------|
| `user_id`        | Employee identifier (`user_1`...`user_50`)     |
| `timestamp`      | Login/access timestamp                         |
| `login_location` | City or connection type the access came from   |
| `access_type`    | `read`, `write`, `upload`, or `download`        |
| `resource`       | System or resource accessed                     |
| `device_type`    | Device used to access the resource              |

About 1% of rows (50 records) are injected anomalies that combine
late-night timing, an unregistered device, and access to a sensitive
resource (`Finance DB` or `Sensitive HR Records`) — the kind of pattern
an insider-threat program would want flagged. Regenerate the dataset at
any time with:

```bash
cd data
python generate_dataset.py
```

## Model

`src/isolation_forest_model.py` implements the full pipeline:

1. **Load** the CSV and parse timestamps.
2. **Feature engineering** — derive `hour`, `day_of_week`, and boolean
   risk flags (`late_night`, `unknown_device`, `sensitive_resource`,
   `unusual_location`).
3. **Encode** categorical columns (`user_id`, `login_location`,
   `access_type`, `resource`, `device_type`) with `LabelEncoder`.
4. **Train** an `IsolationForest(n_estimators=100, contamination=0.01,
   random_state=42)` on the encoded feature matrix.
5. **Score** every record with `decision_function` (continuous anomaly
   score) and `predict` (`-1` = anomaly, `1` = normal).
6. **Explain** each flagged record in plain language using the risk
   flags computed in step 2.

### Run it

```bash
pip install -r requirements.txt
python src/isolation_forest_model.py \
    --input data/synthetic_access_logs.csv \
    --output data/flagged_anomalies.csv
```

Expected output on the provided dataset:

```
Total records scored:   5050
Anomalies flagged:      51
Results written to:     data/flagged_anomalies.csv
```

## Report

See [`report/Isolation_Forest_Insider_Threat_Report.docx`]
for the full write-up: problem statement, methodology, dataset
description, results, limitations, and recommendations.

## Notes and limitations

- `contamination=0.01` is an assumption, not a measured ground-truth
  rate — in production this should be tuned against known incidents or
  analyst feedback.
- `LabelEncoder` imposes an arbitrary numeric ordering on categorical
  values; a one-hot or embedding-based encoding would remove that
  artifact at the cost of a larger feature space.
- This is an unsupervised model: it flags statistical outliers, not
  confirmed threats. Every flagged record should be triaged by a human
  analyst before any action is taken.
