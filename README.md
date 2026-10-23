# Telecom Predictive Maintenance Prototype

An end-to-end research prototype for predicting device failure within 24 hours and routing qualifying predictions into a local SQLite mock CRM. This is intended for synthetic or approved data, and is not a production network integration.

## Scope and assumptions

- Telemetry is one row per device every 5 minutes. `traffic_load` is normalized to `[0, 1]`; error rate and latency are nonnegative; temperature is Celsius.
- Four raw measurements plus a causal seven-day error-rate slope, 1-hour latency standard deviation, and experimental thermal-stress proxy form the seven model inputs. The default thermal reference is 60 C and is an explicit simulation assumption, not a physical engineering limit.
- A positive label means a failure occurs after the current timestamp and within 24 hours. Tail rows without full future coverage are unknown and excluded.
- Temporal splits are 70/10/20 per device. Model threshold is selected on validation predictions only.
- Inference requires 7 days of consecutive telemetry plus enough samples to form a full model window after rolling-feature warm-up (2,159 samples for the default 144-step LSTM), then uses the latest 144 feature rows.
- CRM handling is mock-only. A human reviews alerts; this code does not dispatch maintenance or contact customers.

## Setup

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -e ".[training,dev]"
```

If TensorFlow is not available for your platform, use the code in a compatible Python runtime and copy the generated model bundle into `models/`.

## Generate example data and train

```powershell
python -m predictive_maintenance.simulator
```

The simulator function is also available from Python:

```python
from predictive_maintenance.simulator import simulate_telemetry
simulate_telemetry(devices=4, days=30).to_csv("data/raw/synthetic.csv", index=False)
```

Training can consume this CSV or generate its own synthetic data when `--input` is omitted:

```powershell
python -m predictive_maintenance.train --epochs 30
python -m predictive_maintenance.train --input data/raw/synthetic.csv --output models/run-001
```

Each output directory includes the scaler, config, manifest, validation-selected thresholds, metrics, test predictions, and test window index. The simulator is useful for exercising the software; simulated scores do not establish performance on real telecom networks.

The implementation's automated checks are:

```powershell
$env:PYTHONPATH="src"
python -m pytest -q
```

For a quick training and API smoke check, train one epoch into a disposable bundle and point the service at it:

```powershell
python -m predictive_maintenance.train --epochs 1 --output models/smoke
$env:MODEL_DIR="models/smoke"
uvicorn predictive_maintenance.api:app --app-dir src --reload
```

Use the default 30-epoch training command for the research run. The one-epoch bundle is only for checking the code path.

`requirements-lock.txt` records the resolved package versions used for the local Windows verification run. The dependency list in `pyproject.toml` remains the portable installation entry point.

## Run API and mock CRM

```powershell
$env:MODEL_DIR="models"
uvicorn predictive_maintenance.api:app --app-dir src --reload
```

Open `http://127.0.0.1:8000/docs`. Endpoints: `GET /health`, `POST /predict`, `GET /alerts`, `GET /alerts/{alert_id}`, and `PATCH /alerts/{alert_id}/status`. Set `MOCK_CRM_DB` to choose the SQLite path. The `/predict` request contains one device and its recent telemetry; alert creation can be disabled with `create_alert: false`.

## Layout

```text
config/default.yaml
src/predictive_maintenance/{schema,features,windows,simulator,metrics,train,inference,alerts,store,api}.py
data/raw/              # local, ignored
models/                # local model bundles, ignored
```

## Limitations before any pilot

The simulator is synthetic, feature thresholds need engineering review, the basic priority mapping is only a prototype policy, and the API has no authentication or production CRM connector. Use approved representative data and complete privacy, security, calibration, operational, and CRM sandbox reviews before any real deployment.
