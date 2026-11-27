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

Markdown---

## How the `Jenkinsfile` Measurement Mechanism Works

The pipeline follows an event-driven workflow triggered by a **GitHub Webhook** on every `git push` and deploys the static web application to an **Nginx** server (`/var/www/dora-site`):

```text
GitHub Push (Webhook) ──▶ Checkout ──▶ Semgrep SAST ──▶ Build & Deploy (Nginx) ──▶ Native DORA Calculation
1. Environment ConfigurationThe Jenkinsfile defines core paths and parameters at the pipeline level without external DORA plugins:APP_DIR: Target Nginx web root (/var/www/dora-site).DORA_LOG: Persistent local CSV log path (/var/lib/jenkins/dora-metrics/deployments.csv).DORA_WINDOW_DAYS: Observation window set to 30 days for Deployment Frequency calculation.2. Capturing Commit Timestamp (T_commit)During the Capture Source Info stage, the pipeline extracts the exact Unix Epoch timestamp (UTC seconds) of the latest commit directly from Git metadata:Bashgit log -1 --format=%ct
This value is stored as env.GIT_COMMIT_EPOCH to serve as the starting boundary (T_commit) for Lead Time calculation.3. Static Security Analysis & DeploymentSemgrep SAST: Scans source files using Semgrep CLI, exporting semgrep-report.json and recording the security status (OK, ISSUES, or FAILED).Build & Deploy: Packages the static HTML/CSS/JS assets and deploys them into the Nginx production directory.4. Native DORA Metrics Calculation (stage('DORA Metrics'))Immediately after deployment succeeds, an embedded shell and Groovy routine computes both velocity metrics:Lead Time for Changes (LT):
Captures the deployment completion timestamp via date +%s (T_deploy) and computes the elapsed duration:PlaintextLT = T_deploy - T_commit
The result is stored in seconds (lt_seconds) and formatted into human-readable minutes and seconds (Xm Ys).Persistent CSV Logging (deployments.csv):
Appends a structured record for every build using printf:Code snippetbuild_number,commit,commit_epoch,deploy_epoch,lt_seconds,status,semgrep_status
Rolling 30-Day Deployment Frequency (DF):
Filters deployments.csv using awk to count all successful deployments (SUCCESS or SUCCESS_WITH_ISSUES) whose timestamps fall within the last 30 days (WINDOW_START), then divides by 30:PlaintextDF = N_deploy (30 days) / 30
Structured JSON Artifact (dora-metrics.json):
Uses groovy.json.JsonOutput to serialize the metrics into an archived artifact and updates currentBuild.description directly on the Jenkins dashboard:JSON{
  "buildNumber": "26",
  "commit": "38ddd6b",
  "leadTimeSeconds": "113",
  "leadTimeFormatted": "1m 53s",
  "deployCountLast30Days": "26",
  "deployFrequencyPerDay": "0.8667",
  "semgrepStatus": "OK"
}
Experimental Evaluation SummaryEvaluated across 50 automated deployment cycles (25 minor modifications and 25 major modifications) on an isolated Ubuntu 24.04 LTS virtual instance (4 vCPU, 8 GB RAM) running Jenkins v2.555.1, Semgrep CLI v1.61.0, and Nginx v1.24.0:Metric / ResourceMeanMedianMinMaxNotesPipeline Success Rate100%———50 / 50 builds completed (SUCCESS)Automated Lead Time (LT)30.76 s27.50 s24 s113 sMinor median: 27 s; Major median: 28 s (p = 0.891)Manual Lead Time (Stopwatch)33.32 s—24 s117 sMean difference: 2.56 s (webhook RTT & UI render boundary)Deployment Frequency (DF)1.67 / day———Calculated over a 30-day rolling window (50 builds)CPU Utilization20.05%—11.70%34.27%Lightweight execution without external agentsRAM Utilization12.88%—11.14%13.58%Stable footprint with no memory leaks observedNetwork Ping Latency31.47 ms—28.10 ms99.90 msMax spike occurred during VM cold start (Build #26)Quick Adaptation GuidePrerequisites on Jenkins Host/Agent:Jenkins with Pipeline plugin enabledgit, bash, awk, and semgrep CLI installedWrite permissions to your web server root (e.g., /var/www/dora-site) and metrics log directory (e.g., /var/lib/jenkins/dora-metrics)Configure Variables in Jenkinsfile:
Update APP_DIR, DORA_LOG, DORA_WINDOW_DAYS, and SEMGREP_BIN in the environment block to match your server paths.Connect GitHub Webhook:
Point your repository webhook (application/json, push events) to http://<your-jenkins-url>/github-webhook/.License & CitationThis repository is shared openly for academic and practical DevOps reference. If you adapt this mechanism in your research or CI/CD workflows, please cite the accompanying thesis:Code snippet@mastersthesis{sallum2026dora,
  author  = {Zayyan Abdul Kadir Sallum},
  title   = {Evaluasi Otomatisasi Pengukuran DORA Metrics melalui Modifikasi Jenkinsfile pada Pipeline CI/CD},
  school  = {Universitas Diponegoro},
  year    = {2026},
  address = {Semarang, Indonesia}
}
