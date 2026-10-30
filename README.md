# Secure Merkle NIDS: Windows demo runbook

This project captures TCP/UDP packets, groups them into flows, classifies completed flows with a trained demo Random Forest, encrypts generated alerts with AES-256-GCM, and stores them in SQLite. Checkpoints contain a SHA-256 Merkle root signed with RSA-PSS.

Run all commands from the **project root** in PowerShell. Use separate PowerShell windows for long-running commands.

## 1. One-time setup

```powershell
cd 'C:\Users\Haider\Desktop\Codes\secure_merkle_nids (1)\secure_merkle_nids'
# Only if .venv does not already exist:
py -3 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

For live capture on Windows, install **Npcap** and open PowerShell **as Administrator**. The non-capture demos and manual-alert API demo do not need an elevated terminal. Run from the project root because the model, database, keys, and anchors use paths relative to it.

## 2. Train or check the model

The live detector reads `data\nids_model.joblib`. If that file already exists, skip training. Otherwise put the official CICIDS2017 CSV files under `MachineLearningCSV\MachineLearningCVE` and run:

```powershell
Test-Path .\data\nids_model.joblib
.\.venv\Scripts\python.exe scripts\train_model.py MachineLearningCSV\MachineLearningCVE
```

Training writes `data\nids_model.joblib` and `data\nids_model.metrics.json`. The model is a small NumPy Random Forest trained on eight features shared by the CSVs and the live flow extractor. It predicts `BENIGN` or `ATTACK`. Restart a running API or capture process after training so it loads the new file.

Without a model, live capture still prints `FLOW` lines but creates no classifier alerts. Local traffic can differ greatly from CICIDS2017 flows; an alert on a localhost request is a model prediction, not proof of a real attack.

## 3. Fastest self-contained demos

These scripts create **temporary** databases and keys, so they do not alter `data\forensics.db`:

```powershell
.\.venv\Scripts\python.exe scripts\demo.py
.\.venv\Scripts\python.exe scripts\proof_demo.py
.\.venv\Scripts\python.exe scripts\tamper_demo.py
```

- `demo.py`: inserts a sample alert, creates a signed checkpoint, and verifies integrity.
- `proof_demo.py`: shows the sibling hashes for record 3 of 5, verifies its inclusion proof, then rejects a forged leaf.
- `tamper_demo.py`: verifies an alert and proof, changes one byte of stored ciphertext, then shows failed integrity, failed proof, and rejected alert access. The sample alert is inserted manually; it is not a model detection.

## 4. Live packet-capture demo

For the localhost demonstration, use **three PowerShell windows**. Run the capture window as Administrator. In each window, first `cd` to the project root shown in section 1.

**Window A: start a local HTTP server to make ordinary traffic.** The `--directory` option prevents serving the project source files.

```powershell
New-Item -ItemType Directory -Force .\demo_http | Out-Null
.\.venv\Scripts\python.exe -m http.server 8765 --bind 127.0.0.1 --directory .\demo_http
```

The server prints `Serving HTTP ...`. A request for `/` may list an empty directory; a `/favicon.ico` 404 is harmless. Stop it later with `Ctrl+C`.

**Window B: find the loopback adapter, then capture for 30 seconds.** Start this before sending requests in Window C.

```powershell
.\.venv\Scripts\python.exe scripts\live_capture.py --list-ifaces
.\.venv\Scripts\python.exe scripts\live_capture.py --iface '\Device\NPF_Loopback' --duration 30
```

If the name differs on your machine, use the listed Npcap loopback adapter. For Wi-Fi or Ethernet capture, choose that interface and generate traffic through it. Omit `--duration 30` to run until `Ctrl+C`. Add `--no-open-tree` to save the visualization without launching a browser.

**Window C: send requests while Window B is capturing.**

```powershell
1..5 | ForEach-Object { Invoke-WebRequest 'http://127.0.0.1:8765/' -UseBasicParsing | Out-Null; Start-Sleep -Milliseconds 300 }
```

Window B prints `FLOW` lines and prints `ALERT` lines only for flows classified as attacks. At the end it prints counts, creates a checkpoint, verifies integrity, and opens `anchors\merkle_tree_latest.html`. A checkpoint with zero records means capture worked but produced no alerts. A benign HTTP request shows capture; it is not a guaranteed attack trigger. Use the manual-alert API demo below for a deterministic alert.

## 5. API server and PowerShell client

The FastAPI server is separate from the local HTTP traffic server. Start it in **Window A** from the project root:

```powershell
.\.venv\Scripts\python.exe -m uvicorn secure_nids.api:app --host 127.0.0.1 --port 8000
```

Open [the interactive API documentation](http://127.0.0.1:8000/docs), or use **Window B** as a PowerShell client:

```powershell
Invoke-RestMethod 'http://127.0.0.1:8000/health'
```

`model_loaded: true` confirms the running server found the model. Insert a **clearly labelled manual demo alert** and check its signed checkpoint:

```powershell
$body = @{ src_ip='10.0.0.15'; dst_ip='10.0.0.1'; src_port=51000; dst_port=22; protocol='TCP'; label='DEMO_ATTACK'; confidence=0.98; severity='high' } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri 'http://127.0.0.1:8000/alerts/manual' -ContentType 'application/json' -Body $body
Invoke-RestMethod -Method Post -Uri 'http://127.0.0.1:8000/integrity/checkpoint'
Invoke-RestMethod 'http://127.0.0.1:8000/integrity/verify'
```

The manual endpoint demonstrates storage and integrity; it does not call the classifier. To control capture through the API instead of `live_capture.py`, run the API as Administrator and use these client commands:

```powershell
$iface = [uri]::EscapeDataString('\Device\NPF_Loopback')
Invoke-RestMethod -Method Post -Uri "http://127.0.0.1:8000/capture/start?iface=$iface"
# Generate traffic in another window, as in section 4.
Invoke-RestMethod -Method Post -Uri 'http://127.0.0.1:8000/capture/stop'
Invoke-RestMethod -Method Post -Uri 'http://127.0.0.1:8000/integrity/checkpoint'
Invoke-RestMethod 'http://127.0.0.1:8000/integrity/verify'
```

The API capture endpoints return status but do not stream `FLOW` and `ALERT` lines to the client. The CLI in section 4 is the easiest live presentation. Stop the API with `Ctrl+C`.

## 6. Inspect the results

Read recent alert metadata and checkpoint roots without changing the database:

```powershell
.\.venv\Scripts\python.exe scripts\view_db.py --limit 20
```

Open `data\forensics.db` in **DB Browser for SQLite** if you prefer a table view. The `alerts` table has metadata and encrypted payload bytes; `checkpoints` has roots and signatures. Use read-only mode when inspecting. The API does not currently provide a `GET /alerts` endpoint.

The most recent tree is `anchors\merkle_tree_latest.html`; each checkpoint also gets a numbered HTML file and JSON anchor manifest in `anchors`. Open the latest tree manually or regenerate its HTML from the most recent checkpoint:

```powershell
.\.venv\Scripts\python.exe scripts\visualize_latest.py
Invoke-Item .\anchors\merkle_tree_latest.html
```

Click a leaf in the HTML to highlight its path and sibling hashes. The HTML explains the tree; the signed checkpoint and proof verification provide the integrity evidence.

## Suggested presentation order

1. Run `live_capture.py` and make local HTTP requests to show packet capture and flow extraction.
2. Show `data\nids_model.metrics.json` and explain that classifier alerts depend on the model.
3. Insert a labelled manual alert through the API for a reliable encrypted-storage example, then checkpoint and verify it.
4. Open the latest Merkle tree and show a leaf's path to the root.
5. Run `proof_demo.py` for valid and forged inclusion proofs.
6. Run `tamper_demo.py` for altered ciphertext and rejected retrieval.

Architecture: live packets → flow features → model prediction → AES-GCM encrypted alert → SQLite → SHA-256 Merkle root → RSA-PSS signed checkpoint. The store also creates a checkpoint automatically every 100 alerts.
