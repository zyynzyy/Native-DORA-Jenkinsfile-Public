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

How the Jenkinsfile Measurement Mechanism Works
The pipeline follows an event-driven workflow triggered by a GitHub Webhook on every git push and deploys the static web application to an Nginx server (/var/www/dora-site):

GitHub Push (Webhook) ──▶ Checkout ──▶ Semgrep SAST ──▶ Build & Deploy (Nginx) ──▶ Native DORA Calculation

1. Environment Configuration
The Jenkinsfile defines core paths and parameters at the pipeline level without external DORA plugins:

APP_DIR: Target Nginx web root (/var/www/dora-site).

DORA_LOG: Persistent local CSV log path (/var/lib/jenkins/dora-metrics/deployments.csv).

DORA_WINDOW_DAYS: Observation window set to 30 days for Deployment Frequency calculation.

2. Capturing Commit Timestamp (T_commit)
During the Capture Source Info stage, the pipeline extracts the exact Unix Epoch timestamp (UTC seconds) of the latest commit directly from Git metadata:

git log -1 --format=%ct
