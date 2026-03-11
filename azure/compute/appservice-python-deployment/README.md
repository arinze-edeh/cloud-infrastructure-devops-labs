# Deploying a Python Web Application on Azure App Service

[![Azure](https://img.shields.io/badge/Azure-App%20Service-0078D4?style=flat-square&logo=microsoft-azure)](https://azure.microsoft.com)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python)](https://python.org)
[![Linux](https://img.shields.io/badge/OS-Linux-FCC624?style=flat-square&logo=linux)](https://linux.org)
[![SKU](https://img.shields.io/badge/SKU-Basic%20B1-green?style=flat-square)](https://azure.microsoft.com/en-us/pricing/details/app-service/)
[![Status](https://img.shields.io/badge/App%20State-Running-brightgreen?style=flat-square)]()

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Solution Walkthrough](#solution-walkthrough)
  - [Step 1: Verify Resource Group](#step-1-verify-resource-group)
  - [Step 2: Create App Service Plan](#step-2-create-app-service-plan)
  - [Step 3: Create the Web App](#step-3-create-the-web-app)
  - [Step 4: Verify Deployment](#step-4-verify-deployment)
- [Configuration Reference](#configuration-reference)
- [Screenshots](#screenshots)
- [Validation and Testing](#validation-and-testing)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Cost Implications](#cost-implications)

---

## Overview

This runbook documents the end-to-end process of provisioning a production-grade Python web application on **Azure App Service** using the Azure CLI (`az`). The deployment targets a Linux-based App Service Plan in the **West US** region and follows infrastructure-as-code principles to ensure repeatability, traceability, and operational excellence.

This guide is intended for DevOps engineers, cloud platform engineers, and SREs responsible for provisioning Azure-hosted web workloads.

---

## Problem Statement

### Context

The Nautilus DevOps team required a cloud-hosted, scalable runtime environment for a Python-based web application. The solution needed to satisfy the following non-negotiable requirements:

| Requirement | Specification |
|---|---|
| Web App Name | `nautilus-webapp` |
| Region | West US |
| Resource Group | Default (pre-existing) |
| Publish Mode | Code |
| Runtime Stack | Python 3.11 on Linux |
| App Service Plan | `nautilus-learn-python` |
| SKU | Basic B1 |
| Application Insights | Disabled |
| Tags | `Name=WebAppLearning`, `Environment=Dev` |
| Post-Deployment State | Running |

### Challenges Addressed

**1. Runtime Environment Compatibility**
Ensuring the correct Python version (3.11) was paired with a Linux-based App Service Plan, since Python runtime on Azure App Service is only supported on Linux workers.

**2. Resource Isolation via Tagging**
Applying consistent resource tags at provisioning time to enable cost attribution, environment filtering, and governance compliance across the subscription.

**3. App Service Plan Scoping**
Creating a dedicated App Service Plan scoped to the correct region and SKU, independent of any existing plans, to avoid resource contention.

**4. Validation Without Portal Dependency**
Confirming the app reached `Running` state entirely via CLI, without reliance on the Azure Portal, enabling pipeline-friendly verification.

---

## Architecture

```
Azure Subscription (f0c3bcdd-5ce2-4fa0-8cf3-41559747512b)
|
+-- Resource Group: kml_rg_main-076cef2f5c5143c5 (East US)
    |
    +-- App Service Plan: nautilus-learn-python
    |     Region   : West US
    |     OS       : Linux
    |     SKU      : Basic B1
    |     Workers  : 1
    |
    +-- Web App: nautilus-webapp
          Runtime  : PYTHON|3.11
          State    : Running
          Hostname : nautilus-webapp.azurewebsites.net
          Tags     : Name=WebAppLearning, Environment=Dev
```

> **Note:** The resource group resides in East US (pre-existing), while the App Service Plan and Web App are provisioned in West US per the task specification. Azure supports this cross-region configuration.

---

## Prerequisites

Before executing the commands in this guide, ensure the following are in place:

### Tooling

```bash
# Verify Azure CLI installation
az --version

# Verify active login session
az account show
```

### Access Requirements

* An active Azure subscription with Contributor or Owner role on the target resource group
* Azure CLI version 2.40.0 or later
* Bash or Zsh shell environment (Linux, macOS, or WSL2 on Windows)

### Required Resource

* A pre-existing resource group (this guide uses `kml_rg_main-076cef2f5c5143c5`)

---

## Solution Walkthrough

### Step 1: Verify Resource Group

Before provisioning any resources, confirm the target resource group exists and its provisioning state is `Succeeded`.

```bash
az group list --output table
```

**Expected Output:**

```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-076cef2f5c5143c5  eastus      Succeeded
```

**What this confirms:**
* The resource group is active and available
* No pre-flight errors will block downstream resource creation

> ### Screenshot -- Step 1: Resource Group Verification

<img width="1031" height="331" alt="image" src="https://github.com/user-attachments/assets/36db9c4c-7a52-475c-b690-ec615f481568" />

---

### Step 2: Create App Service Plan

The App Service Plan defines the compute tier, operating system, and region for all web apps hosted within it. A dedicated plan is created here to ensure isolation from other workloads.

```bash
az appservice plan create \
  --name "nautilus-learn-python" \
  --resource-group "kml_rg_main-076cef2f5c5143c5" \
  --location "westus" \
  --sku B1 \
  --is-linux
```

**Flag Breakdown:**

| Flag | Value | Purpose |
|---|---|---|
| `--name` | `nautilus-learn-python` | Unique identifier for the plan |
| `--resource-group` | `kml_rg_main-...` | Target resource group |
| `--location` | `westus` | Deployment region |
| `--sku` | `B1` | Basic tier, 1 core, 1.75 GB RAM |
| `--is-linux` | (flag) | Required for Python runtime support |

**Successful Response Indicators:**

```json
{
  "provisioningState": "Succeeded",
  "reserved": true,
  "kind": "linux",
  "sku": {
    "name": "B1",
    "tier": "Basic"
  }
}
```

> `"reserved": true` confirms this is a Linux-based plan. This is critical since Python runtimes on Azure App Service require Linux workers.

> ### Screenshots -- Step 2: App Service Plan Creation
<img width="1034" height="843" alt="image" src="https://github.com/user-attachments/assets/25de35b1-0b18-44a9-b574-79480c3dd274" />
<img width="1035" height="865" alt="image" src="https://github.com/user-attachments/assets/085a84c2-36cc-4055-a7c4-c77a5ae90b35" />

---

### Step 3: Create the Web App

With the App Service Plan provisioned, the web app is created and bound to it. Tags are applied at creation time for governance alignment.

```bash
az webapp create \
  --name "nautilus-webapp" \
  --resource-group "kml_rg_main-076cef2f5c5143c5" \
  --plan "nautilus-learn-python" \
  --runtime "PYTHON:3.11" \
  --tags Name="WebAppLearning" Environment="Dev"
```

**Flag Breakdown:**

| Flag | Value | Purpose |
|---|---|---|
| `--name` | `nautilus-webapp` | Globally unique web app name |
| `--resource-group` | `kml_rg_main-...` | Scopes the resource to the correct group |
| `--plan` | `nautilus-learn-python` | Binds app to the previously created plan |
| `--runtime` | `PYTHON:3.11` | Sets Python 3.11 as the application runtime |
| `--tags` | `Name=... Environment=...` | Applies resource tags for cost and governance |

**Successful Response Indicators:**

```json
{
  "state": "Running",
  "defaultHostName": "nautilus-webapp.azurewebsites.net",
  "kind": "app,linux",
  "tags": {
    "Environment": "Dev",
    "Name": "WebAppLearning"
  }
}
```

> ### Screenshot -- Step 3: Web App Creation
> **[SCREENSHOT PLACEHOLDER -- Step 3 of 4]**
>
> **What to capture:** Terminal window showing the JSON response from `az webapp create`
>
> **Key fields visible in screenshot:**
> * `"state": "Running"`
> * `"defaultHostName": "nautilus-webapp.azurewebsites.net"`
> * `"kind": "app,linux"`
> * `"tags": { "Environment": "Dev", "Name": "WebAppLearning" }`
> * `"location": "West US"`
>
> *Replace this block with your actual screenshot. Add it as: `![Step 3 - Web App Created](./screenshots/step3-webapp-create.png)`*

---

### Step 4: Verify Deployment

A targeted `az webapp show` query is used to confirm all specifications were applied correctly, without parsing the full JSON payload.

```bash
az webapp show \
  --name "nautilus-webapp" \
  --resource-group "kml_rg_main-076cef2f5c5143c5" \
  --query "{WebAppName:name, State:state, Region:location, OS:kind, Runtime:siteConfig.linuxFxVersion, Tags:tags, Plan:appServicePlanId}" \
  --output json
```

**Verified Output:**

```json
{
  "OS": "app,linux",
  "Plan": "/subscriptions/.../serverfarms/nautilus-learn-python",
  "Region": "West US",
  "Runtime": "PYTHON|3.11",
  "State": "Running",
  "Tags": {
    "Environment": "Dev",
    "Name": "WebAppLearning"
  },
  "WebAppName": "nautilus-webapp"
}
```

**Verification Checklist:**

- [x] `State` is `Running`
- [x] `Runtime` is `PYTHON|3.11`
- [x] `Region` is `West US`
- [x] `OS` confirms Linux (`app,linux`)
- [x] Both tags present and correct
- [x] Plan name matches `nautilus-learn-python`

> ### Screenshot -- Step 4: Deployment Verification
<img width="1026" height="855" alt="image" src="https://github.com/user-attachments/assets/76672fae-dac6-4a39-83c7-f0ada7cf946b" />

---

> ### Screenshot -- Portal Confirmation (Optional)
> **[SCREENSHOT PLACEHOLDER -- Portal View]**
>
> **What to capture:** Azure Portal overview blade for `nautilus-webapp`
>
> **Key elements visible in screenshot:**
> * App status badge showing `Running`
> * URL: `nautilus-webapp.azurewebsites.net`
> * Location: `West US`
> * App Service Plan: `nautilus-learn-python (B1)`
> * Runtime stack: `Python 3.11`
> * Tags panel: `Environment: Dev`, `Name: WebAppLearning`
>
> *Navigate to: portal.azure.com > Resource Groups > kml_rg_main-076cef2f5c5143c5 > nautilus-webapp*
>
> *Replace this block with your actual screenshot. Add it as: `![Portal - Web App Overview](./screenshots/portal-webapp-overview.png)`*

---

## Screenshots

All screenshots should be stored in the `./screenshots/` directory at the root of this repository. The table below maps each screenshot file to its corresponding deployment step.

| # | File | Step | Key Evidence |
|---|---|---|---|
| 1 | `step1-resource-group.png` | Verify Resource Group | `Status: Succeeded` |
| 2 | `step2-appservice-plan.png` | Create App Service Plan | `provisioningState: Succeeded`, `kind: linux` |
| 3 | `step3-webapp-create.png` | Create Web App | `state: Running`, tags applied |
| 4 | `step4-webapp-verify.png` | Verify Deployment | All 7 spec fields confirmed |
| 5 | `portal-webapp-overview.png` | Portal Confirmation | Visual running state |

> **To add screenshots:** Create a `screenshots/` folder in the repository root, place each PNG file inside it, then replace the placeholder blocks in each step above with the corresponding `![alt text](./screenshots/filename.png)` markdown image tag.

### Web App Summary

| Property | Value |
|---|---|
| Web App Name | `nautilus-webapp` |
| Default Hostname | `nautilus-webapp.azurewebsites.net` |
| SCM Endpoint | `nautilus-webapp.scm.azurewebsites.net` |
| Runtime | `PYTHON|3.11` |
| OS | Linux |
| State | Running |
| Region | West US |
| App Service Plan | `nautilus-learn-python` |
| SKU | Basic B1 |
| Resource Group | `kml_rg_main-076cef2f5c5143c5` |
| Subscription ID | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Application Insights | Disabled |
| HTTPS Only | false (configure post-deploy) |
| Always On | false (not supported on Basic tier) |

### Tags Applied

| Key | Value |
|---|---|
| `Name` | `WebAppLearning` |
| `Environment` | `Dev` |

---

## Validation and Testing

### CLI Health Check

```bash
# Confirm app is reachable
curl -I https://nautilus-webapp.azurewebsites.net

# Expected: HTTP/1.1 200 OK or 403 (default Azure page before app code is deployed)
```

### Runtime Verification

```bash
az webapp config show \
  --name "nautilus-webapp" \
  --resource-group "kml_rg_main-076cef2f5c5143c5" \
  --query "linuxFxVersion"

# Expected: "PYTHON|3.11"
```

### Tag Audit

```bash
az resource show \
  --name "nautilus-webapp" \
  --resource-group "kml_rg_main-076cef2f5c5143c5" \
  --resource-type "Microsoft.Web/sites" \
  --query "tags"

# Expected: {"Environment": "Dev", "Name": "WebAppLearning"}
```

---

## Troubleshooting

### Web App Not in Running State

**Symptom:** `az webapp show` returns `"State": "Stopped"` or `"State": "Starting"`

**Resolution:**
```bash
az webapp start \
  --name "nautilus-webapp" \
  --resource-group "kml_rg_main-076cef2f5c5143c5"
```

---

### Runtime Mismatch

**Symptom:** `linuxFxVersion` returns empty string or incorrect runtime after creation.

**Root Cause:** Azure CLI may not immediately reflect runtime in `siteConfig` during creation. The runtime propagates asynchronously.

**Resolution:**
```bash
az webapp config set \
  --name "nautilus-webapp" \
  --resource-group "kml_rg_main-076cef2f5c5143c5" \
  --linux-fx-version "PYTHON|3.11"
```

---

### App Service Plan Already Exists

**Symptom:** `az appservice plan create` returns a conflict error.

**Resolution:** Check existing plans before creation.
```bash
az appservice plan list \
  --resource-group "kml_rg_main-076cef2f5c5143c5" \
  --output table
```

---

### Readonly Attribute Warning

**Symptom:** CLI outputs `Readonly attribute name will be ignored in class AppServicePlan`

**Root Cause:** Known Azure CLI cosmetic warning for App Service Plan creation. Does not affect provisioning.

**Resolution:** No action required. Confirm `provisioningState: Succeeded` in the response body.

---

## Security Considerations

> The following hardening steps are recommended for production promotions beyond the Dev environment.

* **Enable HTTPS Only:** Enforce TLS for all inbound traffic.
  ```bash
  az webapp update \
    --name "nautilus-webapp" \
    --resource-group "kml_rg_main-076cef2f5c5143c5" \
    --https-only true
  ```

* **Restrict SCM Access:** The SCM endpoint (`*.scm.azurewebsites.net`) should be IP-restricted or disabled when not in active use.

* **Disable FTP Publishing:** FTP is enabled by default. Enforce FTPS-only or disable entirely.
  ```bash
  az webapp config set \
    --name "nautilus-webapp" \
    --resource-group "kml_rg_main-076cef2f5c5143c5" \
    --ftps-state Disabled
  ```

* **Rotate Publish Credentials:** Publishing credentials should be rotated after initial deployment.

* **Enable Managed Identity:** For downstream Azure service access (Key Vault, Storage), assign a system-assigned managed identity rather than using connection strings.

---

## Cost Implications

| Resource | SKU | Estimated Monthly Cost (USD) |
|---|---|---|
| App Service Plan (B1) | Basic | ~$13.14 |
| Web App | N/A (billed via plan) | $0 additional |
| Outbound Data Transfer | Pay-per-use | Variable |

> Costs are based on West US region pricing as of Q1 2026. Always validate against the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/) for current rates.

**To avoid unintended charges in non-production use:**
```bash
# Stop the app when not in use
az webapp stop \
  --name "nautilus-webapp" \
  --resource-group "kml_rg_main-076cef2f5c5143c5"
```

---




<img width="1033" height="844" alt="image" src="https://github.com/user-attachments/assets/f9d23c61-4bd0-4922-8d7e-add9af2909b2" />
<img width="1037" height="858" alt="image" src="https://github.com/user-attachments/assets/05a154f3-4509-4af6-92db-488cc8947fcc" />

