# Preprocessing Stack — JupyterLab in Docker (Telecom Churn)
**Step-by-step guide — produce the training data before running the AutoML project**

**Purpose.** This guide builds a **separate, standalone Docker stack** whose only job is to run a Jupyter notebook. The notebook downloads the Telecom Customer Churn dataset, cleans it, encodes it, splits it, and produces the three CSV files the main AutoML project needs:

- `train.csv`
- `sample_test.csv`
- `sample_test_labeled.csv`

This stack does **not** train a model. It only prepares data. When it is done, you copy the three files into the main project and run the AutoML stack as usual.

**Why a separate stack?** Preprocessing is a different concern from serving a model. Keeping it in its own folder with its own `docker-compose.yml` keeps both projects clean and reproducible. You can re-run preprocessing any time without touching the AutoML containers.

---

## 0. Where this folder lives

This guide assumes the preprocessing folder sits **next to** the main project files:

```
End-to-End-AutoML-Insurance-main/
├── backend/                       ← the main AutoML project
├── frontend/
├── docker-compose.yml             ← the main stack
└── preprocessing-telecom-churn/   ← THIS stack (preprocessing only)
    ├── docker-compose.yml
    ├── requirements.txt
    ├── .gitignore
    ├── data/
    │   └── raw/                   ← downloaded raw CSV goes here
    ├── output/                    ← the 3 produced CSV files appear here
    └── notebooks/
        └── 01_preprocess_telecom_churn.ipynb
```

All the files above have already been created for you. This guide explains how to **use** them.

---

## 1. Expected Final Result

- A JupyterLab server running in Docker, accessible at `http://localhost:8888/lab`.
- The raw dataset downloaded into `preprocessing-telecom-churn/data/raw/telco_churn.csv`.
- Three files generated in `preprocessing-telecom-churn/output/`:
  - `train.csv` (with the `Churn` column)
  - `sample_test.csv` (without `Churn`)
  - `sample_test_labeled.csv` (with `Churn`)
- The three files copied into the main project's `backend/data/`.

---

## 2. The Dataset

**Telco Customer Churn.** Each row is one telecom subscriber. The original columns are:

```
customerID, gender, SeniorCitizen, Partner, Dependents, tenure, PhoneService,
MultipleLines, InternetService, OnlineSecurity, OnlineBackup, DeviceProtection,
TechSupport, StreamingTV, StreamingMovies, Contract, PaperlessBilling,
PaymentMethod, MonthlyCharges, TotalCharges, Churn
```

Example of the first rows (exactly the format the notebook expects):

```
7590-VHVEG,Female,0,Yes,No,1,No,No phone service,DSL,No,Yes,No,No,No,No,Month-to-month,Yes,Electronic check,29.85,29.85,No
5575-GNVDE,Male,0,No,No,34,Yes,No,DSL,Yes,No,Yes,No,No,No,One year,No,Mailed check,56.95,1889.5,No
3668-QPYBK,Male,0,No,No,2,Yes,No,DSL,Yes,Yes,No,No,No,No,Month-to-month,Yes,Mailed check,53.85,108.15,Yes
```

**Target column:** `Churn` (`Yes` = the customer left, `No` = the customer stayed).

---

## 3. Prerequisites

You only need **Docker Desktop running**. The Jupyter image already contains pandas, numpy, and scikit-learn — you do not install anything on your machine.

### Step 3.1 — Confirm Docker is running

```powershell
docker info | Select-String "Server Version"
```

If you see a version line, you are good. If you see an error, open Docker Desktop and wait until it says **Running**.

---

## 4. Start the JupyterLab Stack

### Step 4.1 — Go into the preprocessing folder

```powershell
cd "c:\Users\rehou\Documents\00-dream\End-to-End-AutoML-Insurance-main\preprocessing-telecom-churn"
```

Confirm the files are there:

```powershell
Get-ChildItem -Recurse -Name
```

You should see `docker-compose.yml`, `requirements.txt`, the `notebooks/` folder with the `.ipynb`, and the empty `data/raw/` and `output/` folders.

### Step 4.2 — Start the container

```powershell
docker compose up
```

The first time, Docker downloads the JupyterLab image (about 1–2 GB). This takes a few minutes. On later runs it starts in seconds.

Watch the logs. When you see a line similar to this, the server is ready:

```
notebook  | Jupyter Server ... is running at:
notebook  | http://127.0.0.1:8888/lab
```

> The login token is **disabled** in `docker-compose.yml` (`--IdentityProvider.token=''`), so you can open the URL directly without copying a token.

### Step 4.3 — Open JupyterLab in your browser

Open this address:

```
http://localhost:8888/lab
```

You should see the JupyterLab interface.

> **If the page asks for a token or password:** the disable flags did not apply for your image version. Look in the terminal logs for a line like `http://127.0.0.1:8888/lab?token=abc123...` and open that full URL instead.

---

## 5. Run the Notebook

### Step 5.1 — Open the notebook

1. In the JupyterLab file browser on the left, double-click the `notebooks` folder.
2. Double-click `01_preprocess_telecom_churn.ipynb`.

The notebook opens with 8 numbered steps, each explained in a markdown cell.

### Step 5.2 — Run all cells

In the top menu, click **Run → Run All Cells**.

The notebook will, in order:

| Notebook step | What it does | What to check in the output |
|---|---|---|
| Step 1 | Imports and folder paths | Prints the absolute paths for raw and output folders |
| Step 2 | Downloads the CSV with `requests` | `Download OK. Size: ... bytes` |
| Step 2b | Manual fallback (only if download failed) | Instructions, no code to run |
| Step 3 | Loads the raw CSV | `Shape: (7043, 21)` and a preview table |
| Step 4 | Cleans the data | `Missing TotalCharges after conversion: 11` and the churn ratio |
| Step 5 | One-hot encodes categoricals | `Shape after encoding: (7043, ~46)` |
| Step 6 | Splits into train/test | The three shapes printed |
| Step 7 | Saves the three CSV files | Three absolute file paths |
| Step 8 | Sanity checks | `train.csv has_Churn=True`, `sample_test.csv has_Churn=False`, `sample_test_labeled.csv has_Churn=True` |

### Step 5.3 — If the download fails inside the container

If Step 2 prints `Download FAILED` (some networks block container traffic), use the manual fallback shown in **Step 2b** of the notebook. The simplest option is to run this on your machine (in the `preprocessing-telecom-churn` folder):

```powershell
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/IBM/telco-customer-churn-on-icp4d/master/data/Telco-Customer-Churn.csv" `
  -OutFile "data\raw\telco_churn.csv"
```

Then go back to the notebook and run **Run → Run All Cells** again.

---

## 6. Verify the Output Files

### Step 6.1 — Check the files on your machine

Open a new PowerShell (or reuse one) in the preprocessing folder:

```powershell
Get-ChildItem output
```

Expected:

```
train.csv
sample_test.csv
sample_test_labeled.csv
```

### Step 6.2 — Confirm the column counts

```powershell
Import-Csv output\train.csv | Select-Object -First 1 | Get-Member -MemberType NoteProperty | Measure-Object
```

`train.csv` and `sample_test_labeled.csv` must have the same number of columns. `sample_test.csv` has exactly **one fewer** (it is missing the `Churn` column).

---

## 7. Stop the JupyterLab Stack

When you are done preprocessing:

```powershell
docker compose down
```

Your `output/` files stay on disk — they are not deleted when the container stops.

---

## 8. Copy the Files Into the Main AutoML Project

This is the bridge between the two stacks.

### Step 8.1 — Copy the three files

From inside the `preprocessing-telecom-churn` folder:

```powershell
Copy-Item output\train.csv               ..\backend\data\processed\train.csv -Force
Copy-Item output\sample_test.csv         ..\backend\data\sample_test.csv -Force
Copy-Item output\sample_test_labeled.csv ..\backend\data\sample_test_labeled.csv -Force
```

### Step 8.2 — Verify they landed in the main project

```powershell
Test-Path ..\backend\data\processed\train.csv
Test-Path ..\backend\data\sample_test.csv
Test-Path ..\backend\data\sample_test_labeled.csv
```

All three must print `True`.

> **Note:** do NOT copy `train_col_types.json`. The main project's `train.py` generates it automatically during training.

---

## 9. Adapt the Main Project for Churn

The main project was built for the insurance dataset. To switch it to Telecom Churn, change just **two files**.

### Step 9.1 — In the main `docker-compose.yml` (trainer service)

Change the target column and the model name:

```yaml
trainer:
  command: ["python", "train.py", "--target", "Churn"]
  environment:
    MODEL_NAME: telecom-churn-automl
```

And in the `backend` service, set the same `MODEL_NAME`:

```yaml
backend:
  environment:
    MODEL_NAME: telecom-churn-automl
```

### Step 9.2 — In the main `frontend/app.py`

Change the two constants near the top:

```python
TARGET_COL = 'Churn'
LABELS     = {1: 'Likely to cancel — intervention needed', 0: 'Likely to stay'}
```

### Step 9.3 — Launch the main stack

From the main project folder:

```powershell
cd ..
docker compose down -v
docker compose up --build
```

The `-v` forces a fresh training run on the new Churn data.

---

## 10. Common Errors and Solutions

### Error 10.1 — `http://localhost:8888` asks for a token

**Cause:** your JupyterLab image version ignored the token-disable flags.

**Solution:** read the container logs (`docker compose logs notebook`), find the line containing `?token=`, and open that full URL.

### Error 10.2 — Step 2 prints `Download FAILED`

**Cause:** the container cannot reach the internet (corporate firewall, offline machine).

**Solution:** use the host PowerShell download in Step 5.3, then re-run all cells.

### Error 10.3 — `KeyError: 'Churn'` when running the notebook

**Cause:** the raw CSV you placed manually has a different column name (e.g., the Kaggle file). 

**Solution:** open the CSV and confirm the last column is named exactly `Churn`. If your file uses a different name, rename it in the file or adjust the `TARGET` variable in Step 6 of the notebook.

### Error 10.4 — `output/` files do not appear on my machine

**Cause:** the volume mount path is wrong, or you ran the notebook from a different directory.

**Solution:** confirm you started `docker compose up` from inside `preprocessing-telecom-churn`. The `output/` folder must be a sibling of `docker-compose.yml`.

### Error 10.5 — Port 8888 already in use

**Cause:** another Jupyter server is running.

**Solution:**

```powershell
netstat -ano | findstr ":8888 "
Stop-Process -Id <PID> -Force
```

Or change the port in `docker-compose.yml` to `"8889:8888"` and open `http://localhost:8889/lab`.

---

## 11. Summary

- You started a standalone JupyterLab stack with `docker compose up`.
- You opened the notebook at `http://localhost:8888/lab`.
- The notebook downloaded the Telecom Churn data with `requests`, cleaned it (dropped `customerID`, fixed `TotalCharges`, encoded `Churn`), one-hot encoded the categoricals, split it, and saved three CSV files.
- You copied those files into the main AutoML project.
- You changed `--target Churn`, `MODEL_NAME`, `TARGET_COL`, and `LABELS`, then launched the main stack.

**Phrase to remember:** preprocessing and serving are two separate stages. Keeping the preprocessing notebook in its own Docker stack makes the data preparation reproducible and independent from the model-serving pipeline.
