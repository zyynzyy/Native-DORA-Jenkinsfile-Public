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

---

## Experimental Evaluation Summary

Evaluated across **50 automated deployment cycles** (25 minor modifications and 25 major modifications) on an isolated Ubuntu 24.04 LTS virtual instance (4 vCPU, 8 GB RAM) running Jenkins v2.555.1, Semgrep CLI v1.61.0, and Nginx v1.24.0:

| Metric / Resource | Mean | Median | Min | Max | Notes |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **Pipeline Success Rate** | 100% | — | — | — | 50 / 50 builds completed (`SUCCESS`) |
| **Automated Lead Time (LT)** | 30.76 s | 27.50 s | 24 s | 113 s | Minor median: 27 s; Major median: 28 s (p = 0.891) |
| **Manual Lead Time (Stopwatch)** | 33.32 s | — | 24 s | 117 s | Mean difference: 2.56 s (webhook RTT & UI render boundary) |
| **Deployment Frequency (DF)** | 1.67 / day | — | — | — | Calculated over a 30-day rolling window (50 builds) |
| **CPU Utilization** | 20.05% | — | 11.70% | 34.27% | Lightweight execution without external agents |
| **RAM Utilization** | 12.88% | — | 11.14% | 13.58% | Stable footprint with no memory leaks observed |
| **Network Ping Latency** | 31.47 ms | — | 28.10 ms | 99.90 ms | Max spike occurred during VM cold start (Build #26) |

---

## Quick Adaptation Guide

1. **Prerequisites on Jenkins Host/Agent:**
   * Jenkins with Pipeline plugin enabled
   * `git`, `bash`, `awk`, and `semgrep` CLI installed
   * Write permissions to your web server root (e.g., `/var/www/dora-site`) and metrics log directory (e.g., `/var/lib/jenkins/dora-metrics`)
2. **Configure Variables in `Jenkinsfile`:**
   Update `APP_DIR`, `DORA_LOG`, `DORA_WINDOW_DAYS`, and `SEMGREP_BIN` in the `environment` block to match your server paths.
3. **Connect GitHub Webhook:**
   Point your repository webhook (`application/json`, push events) to `http://<your-jenkins-url>/github-webhook/`.

---

## License & Citation

This repository is shared openly for academic and practical DevOps reference. If you adapt this mechanism in your research or CI/CD workflows, please cite the accompanying thesis:

* **Author:** Zayyan Abdul Kadir Sallum (2026)
* **Title:** *Evaluasi Otomatisasi Pengukuran DORA Metrics melalui Modifikasi Jenkinsfile pada Pipeline CI/CD*
* **Institution:** Master of Information Systems, Universitas Diponegoro, Semarang, Indonesia
