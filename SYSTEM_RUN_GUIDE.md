# Telecom Predictive Maintenance: full running guide

This guide explains how to install, start, use, and test the local prototype on Windows. Commands are for PowerShell. Run them from the project folder unless a step says otherwise.

The system uses a trained neural network to score one device's telemetry for a possible failure in the next 24 hours. When enabled and the score reaches the model threshold, it stores an alert in a local SQLite mock CRM for human review. The supplied model was trained on simulated data; its scores have not been validated on a live network. It does not connect to a commercial CRM or dispatch maintenance.

## 1. What you need

- Windows and Python 3.11, the version used to run the project locally.
- Internet access for the initial package installation.
- The project folder, including `frontend`, `backend`, `config`, and `models/demo`.
- A browser such as Edge or Chrome.

Check Python:

```powershell
python --version
```

The demo model bundle should contain `model_manifest.json`, `feature_config.yaml`, `scaler.joblib`, and `lstm_best.keras`. Check it with:

```powershell
Get-ChildItem 'D:\Ass\Sajindu\TOPIC TWO SYSTEM\models\demo'
```

## 2. First-time setup

Open PowerShell and go to the project:

```powershell
cd 'D:\Ass\Sajindu\TOPIC TWO SYSTEM'
```

Create a virtual environment and install the project, TensorFlow, Excel support, and test tools:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -e '.[training,dev]'
```

Using the environment's Python directly avoids PowerShell activation-policy issues. The first install may take several minutes because TensorFlow is large. If installation fails, read the error, confirm Python 3.11 and internet access, and rerun the install command.

## 3. Start the system

From the project folder, run:

```powershell
$env:MODEL_DIR = 'models/demo'
.\.venv\Scripts\python.exe -m uvicorn predictive_maintenance.api:app --app-dir backend/src --host 127.0.0.1 --port 8000
```

Keep this PowerShell window open while using the system. Wait for `Application startup complete`. Open:

- Dashboard: [http://127.0.0.1:8000/](http://127.0.0.1:8000/)
- API documentation: [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)
- Health: [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health)

The dashboard should show **Service online** and **Ready**. Health should return `{"status":"ok","model_loaded":true}`. To stop the server, return to its PowerShell window and press **Ctrl+C**.

If port 8000 is already in use, use `--port 8001` and open the same addresses with `8001`. Set `MODEL_DIR` in each new PowerShell session before starting the server. To select the baseline model instead of the LSTM, set `$env:MODEL_NAME = 'baseline'` before starting; to return to the LSTM, use `$env:MODEL_NAME = 'lstm'` or remove the variable.

## 4. Create sample telemetry

Open a second PowerShell window, go to the project folder, and run:

```powershell
cd 'D:\Ass\Sajindu\TOPIC TWO SYSTEM'
.\.venv\Scripts\python.exe -m predictive_maintenance.simulator
```

This creates `data/raw/synthetic.csv`, with synthetic devices and five-minute readings. It is suitable for demonstrating the workflow; it does not establish real-network performance. Running the command again replaces that synthetic CSV.

## 5. Use the dashboard

1. Start the server as described above and open the dashboard.
2. Under **Run a prediction**, select **Load synthetic demo data** to try the system immediately, or choose an `.xlsx` or `.csv` file (up to 15 MB).
3. Select a sheet for an Excel workbook and select a device. The dashboard displays its sample count and time range.
4. Leave **Create a mock alert when the threshold is met** checked to store qualifying alerts. Clear it to obtain a prediction without creating an alert.
5. Select **Run prediction**. The result shows the device, model version, risk score, decision threshold, 24-hour horizon, prediction timestamp, and alert outcome.
6. In **Mock CRM**, use **Refresh** to reload alerts. Choose `reviewed`, `actioned`, `false_alarm`, or `closed`, and select **Save**. The status is stored locally and should remain after a page refresh.

The risk score is a model output, not a guarantee that a failure will happen. Alerts require human review.

## 6. Telemetry file requirements

The selected sheet or CSV needs one row per device and timestamp with these headers:

| Column | Expected value |
| --- | --- |
| `device_id` | A synthetic or approved pseudonymous device identifier |
| `timestamp` | Measurement time, preferably ISO 8601 UTC, such as `2026-10-01T08:30:00Z` |
| `error_rate` | Nonnegative numeric interface error rate |
| `latency_ms` | Nonnegative numeric latency in milliseconds |
| `temperature_c` | Numeric temperature in degrees Celsius |
| `traffic_load` | Numeric normalized load between 0 and 1 |

For the default LSTM, each selected device needs at least **2,159 consecutive readings**, exactly five minutes apart and ordered by timestamp. This supplies the seven-day rolling feature history and 144 valid input rows. The baseline has a different window length; the dashboard reads the requirement from the selected model. Duplicate timestamps, gaps, invalid values, and missing measurements are rejected. Extra columns in a telemetry sheet are ignored by prediction.

The supplied `FaultsAssignedToMyWG_1793184706052 data base.xlsx` was checked: its single sheet contains fault records with `ID`, `Circuit Display Name`, `SA_LEA`, and `Cause Of Fault`. Those fields do not supply the six telemetry columns above. The dashboard can preview the file, but the current model cannot calculate a valid prediction from it. Do not invent measurements or treat fault descriptions as telemetry. Obtain an approved telemetry export with the required history and units before trying to predict from operational data.

## 7. Check the API

With the server running, open `/docs` for interactive request schemas and **Try it out** controls. Main endpoints are:

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/health` | Service and model load status |
| GET | `/model-info` | Model version, threshold, horizon, and required samples |
| POST | `/predict` | Score telemetry supplied as JSON |
| POST | `/workbook/preview` | Inspect sheet names, columns, and devices from a base64-encoded CSV/XLSX |
| POST | `/workbook/predict` | Score a selected device in an encoded workbook |
| GET | `/alerts` | List mock CRM alerts |
| GET | `/alerts/{alert_id}` | Retrieve an alert |
| PATCH | `/alerts/{alert_id}/status` | Save a review status |

For a quick PowerShell check in a second window:

```powershell
Invoke-RestMethod 'http://127.0.0.1:8000/health'
Invoke-RestMethod 'http://127.0.0.1:8000/model-info'
Invoke-RestMethod 'http://127.0.0.1:8000/alerts'
```

To review an existing alert, copy its actual ID from the dashboard or `/alerts` response:

```powershell
$alertId = 'PASTE-ALERT-ID-HERE'
$body = @{ status = 'reviewed'; actor = 'demo reviewer'; note = 'Reviewed locally' } | ConvertTo-Json
Invoke-RestMethod -Method Patch -Uri "http://127.0.0.1:8000/alerts/$alertId/status" -ContentType 'application/json' -Body $body
```

For `/predict`, provide one device's full telemetry history as shown by the schema in `/docs`. A single sample is not enough. The dashboard handles workbook encoding and constructs the request for you. `422` normally means input validation failed; `404` means an alert was not found; `503` means a model is not loaded.

## 8. Run automated checks

From the project folder:

```powershell
.\.venv\Scripts\python.exe -m pytest -q
```

The suite covers telemetry and window validation, features, metrics, workbook preview, prediction behavior, and mock alert storage. A passing suite checks those cases; it does not prove that every possible workbook or operational situation works.

## 9. Train a new model bundle

Training can be time consuming. The default command generates simulated telemetry when no input is supplied, trains both models, and writes a bundle:

```powershell
.\.venv\Scripts\python.exe -m predictive_maintenance.train --epochs 30 --output models/new-run
```

To train on a compatible CSV containing telemetry and `failure_event`, use:

```powershell
.\.venv\Scripts\python.exe -m predictive_maintenance.train --input data/raw/synthetic.csv --epochs 30 --output models/new-run
```

The model directory contains saved models, scaler, feature configuration, manifest, data-quality and split audits, test predictions, and metrics. For a new demonstration, stop the server, set `$env:MODEL_DIR = 'models/new-run'`, and restart it. A one-epoch run is useful for testing the training path, but it is not the full research run. Preserve each experiment in a separate directory.

## 10. Common problems

| Problem | What to check |
| --- | --- |
| Dashboard cannot connect | Confirm the server window is running and the browser uses the same port. |
| Model unavailable | Set `MODEL_DIR` to the model bundle before starting; check the server logs and required bundle files. |
| Port already in use | Stop the other server or start this one on a different port. |
| Missing columns | Choose the correct sheet and compare its headers with section 6. |
| Too few samples | Provide the model's required number of readings for the selected device. |
| Irregular sampling or duplicate timestamps | Correct the source data using documented evidence; do not fabricate readings. |
| Excel file cannot be read | Use a valid `.xlsx` file and verify `openpyxl` installed successfully. Legacy `.xls` files are not supported by this uploader. |
| No alert appears | The score may be below threshold, or the alert checkbox may be cleared. The prediction result shows the outcome. |
| A package or TensorFlow fails to import | Confirm that all commands use `.venv\Scripts\python.exe` and reinstall the project dependencies. |

The mock CRM database defaults to `data/mock_crm.sqlite3`. To use another local database, set `$env:MOCK_CRM_DB = 'data/my-demo.sqlite3'` before starting the server. The API is intended for local use and has no production authentication; do not expose it to a network without further security work.
