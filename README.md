# AI-Driven Cyber Threat Detection and Network Anomaly Intelligence System

**SENTINEL AI** is a machine-learning based network threat detection platform. It classifies network flows with a
**Random Forest**, flags statistically unusual flows with an **Isolation Forest**, stores every decision in **SQLite**,
and presents the results in a real-time **Streamlit** dashboard with autonomous IP blocking and human-in-the-loop review.

> Academic project (BIS703) – Department of Information Science and Engineering,
> Don Bosco Institute of Technology, Bengaluru (VTU), 2026-27.

## Features

- **Supervised detection** – Random Forest (150 trees, depth 20, balanced class weights) on 68 CICIDS2017 flow features.
- **Unsupervised anomaly detection** – Isolation Forest trained only on benign traffic (contamination 0.01).
- **REST API** (Flask) – `/predict`, `/logs`, `/blacklist`, `/defense`, `/review`, `/stats`, `/system`, `/health`.
- **Autonomous defense** – source IPs of suspicious flows are added to a blacklist; can be switched on/off at run time.
- **Human-in-the-loop** – analysts can *Verify* or *Dismiss* flagged anomalies.
- **Dashboard** – security grade, AI-analyst advice, KPIs, risk timeline, attack-vector charts, live stream, forensics,
  blocked IPs, full audit log, Markdown/CSV export.
- **Traffic simulator** – replays dataset flows so the system can be demonstrated without live capture.

## Architecture

```
CICIDS2017 flows ──► Flask API (/predict) ──► StandardScaler ──► Random Forest  ─┐
                                      └────────────────────────► Isolation Forest ─┤
                                                                                  ▼
                              SQLite (logs, blacklist) ◄── risk, severity, anomaly flag, top contributors, defense rule
                                      ▲
                       Streamlit dashboard (polls the API every 2.5 s)
```

## Dataset

The web-attack portion of **CICIDS2017** (`dataset/cicids.csv`, 170,366 flows × 78 features): BENIGN, Web Attack – Brute Force,
Web Attack – XSS, Web Attack – SQL Injection. After cleaning (`model/preprocess.py`) 170,231 flows remain
(`dataset/cleaned_cicids.csv`). The trained model therefore covers **web attacks**, not DoS/DDoS/port-scan traffic.

## Measured results (20 % held-out split, `random_state=42`)

| Metric | Value |
|---|---|
| 4-class accuracy | 99.51 % |
| Attack-vs-benign precision / recall / F1 | 98.14 % / 97.02 % / 97.58 % |
| Benign false-positive rate | 0.024 % |
| Brute Force / XSS / SQL-Injection recall | 66.9 % / 55.4 % / 50.0 % (only 4 SQLi test flows) |

Brute Force and XSS flows are frequently confused with each other; see the project report for the full discussion.

## Quick start

```bash
pip install -r requirements.txt

# 1) backend API (http://127.0.0.1:5000)
cd AI-Cyber-Threat-Detection/backend
python app.py

# 2) dashboard (new terminal, from the repository root)
streamlit run AI-Cyber-Threat-Detection/dashboard/dashboard.py
```

Once started, the backend replays benign dataset rows every 4 s. Further flows can be posted to `/predict` directly
(JSON with the model's feature columns and an optional `source_ip`) or injected with the dashboard's **Simulate Breach Attempt** button.

### Re-training the models

```bash
cd AI-Cyber-Threat-Detection/model
python preprocess.py      # cleans dataset/cicids.csv -> dataset/cleaned_cicids.csv
python train_model.py     # Random Forest + scaler  -> cyber_model.pkl, scaler.pkl
python train_anomaly.py   # Isolation Forest        -> anomaly_model.pkl
```

The shipped `.pkl` files were produced with scikit-learn 1.8.0; use the same version to load them without warnings.

### Deployment (Render)

`AI-Cyber-Threat-Detection/start.sh` starts gunicorn (Flask API on port 5000, one worker) and Streamlit on `$PORT`
(default 10000). Run it from the repository root: `bash AI-Cyber-Threat-Detection/start.sh`.

## API overview

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/health` | `ready` once the models are loaded |
| POST | `/predict` | classify one flow (JSON of the 68 model features, optional `source_ip`), log it, apply defense rule |
| GET | `/logs` | latest 100 log rows |
| GET | `/stats` | total number of logged flows |
| GET | `/blacklist` | blocked IPs |
| GET / POST | `/defense` | read / set autonomous-defense mode (`{"status": true|false}`) |
| POST | `/review` | set review status of a log (`{"id": 1, "status": "Verified"}`) |
| GET | `/system` | simulated node-activity telemetry |

## Repository layout

```
requirements.txt
AI-Cyber-Threat-Detection/
├── backend/      app.py (Flask API), database.py (SQLite helpers), cyber_threats.db
├── dashboard/    dashboard.py (Streamlit UI)
├── dataset/      cicids.csv, cleaned_cicids.csv
├── model/        preprocess.py, train_model.py, train_anomaly.py, *.pkl
├── simulator/    send_data.py
├── start.sh      production start-up script
└── test_network.py, test_predict.py   ad-hoc test scripts
```

## Team

Adithya Adivesh Diwanad (1DB23IS006), Prabhat Kumar Chaurasia (1DB23IS113), Basavaraj Kumari (1DB23IS026), Prashant Patel (1DB23IS117)
– guided by Mrs. Bhavya B V, Assistant Professor, Dept. of ISE, DBIT.
