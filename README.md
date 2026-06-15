# Preprocessing Stack — Telecom Churn (JupyterLab in Docker)

Standalone Docker stack that runs a Jupyter notebook to download, clean, and split
the Telecom Customer Churn dataset. It produces three CSV files used by the main
AutoML project. **It does not train any model.**

---

## Quick start (commands)

All commands are run from **this folder** (`preprocessing-telecom-churn`).

### 1. Go into this folder

```powershell
git clone 
cd preprocessing-telecom-churn/
```

### 2. Start JupyterLab

```powershell
docker compose up
```

> First run downloads the Jupyter image (~1-2 GB). Later runs start in seconds.

### 3. Open JupyterLab in your browser

```
http://localhost:8888/lab
```

The login token is disabled, so this opens directly.

### 4. Run the notebook

In JupyterLab: open `notebooks/01_preprocess_telecom_churn.ipynb`, then click
**Run -> Run All Cells**. The three CSV files appear in the `output/` folder.

### 5. Stop the stack

Press `Ctrl+C` in the terminal, then:

```powershell
docker compose down
```

---

## Run in the background (optional)

```powershell
docker compose up -d
docker compose logs -f notebook
```

Stop it later with:

```powershell
docker compose down
```

---

## If JupyterLab still asks for a token

The token is disabled in `docker-compose.yml`. If your image version ignores that,
get the token from the logs:

```powershell
docker compose logs notebook | Select-String "token="
```

Or from inside the container:

```powershell
docker compose exec notebook jupyter server list
```

Copy the full URL (the part after `?token=`) into your browser.

---

## If the dataset download fails inside the notebook

Download it on your machine instead (from this folder), then re-run all cells:

```powershell
Invoke-WebRequest `
  -Uri "https://raw.githubusercontent.com/IBM/telco-customer-churn-on-icp4d/master/data/Telco-Customer-Churn.csv" `
  -OutFile "data\raw\telco_churn.csv"
```

---

## After preprocessing: copy the files into the main project

```powershell
Copy-Item output\train.csv               ..\backend\data\processed\train.csv -Force
Copy-Item output\sample_test.csv         ..\backend\data\sample_test.csv -Force
Copy-Item output\sample_test_labeled.csv ..\backend\data\sample_test_labeled.csv -Force
```

Verify:

```powershell
Test-Path ..\backend\data\processed\train.csv
Test-Path ..\backend\data\sample_test.csv
Test-Path ..\backend\data\sample_test_labeled.csv
```

All three must print `True`.

---

## Useful commands

| Goal | Command |
|---|---|
| Start | `docker compose up` |
| Start in background | `docker compose up -d` |
| See logs | `docker compose logs -f notebook` |
| Find the token | `docker compose logs notebook \| Select-String "token="` |
| Open a shell in the container | `docker compose exec notebook bash` |
| Stop | `docker compose down` |
| Check files produced | `Get-ChildItem output` |

---

## What the notebook produces

| File | Contains `Churn`? | Used for |
|---|---|---|
| `output/train.csv` | yes | Training the AutoML model |
| `output/sample_test.csv` | no | Prediction-only demo in the UI |
| `output/sample_test_labeled.csv` | yes | Prediction + evaluation (confusion matrix) |

Full step-by-step explanation: see `../06-GUIDE-PREPROCESSING-NOTEBOOK.md`.
