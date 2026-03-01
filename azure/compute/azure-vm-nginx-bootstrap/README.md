 ☁️ Azure VM Nginx Web Server Deployment

> **Enterprise-Grade Infrastructure Provisioning on Microsoft Azure**
> A production-ready Ubuntu web server deployed via Azure CLI with automated Nginx configuration, network security hardening, and live HTTP validation.

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Business Context](#-business-context)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Environment Configuration](#-environment-configuration)
- [Step-by-Step Deployment Guide](#-step-by-step-deployment-guide)
- [Validation & Testing](#-validation--testing)
- [Results Summary](#-results-summary)
- [Security Posture](#-security-posture)
- [Key Learnings](#-key-learnings)
- [References](#-references)

---

## 🔭 Project Overview

This repository documents the **end-to-end provisioning and live validation** of an Azure Virtual Machine serving as a public-facing Nginx web server. The deployment was executed entirely through the **Azure CLI** using a service principal identity within a time-boxed lab environment.

| Attribute | Value |
|---|---|
| **VM Name** | `devops-vm` |
| **Region** | East US |
| **OS** | Ubuntu Server 22.04 LTS |
| **Web Server** | Nginx 1.18.0 |
| **VM SKU** | Standard_B1s |
| **OS Disk** | 30 GB · Standard_LRS |
| **Public IP** | `20.51.140.233` (Standard SKU) |
| **Deployment Method** | Azure CLI (`az vm create`) |
| **Config Method** | `--custom-data` (cloud-init script) |

---

## 🏢 Business Context

The **Nautilus DevOps Team** required a reliable, internet-accessible web server as the foundational compute layer for a critical application deployment. This VM represents the first provisioned resource in the Nautilus project's infrastructure rollout.

### Requirements

- Deploy in a **restricted Azure subscription** governed by organizational policy
- VM must be named `devops-vm` and hosted in **East US**
- Nginx must be installed and started **automatically at first boot** (zero-touch)
- Port **80 (HTTP)** must be reachable from the public internet
- Solution must comply with **storage SKU and disk size policies**

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure Subscription                           │
│              Azure Free Labs · f0c3bcdd-5ce2-4fa0               │
│                                                                 │
│  Resource Group: kml_rg_main-b7a58fc8335847fe  (East US)       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Virtual Network                         │   │
│  │   10.0.0.0/24                                           │   │
│  │                                                         │   │
│  │   ┌──────────────────────────────────────────────┐     │   │
│  │   │  devops-vm  (Standard_B1s)                   │     │   │
│  │   │  Ubuntu 22.04 LTS                            │     │   │
│  │   │  Private IP: 10.0.0.4                        │     │   │
│  │   │  MAC: 70-A8-A5-40-80-52                      │     │   │
│  │   │  Nginx 1.18.0 ──► :80                        │     │   │
│  │   └───────────────────┬──────────────────────────┘     │   │
│  │                       │                                 │   │
│  │   ┌───────────────────▼──────────────────────────┐     │   │
│  │   │  NIC: devops-vmVMNic                         │     │   │
│  │   │  NSG: devops-vmNSG                           │     │   │
│  │   │   ├─ Allow SSH  (22)  Priority 1000          │     │   │
│  │   │   └─ Allow HTTP (80)  Priority 1010  ◄───────┼─────┼── Internet
│  │   └──────────────────────────────────────────────┘     │   │
│  │                                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Public IP:  20.51.140.233 (Standard SKU)                      │
│  OS Disk:    devops-vm-osdisk (30 GB · Standard_LRS)           │
│  Tenant ID:  54c1a2d3-d100-453c-9636-3a109eb45552              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Azure CLI | ≥ 2.50 | Primary provisioning tool |
| Bash | ≥ 5.0 | Script execution environment |
| `curl` | Any | HTTP validation |
| Azure Subscription | Active | Target deployment environment |

```bash
# Verify Azure CLI is installed and authenticated
az --version
az account show
```

---

## ⚙️ Environment Configuration

### Step 1: Authenticate & Verify Subscription

```bash
az account show
az account list --output table
```

**Output:**

```json
{
  "environmentName": "AzureCloud",
  "homeTenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "isDefault": true,
  "name": "Azure Free Labs",
  "state": "Enabled",
  "user": {
    "name": "6c6d1731-813d-49a4-bef1-4a59f6ba44ac",
    "type": "servicePrincipal"
  }
}
```

> 📸 **Screenshots: Account Authentication**

<img width="1031" height="575" alt="image" src="https://github.com/user-attachments/assets/8b26b680-f89b-49fb-93f9-73596687a1b7" />
<img width="1032" height="724" alt="image" src="https://github.com/user-attachments/assets/6f5ff40c-4081-4d46-aa46-0aa7fa98fab3" />

---

### Step 2: Discover Resource Group

```bash
az group list --query "[].name" -o tsv
```

**Output:**
```
kml_rg_main-b7a58fc8335847fe
```

> 📸 **Screenshot: Subscription & Resource Group Discovery**

<img width="1034" height="677" alt="image" src="https://github.com/user-attachments/assets/9245ffe8-f517-48eb-b92e-324c96998c9c" />

---

### Step 3: Set Deployment Variables

```bash
RESOURCE_GROUP="kml_rg_main-b7a58fc8335847fe"
VM_NAME="devops-vm"
LOCATION="eastus"
IMAGE="Ubuntu2204"
ADMIN_USER="azureuser"
ADMIN_PASS="AdminPass@1234"
DISK_NAME="devops-vm-osdisk"
VM_SIZE="Standard_B1s"
```

> 📸 **Screenshot: Variable Configuration**

<img width="1029" height="735" alt="image" src="https://github.com/user-attachments/assets/cc81f9e3-421a-4f76-ade8-3cda648b36ec" />

---

## 🚀 Step-by-Step Deployment Guide

### Step 4: Create the Nginx Bootstrap Script

This script runs once at first boot via the Azure `--custom-data` mechanism (cloud-init compatible):

```bash
cat > /tmp/nginx-setup.sh << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF
```

> 📸 **Screenshot: Bootstrap Script Creation**
<img width="1030" height="492" alt="image" src="https://github.com/user-attachments/assets/b46b5d6b-699b-47c5-bd53-093b0d53cbe8" />

---

### Step 5: Provision the Virtual Machine

```bash
az vm create \
  --resource-group  $RESOURCE_GROUP      \
  --name            $VM_NAME             \
  --image           $IMAGE               \
  --location        $LOCATION            \
  --admin-username  $ADMIN_USER          \
  --admin-password  $ADMIN_PASS          \
  --custom-data     /tmp/nginx-setup.sh  \
  --public-ip-sku   Standard             \
  --size            $VM_SIZE             \
  --os-disk-name    $DISK_NAME           \
  --storage-sku     os=Standard_LRS      \
  --output table
```

**Successful output:**

```
ResourceGroup                 PowerState    PublicIpAddress    PrivateIpAddress    MacAddress         Location
----------------------------  ------------  -----------------  ------------------  -----------------  ----------
kml_rg_main-b7a58fc8335847fe  VM running    20.51.140.233      10.0.0.4            70-A8-A5-40-80-52  eastus
```

> 📸 **Screenshot: VM Successfully Provisioned**
<img width="1036" height="407" alt="image" src="https://github.com/user-attachments/assets/15b40892-93de-4aa7-a7c9-6c5eba8ebaea" />

---

### Step 6: Open Port 80 via NSG Rule

```bash
az vm open-port \
  --resource-group  $RESOURCE_GROUP \
  --name            $VM_NAME        \
  --port            80              \
  --priority        1010
```

This creates an explicit inbound NSG rule `open-port-80` permitting all internet traffic on port 80.

**Confirmed NSG security rules:**

| Rule Name | Priority | Port | Protocol | Direction | Access | Source |
|---|---|---|---|---|---|---|
| `default-allow-ssh` | 1000 | 22 | TCP | Inbound | Allow | `*` |
| `open-port-80` | 1010 | 80 | `*` | Inbound | Allow | `*` |

> 📸 **Screenshot: NSG Port 80 Rule Applied**

<img width="1077" height="853" alt="image" src="https://github.com/user-attachments/assets/1ba95ec7-ab36-49c5-bac1-04650293c5bf" />

> 📸 **Screenshot: NSG JSON Confirmation**

<img width="1082" height="872" alt="image" src="https://github.com/user-attachments/assets/c598926c-cd5e-413d-b385-79d0736ce7c8" />

---

## ✅ Validation & Testing

### HTTP Connectivity Test

```bash
curl -I http://20.51.140.233
```

**Live response:**

```http
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 01 Mar 2026 02:06:17 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Sun, 01 Mar 2026 02:05:14 GMT
Connection: keep-alive
ETag: "69a39eda-264"
Accept-Ranges: bytes
```

The `HTTP/1.1 200 OK` response with `Server: nginx/1.18.0 (Ubuntu)` confirms:

- The VM is publicly reachable from the internet
- Nginx was successfully installed and started via cloud-init at boot
- The NSG inbound rule on port 80 is active and functioning

> 📸 **Screenshot: Live HTTP 200 OK Validation**
<img width="1083" height="869" alt="image" src="https://github.com/user-attachments/assets/f5983159-7902-494f-a452-bf84b7d236a8" />

---

## 📊 Results Summary

| Requirement | Target | Actual | Status |
|---|---|---|---|
| VM instance name | `devops-vm` | `devops-vm` | ✅ Pass |
| Operating system | Ubuntu (any) | Ubuntu 22.04 LTS | ✅ Pass |
| Region | East US | `eastus` | ✅ Pass |
| Nginx installed at boot | Yes | Yes (cloud-init) | ✅ Pass |
| Nginx service running | Yes | `Active (running)` | ✅ Pass |
| Port 80 accessible from internet | Yes | HTTP 200 OK | ✅ Pass |
| Storage policy compliant | Standard_LRS | Standard_LRS | ✅ Pass |
| Disk size ≤ 128 GB | Yes | 30 GB | ✅ Pass |

**Final HTTP Response:** `200 OK` · `nginx/1.18.0 (Ubuntu)` · `Sun, 01 Mar 2026 02:06:17 GMT`

---

## 🔐 Security Posture

| Control | Implementation |
|---|---|
| **SSH Access** | Password auth enabled · Port 22 open (lab environment) |
| **HTTP Exposure** | Port 80 open to `0.0.0.0/0` as per specification |
| **HTTPS / TLS** | Not configured (out of scope for this lab) |
| **Firewall** | Azure NSG · explicit allow rules · default-deny inbound |
| **Disk Encryption** | Azure platform-managed encryption at rest (default) |
| **Trusted Launch** | Enabled — Secure Boot + vTPM active |
| **Storage SKU** | Standard_LRS (policy compliant, non-Premium) |

> **Production Recommendation:** Replace password authentication with SSH key pairs, restrict port 22 to known CIDR ranges, and terminate TLS at an Azure Application Gateway or Load Balancer in front of this VM.

---

## 🧠 Key Learnings

**1. Specify VM size explicitly, never rely on defaults.**
Azure default SKUs are subject to regional capacity constraints. Targeting broadly available sizes like `Standard_B1s` ensures consistent, predictable deployments across regions and time windows.

**2. Enforce storage SKU compliance from the start.**
Enterprise Azure subscriptions commonly carry Azure Policy definitions that block Premium storage. Passing `--storage-sku os=Standard_LRS` proactively prevents policy-driven deployment failures before they occur.

**3. Name your OS disk explicitly in iterative deployments.**
Providing `--os-disk-name` prevents orphaned disk conflicts when redeploying into the same resource group — a critical pattern in CI/CD pipelines and environments with short resource lifespans.

**4. Resource deletion is asynchronous, always poll before retry.**
`az vm delete` returning exit code `0` does not guarantee backend teardown is complete. Polling with `az vm show` until `ResourceNotFound` eliminates race conditions on redeployment.

**5. Always validate from an external client perspective.**
Running `curl -I` from outside the VM is the only definitive test of public HTTP reachability. Internal checks bypass NSG rules entirely and can produce false-positive results.

---

## 📚 References

| Resource | Link |
|---|---|
| Azure VM CLI Reference | https://learn.microsoft.com/en-us/cli/azure/vm |
| Azure Policy Overview | https://learn.microsoft.com/en-us/azure/governance/policy/overview |
| Nginx on Ubuntu | https://nginx.org/en/linux_packages.html#Ubuntu |
| Azure NSG Documentation | https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview |
| Cloud-init on Azure | https://learn.microsoft.com/en-us/azure/virtual-machines/linux/using-cloud-init |
| Azure Managed Disk SKUs | https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types |

---

<div align="center">

**Nautilus DevOps Project · Azure Infrastructure Lab**

*Provisioned · Secured · Validated · Production-Ready*

`devops-vm` · `eastus` · `nginx/1.18.0` · `HTTP 200 OK`

</div>
