# 🚀 Azure VM Provisioning with Nginx Web Server
### *Enterprise Infrastructure Deployment via Azure CLI — Nautilus DevOps Project*

---

[![Azure](https://img.shields.io/badge/Azure-Cloud-0089D6?style=flat-square&logo=microsoft-azure)](https://azure.microsoft.com)

[![Nginx](https://img.shields.io/badge/Nginx-Web_Server-009639?style=flat-square&logo=nginx)](https://nginx.org)

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?style=flat-square&logo=ubuntu)](https://ubuntu.com)

[![CLI](https://img.shields.io/badge/Azure_CLI-Automated-0078D4?style=flat-square)](https://docs.microsoft.com/cli/azure)

[![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=flat-square)]()

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Verification](#environment-verification)
- [Deployment Guide](#deployment-guide)
  - [Phase 1: Network Security Group](#phase-1-network-security-group-setup)
  - [Phase 2: VM Provisioning](#phase-2-vm-provisioning)
  - [Phase 3: Port Exposure](#phase-3-port-exposure)
  - [Phase 4: Verification](#phase-4-verification--validation)
- [Known Issues & Resolutions](#-known-issues--resolutions)
- [Final Validation](#-final-validation)
- [Lessons Learned](#-lessons-learned)

---

## 📌 Project Overview

This repository documents the end-to-end provisioning of a production-grade **Azure Virtual Machine** configured as an **Nginx web server** for the **Nautilus DevOps infrastructure project**. The deployment is fully automated via **Azure CLI** and follows enterprise cloud security standards.

### 🎯 Objectives

| Objective | Specification |
|-----------|--------------|
| **VM Name** | `datacenter-vm` |
| **Base Image** | Ubuntu 22.04 LTS |
| **Web Server** | Nginx (auto-installed via custom user data) |
| **Traffic** | HTTP port `80` open from Internet |
| **Region** | `East US` |
| **Automation** | Azure CLI + shell bootstrap script |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│               Azure Resource Group                   │
│         kml_rg_main-547bf0c136014089 (East US)       │
│                                                      │
│   ┌─────────────┐       ┌──────────────────────┐    │
│   │  datacenter │       │   datacenter-nsg     │    │
│   │     -nsg    │──────▶│  Allow-HTTP Rule     │    │
│   └─────────────┘       │  Port 80 / Internet  │    │
│                         └──────────┬───────────┘    │
│                                    │                 │
│                         ┌──────────▼───────────┐    │
│                         │    datacenter-vm      │    │
│                         │   Ubuntu 22.04 LTS    │    │
│                         │   Standard_B1s        │    │
│                         │   Standard_LRS 64GB   │    │
│                         │   Public IP:          │    │
│                         │   20.127.103.60       │    │
│                         │   ┌───────────────┐   │    │
│                         │   │  Nginx Server │   │    │
│                         │   │  Port :80     │   │    │
│                         │   └───────────────┘   │    │
│                         └──────────────────────┘    │
└─────────────────────────────────────────────────────┘
              ▲
              │  HTTP Request
         [ Internet ]
```

---

## ✅ Prerequisites

- Azure CLI installed (`az --version`)
- Active Azure subscription with **Contributor** access
- Bash shell environment
- Azure Free Labs or equivalent environment access

---

## 🔍 Environment Verification

Before deployment, confirm your Azure subscription and resource group are active.

```bash
az account show
az group list --output table
```

***📸 Screenshot Placeholder — `az account show` output showing Enabled subscription and servicePrincipal user***
<img width="1037" height="604" alt="image" src="https://github.com/user-attachments/assets/106dd335-c7ce-4808-9bd5-cf2a2e19435f" />

> **Expected:** `"state": "Enabled"` with an existing resource group in `eastus`

---

## 🚀 Deployment Guide

### Phase 1: Network Security Group Setup

Create the NSG and attach an inbound rule to allow HTTP traffic on port 80.

```bash
# Create NSG
az network nsg create \
  --resource-group kml_rg_main-547bf0c136014089 \
  --name datacenter-nsg

# Add HTTP inbound rule
az network nsg rule create \
  --resource-group kml_rg_main-547bf0c136014089 \
  --nsg-name datacenter-nsg \
  --name Allow-HTTP \
  --protocol Tcp \
  --direction Inbound \
  --priority 100 \
  --source-address-prefix Internet \
  --destination-port-range 80 \
  --access Allow
```

***📸 Screenshot Placeholder — NSG creation success JSON showing `"provisioningState": "Succeeded"` and Allow-HTTP rule***

---

### Phase 2: VM Provisioning

Create the Nginx bootstrap script, then deploy the VM with compliant disk configuration.

```bash
# Step 1: Create user data script
cat <<'EOF' > nginx-init.sh
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF
```

```bash
# Step 2: Deploy VM
az vm create \
  --resource-group kml_rg_main-547bf0c136014089 \
  --name datacenter-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --nsg datacenter-nsg \
  --public-ip-sku Standard \
  --custom-data nginx-init.sh \
  --os-disk-size-gb 64 \
  --storage-sku Standard_LRS \
  --location eastus
```

***📸 Screenshot Placeholder — Successful VM creation JSON output showing `"powerState": "VM running"` and assigned `publicIpAddress`***

---

### Phase 3: Port Exposure

```bash
az vm open-port \
  --resource-group kml_rg_main-547bf0c136014089 \
  --name datacenter-vm \
  --port 80
```
***📸 Screenshot***

<img width="1037" height="789" alt="image" src="https://github.com/user-attachments/assets/515d0061-997f-4c7c-a982-efd49b62ea00" />

<img width="1035" height="857" alt="image" src="https://github.com/user-attachments/assets/efd3ceab-19f5-46b5-8d05-556ea3b5f426" />

<img width="1034" height="868" alt="image" src="https://github.com/user-attachments/assets/8b63b4c3-0713-40b9-885e-1f7995018eb9" />

<img width="1038" height="869" alt="image" src="https://github.com/user-attachments/assets/89b19caa-2ec4-45ea-b9fb-744dd50612a3" />

<img width="1038" height="863" alt="image" src="https://github.com/user-attachments/assets/72a3c5d7-5259-46ce-92b6-1ee2a7d8ddcd" />

---

### Phase 4: Verification & Validation

```bash
# Retrieve public IP
az vm list-ip-addresses \
  --resource-group kml_rg_main-547bf0c136014089 \
  --name datacenter-vm \
  --output table

# Verify Nginx is responding (wait ~2 mins after VM creation)
curl http://20.127.103.60
```

***📸 Screenshot Placeholder — `curl` response showing full Nginx HTML page with `<h1>Welcome to nginx!</h1>`***

---

## ⚠️ Known Issues & Resolutions

This section documents all blockers encountered during deployment and their exact resolutions — critical for team knowledge transfer and future re-deployments.

---

### ❌ Issue 1: Policy Blocked Default Disk Configuration

**Error Code:** `RequestDisallowedByPolicy`

```
Resource 'datacenter-vm_disk1_...' was disallowed by policy.
Reasons: 'This storage configuration is not allowed.
Ensure the disk size is 128 GB or less and the SKU is not Premium.'
```

**Root Cause:** Azure Free Labs enforces a policy that blocks `Premium_LRS` (default managed disk SKU) and disk sizes over 128 GB.

**Resolution:** Explicitly declare a compliant disk configuration using:

```bash
--os-disk-size-gb 64 \
--storage-sku Standard_LRS
```

***📸 Screenshot Placeholder — Terminal showing `RequestDisallowedByPolicy` error on first VM create attempt***

---

### ❌ Issue 2: Invalid CLI Flag `--os-disk-storage-account-type`

**Error:** `unrecognized arguments: --os-disk-storage-account-type Standard_LRS`

**Root Cause:** `--os-disk-storage-account-type` is not a valid `az vm create` parameter. The correct flag for controlling managed disk SKU during VM creation is `--storage-sku`.

**Resolution:** Replace with:
```bash
--storage-sku Standard_LRS   ✅
```

---

### ❌ Issue 3: VM Stuck in `Failed` State — Cannot Recreate

**Error Code:** `OperationNotAllowed`

```
Operation 'Update VM' is not allowed on VM 'datacenter-vm'
since the VM is marked for deletion.
```

**Root Cause:** A previous failed deployment left the VM in a terminal `Failed` provisioning state. Azure prevents new deployments with the same name until cleanup is complete.

**Resolution — 3-step cleanup:**

```bash
# Step 1: Force delete the failed VM
az vm delete \
  --resource-group kml_rg_main-547bf0c136014089 \
  --name datacenter-vm \
  --force-deletion true \
  --yes

# Step 2: Confirm deletion (expect ResourceNotFound)
az vm show \
  --resource-group kml_rg_main-547bf0c136014089 \
  --name datacenter-vm \
  --query "provisioningState" 2>&1

# Step 3: Remove orphaned disks
az disk list \
  --resource-group kml_rg_main-547bf0c136014089 \
  --query "[].name" -o tsv | xargs -I {} az disk delete \
  --resource-group kml_rg_main-547bf0c136014089 \
  --name {} --yes
```

***📸 Screenshot Placeholder — Terminal showing `ResourceNotFound` after successful force deletion confirming environment is clean***

---

### ❌ Issue 4: Conflicting Orphaned Disk From Previous Failed Deployment

**Error Code:** `OperationNotAllowed` on `osDisk.managedDisk.storageAccountType`

**Root Cause:** A ghost disk from a prior failed deployment (`pps-vm-01_...`) was still registered in the subscription, causing a conflict when attempting to change the managed disk SKU.

**Resolution:** The `--force-deletion` cleanup in Issue 3 cleared the orphaned disk, allowing the subsequent deploy with `--storage-sku Standard_LRS` to succeed.

---

## 🏁 Final Validation

| Check | Command | Expected Result |
|-------|---------|----------------|
| VM Running | `az vm show ... --query powerState` | `"VM running"` |
| Port 80 Open | `az network nsg rule list ...` | Allow-HTTP rule present |
| Nginx Live | `curl http://20.127.103.60` | `Welcome to nginx!` HTML |

***📸 Screenshot Placeholder — Final `curl http://20.127.103.60` output showing complete Nginx HTML response in terminal***

<img width="1030" height="563" alt="image" src="https://github.com/user-attachments/assets/78422bad-e5e5-40ec-b76a-063564f42149" />

---

## 📚 Lessons Learned

| # | Lesson | Impact |
|---|--------|--------|
| 1 | Always specify `--storage-sku Standard_LRS` and `--os-disk-size-gb` in lab/restricted environments | Prevents policy-block failures |
| 2 | Use `--force-deletion true` when a VM is stuck in `Failed` state | Bypasses standard deletion lock |
| 3 | Verify `ResourceNotFound` before re-deploying with same VM name | Prevents `OperationNotAllowed` timing errors |
| 4 | Run `az disk list` and purge orphaned disks before retry | Eliminates ghost resource conflicts |
| 5 | Wait ~2 minutes after VM creation before testing Nginx | `--custom-data` runs asynchronously post-boot |

---






<img width="1028" height="856" alt="image" src="https://github.com/user-attachments/assets/d5129781-96a4-4531-bda3-237a78362414" />

<img width="1037" height="856" alt="image" src="https://github.com/user-attachments/assets/5afccb76-6d63-40c3-a955-c6e47c678539" />

<img width="1038" height="868" alt="image" src="https://github.com/user-attachments/assets/dd796fad-1993-4c08-8182-ca2dd4a20ce5" />

<img width="1038" height="874" alt="image" src="https://github.com/user-attachments/assets/70173bd9-0438-4959-bf52-adf455288a9d" />

<img width="1038" height="644" alt="image" src="https://github.com/user-attachments/assets/3f99511b-1bfe-4df2-a6d1-61b055369fee" />

<img width="1032" height="804" alt="image" src="https://github.com/user-attachments/assets/1d0c5cae-a05f-4d1c-a8da-6855d36f8cfb" />

<img width="1037" height="526" alt="image" src="https://github.com/user-attachments/assets/15c982a6-5566-48ea-81e4-2ea4709a245f" />

<img width="1031" height="405" alt="image" src="https://github.com/user-attachments/assets/810a54e9-23c3-447d-b29a-5ac2542447d7" />

<img width="1039" height="455" alt="image" src="https://github.com/user-attachments/assets/38d12eab-dfee-47f2-a9fb-f35cc329529c" />


<img width="1029" height="286" alt="image" src="https://github.com/user-attachments/assets/27b8ec0b-3464-4cc0-9b30-f0b5bd6ccd56" />

<img width="1036" height="250" alt="image" src="https://github.com/user-attachments/assets/1e4f3141-e544-4c1d-aafd-acba0cc5d8be" />


<img width="1042" height="418" alt="image" src="https://github.com/user-attachments/assets/8a21fccf-5c55-49f5-a743-85227406e94d" />

<img width="1032" height="464" alt="image" src="https://github.com/user-attachments/assets/fa2d3a5a-0ca3-423a-8e86-a0eb01cfc531" />

<img width="1035" height="531" alt="image" src="https://github.com/user-attachments/assets/e0875294-29ed-475b-8ab2-718ce029a7e4" />


<img width="1031" height="734" alt="image" src="https://github.com/user-attachments/assets/af9c9de3-233c-47d7-8198-1d5f7b0cd38d" />







