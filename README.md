# Native DORA Metrics Measurement in Jenkinsfile

A lightweight, **plugin-free** implementation for automating **DORA Metrics** (*Lead Time for Changes* and *Deployment Frequency*) natively inside a declarative `Jenkinsfile` within a CI/CD pipeline.

This repository accompanies the Master's Thesis:
> **"Evaluasi Otomatisasi Pengukuran DORA Metrics melalui Modifikasi Jenkinsfile pada Pipeline CI/CD"**  
> *Zayyan Abdul Kadir Sallum — Master of Information Systems, Universitas Diponegoro (2026)*

---

## Overview

Measuring software delivery performance typically relies on third-party Jenkins plugins, external databases, or heavy observability platforms—adding configuration overhead and infrastructure costs, particularly for Small and Medium Enterprises (SMEs).

This project demonstrates how to capture, calculate, and export key DORA velocity metrics directly inside the CI/CD pipeline using standard Linux shell utilities (`git`, `date`, `awk`, `printf`) and Groovy's built-in `JsonOutput`:
* **Lead Time for Changes (LT):** Measured in seconds from the Git commit timestamp (`T_commit`) to successful deployment completion (`T_deploy`).
* **Deployment Frequency (DF):** Calculated dynamically over a **30-day rolling window** based on historical `SUCCESS` builds logged in a local CSV file.

---

## Repository Structure

```text
Native-DORA-Jenkinsfile-Public/
├── Jenkinsfile                   # Declarative CI/CD pipeline with native DORA metrics calculation
├── index.html                    # Sample static web application (main entry point)
├── templates.html                # Secondary application template page
├── templatemo-quantix-style.css  # Application stylesheet
└── templatemo-quantix-script.js  # Application client-side script
```

---

## How the `Jenkinsfile` Measurement Mechanism Works

The pipeline follows an event-driven workflow triggered by a **GitHub Webhook** on every `git push` and deploys the static web application to an **Nginx** server (`/var/www/dora-site`):

`GitHub Push (Webhook)` ➔ `Checkout` ➔ `Semgrep SAST` ➔ `Build & Deploy (Nginx)` ➔ `Native DORA Calculation`

### 1. Environment Configuration
The `Jenkinsfile` defines core paths and parameters at the pipeline level without external DORA plugins:
* **`APP_DIR`**: Target Nginx web root (`/var/www/dora-site`).
* **`DORA_LOG`**: Persistent local CSV log path (`/var/lib/jenkins/dora-metrics/deployments.csv`).
* **`DORA_WINDOW_DAYS`**: Observation window set to `30` days for Deployment Frequency calculation.

### 2. Capturing Commit Timestamp (`T_commit`)
During the **Capture Source Info** stage, the pipeline extracts the exact Unix Epoch timestamp (UTC seconds) of the latest commit directly from Git metadata using `git log -1 --format=%ct`. This value is stored as `env.GIT_COMMIT_EPOCH` to serve as the starting boundary (`T_commit`) for Lead Time calculation.

### 3. Static Security Analysis & Deployment
* **Semgrep SAST:** Scans source files using Semgrep CLI, exporting `semgrep-report.json` and recording the security status (`OK`, `ISSUES`, or `FAILED`).
* **Build & Deploy:** Packages the static HTML/CSS/JS assets and deploys them into the Nginx production directory.

### 4. Native DORA Metrics Calculation (`stage('DORA Metrics')`)
Immediately after deployment succeeds, an embedded shell and Groovy routine computes both velocity metrics:
1. **Lead Time for Changes (LT):** Captures the deployment completion timestamp via `date +%s` (`T_deploy`) and computes `LT = T_deploy - T_commit` in seconds (`lt_seconds`), plus a human-readable format (`Xm Ys`).
2. **Persistent CSV Logging (`deployments.csv`):** Appends a structured record for every build using `printf` with columns: `build_number,commit,commit_epoch,deploy_epoch,lt_seconds,status,semgrep_status`.
3. **Rolling 30-Day Deployment Frequency (DF):** Filters `deployments.csv` using `awk` to count all successful deployments (`SUCCESS` or `SUCCESS_WITH_ISSUES`) within the last 30 days (`WINDOW_START`), then divides by `30` (`DF = N_deploy / 30`).
4. **Structured JSON Artifact (`dora-metrics.json`):** Uses `groovy.json.JsonOutput` to serialize the metrics (`buildNumber`, `commit`, `leadTimeSeconds`, `leadTimeFormatted`, `deployCountLast30Days`, `deployFrequencyPerDay`, and `semgrepStatus`) into an archived artifact and updates `currentBuild.description` on the Jenkins dashboard.

