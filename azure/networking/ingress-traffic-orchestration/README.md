# Azure Load Balancer Provisioning for Nginx Workload

![Azure](https://img.shields.io/badge/Azure-Load%20Balancer-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Region](https://img.shields.io/badge/Region-East%20US-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![SKU](https://img.shields.io/badge/SKU-Standard-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=for-the-badge)
![CLI](https://img.shields.io/badge/Azure%20CLI-2.67.0-blue?style=for-the-badge&logo=windowsterminal&logoColor=white)

---

## Table of Contents

* [Problem Statement](#problem-statement)
* [Resolution Overview](#resolution-overview)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Environment Reference](#environment-reference)
* [Phase 1: Pre-Flight Verification](#phase-1-pre-flight-verification)
* [Phase 2: Create the Public IP Address](#phase-2-create-the-public-ip-address)
* [Phase 3: Create the Load Balancer](#phase-3-create-the-load-balancer)
* [Phase 4: Create the Health Probe](#phase-4-create-the-health-probe)
* [Phase 5: Create the Load Balancer Rule](#phase-5-create-the-load-balancer-rule)
* [Phase 6: Add the VM to the Backend Pool](#phase-6-add-the-vm-to-the-backend-pool)
* [Phase 7: Configure NSG Inbound Rule](#phase-7-configure-nsg-inbound-rule)
* [Phase 8: End-to-End Validation](#phase-8-end-to-end-validation)
* [Resource Summary](#resource-summary)
* [Troubleshooting](#troubleshooting)

---

## Problem Statement

The Nautilus DevOps team required a production-grade Layer 4 ingress path for an Nginx workload running on an Azure Virtual Machine. The VM was accessible only via its direct public IP, with no load balancing, health monitoring, or controlled inbound traffic policy in place. This created a single point of failure with no ability to scale horizontally or enforce traffic governance at the network boundary.

**The following gaps were identified:**

* No Azure Load Balancer fronting the VM
* No public IP decoupled from the VM for LB ingress
* No backend pool registering the VM as a healthy target
* No health probe monitoring the Nginx service on port 80
* No load balancing rule to route external HTTP traffic to the backend
* No NSG inbound rule permitting HTTP traffic on port 80 to the VM NIC

---

## Resolution Overview

A Standard Azure Load Balancer was provisioned in the `eastus` region with a dedicated static public IP, a backend pool containing the existing VM, an HTTP health probe, a TCP load balancing rule on port 80, and an NSG inbound rule to allow HTTP traffic. End-to-end validation confirmed the Nginx welcome page was served through the load balancer's public IP.

---

## Architecture

```
Internet
    |
    v
[ xfusion-lb-ip ]  (Static Public IP: 52.188.1.76)
    |
    v
[ xfusion-lb ]  (Standard Azure Load Balancer - eastus)
    |
    |-- Frontend IP Config: xfusion-lb-ip
    |-- LB Rule: xfusion-lb-rule (TCP 80 -> 80)
    |-- Health Probe: xfusion-health-probe (HTTP :80 /)
    |
    v
[ xfusion-backend-pool ]
    |
    v
[ xfusion-vm ]  (NIC: xfusion-vmVMNic | IP Config: ipconfigxfusion-vm)
    |
    |-- Private IP: 10.0.0.4
    |-- NSG: xfusion-vmNSG (Allow TCP 80 Inbound)
    |
    v
[ Nginx :80 ]
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Version 2.67.0 or later |
| Authenticated Session | Service Principal or User account with Contributor role |
| Existing Resource Group | Must exist in `eastus` before execution |
| Existing Virtual Machine | Running, with Nginx serving on port 80 |
| Existing NSG | Attached to the VM NIC |

---

## Environment Reference

All commands in this guide use the following resolved values. No placeholders remain.

| Variable | Value |
|---|---|
| **Subscription ID** | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| **Tenant ID** | `54c1a2d3-d100-453c-9636-3a109eb45552` |
| **Resource Group** | `kml_rg_main-d92445381fae45bd` |
| **Region** | `eastus` |
| **Virtual Machine** | `xfusion-vm` |
| **VM NIC** | `xfusion-vmVMNic` |
| **IP Config Name** | `ipconfigxfusion-vm` |
| **NSG Name** | `xfusion-vmNSG` |
| **Public IP Name** | `xfusion-lb-ip` |
| **Public IP Address** | `52.188.1.76` |
| **Load Balancer Name** | `xfusion-lb` |
| **Backend Pool** | `xfusion-backend-pool` |
| **Health Probe** | `xfusion-health-probe` |
| **LB Rule** | `xfusion-lb-rule` |

---

## Phase 1: Pre-Flight Verification

Verify the Azure CLI version, active subscription, and confirm the target resource group and VM are present and healthy in `eastus` before provisioning any resources.

### Step 1: Verify Azure CLI Version and Login State

```bash
az --version
az account show
```

**Expected:** CLI version `2.67.0` or later. Account state must be `Enabled`.

> **Screenshot**
<img width="1030" height="783" alt="image" src="https://github.com/user-attachments/assets/0b41e077-6b95-42fe-b7a3-39349b590976" />

> *Caption: Azure CLI version output and active account confirmation showing subscription "Azure Free Labs" in Enabled state.*

---

### Step 2: Confirm Active Subscription

```bash
az account list --output table
```

**Expected:** One row showing `Azure Free Labs` with `IsDefault: True` and `State: Enabled`.

> **Screenshot**
<img width="1032" height="824" alt="image" src="https://github.com/user-attachments/assets/168b0aa0-530c-437f-b67f-5b31f49ecfda" />

> *Caption: Subscription list confirming Azure Free Labs is the active default subscription.*

---

### Step 3: Confirm Resource Group and VM Exist in eastus

```bash
az group list --output table
az vm list --output table --show-details
```

**Expected:**
* Resource group `kml_rg_main-d92445381fae45bd` with `Location: eastus` and `Status: Succeeded`
* VM `xfusion-vm` with `PowerState: VM running` and `Location: eastus`

> **Screenshot**

<img width="1032" height="631" alt="image" src="https://github.com/user-attachments/assets/ae18cdb7-b7c0-42bd-a57a-3eb86b1f7512" />

> *Caption: Resource group list and VM list confirming xfusion-vm is running in eastus under the correct resource group.*

---

## Phase 2: Create the Public IP Address

Provision a Standard SKU static public IP named `xfusion-lb-ip`. This IP will serve as the single ingress point for all traffic routed through the load balancer.

### Step 4: Create the Public IP

```bash
az network public-ip create \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-lb-ip \
  --location eastus \
  --sku Standard \
  --allocation-method Static \
  --output table
```

> **Note:** A warning about zone-redundancy behavior for Standard SKU IPs may appear. This is an informational notice about a future CLI default change and does not affect the resource or this implementation.

---

### Step 5: Verify the Public IP

```bash
az network public-ip show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-lb-ip \
  --output table
```

**Expected:** `ProvisioningState: Succeeded`, `Location: eastus`, `Address: 52.188.1.76`

> **Screenshot**
<img width="1035" height="712" alt="image" src="https://github.com/user-attachments/assets/b6e79d67-4be0-4344-91d1-5c4ad50784d0" />

> *Caption: Public IP xfusion-lb-ip showing ProvisioningState Succeeded with assigned address 52.188.1.76 in eastus.*

---

## Phase 3: Create the Load Balancer

Create the Standard Load Balancer `xfusion-lb` with the frontend IP configuration and backend pool provisioned in a single command.

### Step 6: Create the Load Balancer

```bash
az network lb create \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-lb \
  --location eastus \
  --sku Standard \
  --frontend-ip-name xfusion-lb-ip \
  --public-ip-address xfusion-lb-ip \
  --backend-pool-name xfusion-backend-pool \
  --output table
```

**This single command provisions:**
* The load balancer `xfusion-lb`
* The frontend IP configuration named `xfusion-lb-ip` bound to the public IP
* The backend pool `xfusion-backend-pool`

---

### Step 7: Verify the Load Balancer

```bash
az network lb show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-lb \
  --output table
```

**Expected:** `Name: xfusion-lb`, `Location: eastus`, `ProvisioningState: Succeeded`

> **Screenshot**
<img width="1034" height="562" alt="image" src="https://github.com/user-attachments/assets/ae0e273d-c79f-4399-b711-e3d91161a60a" />

> *Caption: Load balancer xfusion-lb showing ProvisioningState Succeeded in eastus with its assigned ResourceGuid.*

---

## Phase 4: Create the Health Probe

Create an HTTP health probe that checks the Nginx service at the root path on port 80 every 15 seconds.

### Step 8: Create the Health Probe

```bash
az network lb probe create \
  --resource-group kml_rg_main-d92445381fae45bd \
  --lb-name xfusion-lb \
  --name xfusion-health-probe \
  --protocol Http \
  --port 80 \
  --path "/" \
  --interval 15 \
  --threshold 2 \
  --output table
```

> **Note:** A warning about `numberOfProbes` being deprecated in favor of `probeThreshold` may appear. This is informational only. The probe is created and functional as confirmed by `ProvisioningState: Succeeded`.

---

### Step 9: Verify the Health Probe

```bash
az network lb probe show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --lb-name xfusion-lb \
  --name xfusion-health-probe \
  --output table
```

**Expected:** `Name: xfusion-health-probe`, `Port: 80`, `Protocol: Http`, `RequestPath: /`, `IntervalInSeconds: 15`, `ProvisioningState: Succeeded`

> **Screenshot**
<img width="1322" height="696" alt="image" src="https://github.com/user-attachments/assets/65b32b40-dd9a-46f0-a092-3eb9d10ed6de" />

> *Caption: Health probe xfusion-health-probe confirmed with HTTP protocol on port 80, 15-second interval, and ProvisioningState Succeeded.*

---

## Phase 5: Create the Load Balancer Rule

Create a TCP load balancing rule that routes inbound traffic on frontend port 80 to backend port 80, linked to the health probe.

### Step 10: Create the Load Balancer Rule

```bash
az network lb rule create \
  --resource-group kml_rg_main-d92445381fae45bd \
  --lb-name xfusion-lb \
  --name xfusion-lb-rule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name xfusion-lb-ip \
  --backend-pool-name xfusion-backend-pool \
  --probe-name xfusion-health-probe \
  --idle-timeout 4 \
  --output table
```

---

### Step 11: Verify the Load Balancer Rule

```bash
az network lb rule show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --lb-name xfusion-lb \
  --name xfusion-lb-rule \
  --output table
```

**Expected:** `Name: xfusion-lb-rule`, `FrontendPort: 80`, `BackendPort: 80`, `Protocol: Tcp`, `LoadDistribution: Default`, `ProvisioningState: Succeeded`

> **Screenshot**
<img width="1402" height="799" alt="image" src="https://github.com/user-attachments/assets/db08d8fd-5c57-424c-bfb6-cd500c1c2d9b" />

> *Caption: LB rule xfusion-lb-rule confirmed routing TCP port 80 to backend port 80 with Default load distribution and ProvisioningState Succeeded.*

---

## Phase 6: Add the VM to the Backend Pool

Identify the VM NIC and its IP configuration, then register the VM with the backend pool by binding the NIC IP config to `xfusion-backend-pool`.

### Step 12: Retrieve the VM NIC Resource ID

```bash
az vm show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-vm \
  --query "networkProfile.networkInterfaces[].id" \
  --output tsv
```

**Expected output (extract NIC name from the final path segment):**

```
/subscriptions/.../networkInterfaces/xfusion-vmVMNic
```

**Resolved NIC Name:** `xfusion-vmVMNic`

---

### Step 13: Retrieve the NIC IP Configuration Name

```bash
az network nic ip-config list \
  --resource-group kml_rg_main-d92445381fae45bd \
  --nic-name xfusion-vmVMNic \
  --output table
```

**Expected:** `Name: ipconfigxfusion-vm`, `Primary: True`, `PrivateIPAddress: 10.0.0.4`, `ProvisioningState: Succeeded`

> **Screenshot**
<img width="1081" height="565" alt="image" src="https://github.com/user-attachments/assets/7e875323-8006-4c60-a91b-fcbbc5617f43" />

> *Caption: NIC IP config list for xfusion-vmVMNic showing ipconfigxfusion-vm as the primary config with private IP 10.0.0.4.*

---

### Step 14: Add the VM NIC to the Backend Pool

```bash
az network nic ip-config address-pool add \
  --resource-group kml_rg_main-d92445381fae45bd \
  --nic-name xfusion-vmVMNic \
  --ip-config-name ipconfigxfusion-vm \
  --lb-name xfusion-lb \
  --address-pool xfusion-backend-pool \
  --output table
```

**Expected:** Command returns the IP config row with `ProvisioningState: Succeeded`, confirming the NIC is now registered.

---

### Step 15: Verify the Backend Pool Registration

```bash
az network lb address-pool show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --lb-name xfusion-lb \
  --name xfusion-backend-pool \
  --output table
```

**Expected:** `Name: xfusion-backend-pool`, `ProvisioningState: Succeeded`

> **Screenshot**

<img width="1082" height="849" alt="image" src="https://github.com/user-attachments/assets/f099cba7-58e2-4489-9f53-07c550f17f9b" />

> *Caption: Backend pool xfusion-backend-pool showing ProvisioningState Succeeded, confirming xfusion-vm is a registered target.*

---

## Phase 7: Configure NSG Inbound Rule

Retrieve the NSG attached to the VM NIC and add an inbound rule permitting HTTP traffic on port 80 from any source.

### Step 16: Retrieve the NSG Name from the VM NIC

```bash
az network nic show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-vmVMNic \
  --query "networkSecurityGroup.id" \
  --output tsv
```

**Expected output (extract NSG name from the final path segment):**

```
/subscriptions/.../networkSecurityGroups/xfusion-vmNSG
```

**Resolved NSG Name:** `xfusion-vmNSG`

---

### Step 17: Create the Inbound HTTP Rule

```bash
az network nsg rule create \
  --resource-group kml_rg_main-d92445381fae45bd \
  --nsg-name xfusion-vmNSG \
  --name Allow-HTTP-80 \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80 \
  --output table
```

---

### Step 18: Verify the NSG Rule

```bash
az network nsg rule show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --nsg-name xfusion-vmNSG \
  --name Allow-HTTP-80 \
  --output table
```

**Expected:** `Name: Allow-HTTP-80`, `Priority: 100`, `Direction: Inbound`, `Access: Allow`, `Protocol: Tcp`, `DestinationPortRanges: 80`, `ProvisioningState: Succeeded`

> **Screenshot**
<img width="1401" height="785" alt="image" src="https://github.com/user-attachments/assets/9c3808a2-4d02-472e-b201-458d1314e3fa" />

> *Caption: NSG rule Allow-HTTP-80 confirmed on xfusion-vmNSG allowing inbound TCP port 80 from any source at priority 100.*

---

## Phase 8: End-to-End Validation

Confirm the full traffic path from the load balancer public IP through to the Nginx service running on the VM.

### Step 19: Confirm the Load Balancer Public IP

```bash
az network public-ip show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-lb-ip \
  --query "ipAddress" \
  --output tsv
```

**Expected:** `52.188.1.76`

---

### Step 20: Test HTTP Connectivity Through the Load Balancer

```bash
curl -v http://52.188.1.76
```

**Expected response:**

```
* Connected to 52.188.1.76 port 80
< HTTP/1.1 200 OK
< Server: nginx/1.18.0 (Ubuntu)
< Content-Type: text/html
...
<h1>Welcome to nginx!</h1>
```

**An HTTP 200 response with the Nginx welcome page confirms:**
* The load balancer public IP is reachable
* The frontend IP config and LB rule are correctly routing traffic
* The backend pool contains a healthy, registered VM
* The health probe is passing on HTTP port 80
* The NSG inbound rule is permitting traffic to reach the VM NIC

> **Screenshots**

<img width="1130" height="223" alt="image" src="https://github.com/user-attachments/assets/fc907e91-ea17-439d-9b25-a9090e5cf85a" />
<img width="1393" height="849" alt="image" src="https://github.com/user-attachments/assets/a1a166cf-7d71-4bc8-8b1a-c7cdcfdf5071" />
<img width="1399" height="853" alt="image" src="https://github.com/user-attachments/assets/dcf84acc-98db-4461-8339-645dda3b0a12" />

> *Caption: curl output showing HTTP 200 OK response from Nginx via the load balancer public IP 52.188.1.76, confirming full end-to-end traffic flow.*

---

## Resource Summary

| Resource | Name | Type | State |
|---|---|---|---|
| Public IP | `xfusion-lb-ip` | Standard Static | Succeeded |
| Load Balancer | `xfusion-lb` | Standard SKU | Succeeded |
| Frontend IP Config | `xfusion-lb-ip` | LB Frontend | Succeeded |
| Backend Pool | `xfusion-backend-pool` | LB Backend Pool | Succeeded |
| Health Probe | `xfusion-health-probe` | HTTP :80 / | Succeeded |
| LB Rule | `xfusion-lb-rule` | TCP 80 to 80 | Succeeded |
| NSG Rule | `Allow-HTTP-80` | Inbound TCP :80 | Succeeded |

---

## Troubleshooting

**Health probe showing unhealthy after deployment**

Verify Nginx is running on the VM and listening on port 80:

```bash
az vm run-command invoke \
  --resource-group kml_rg_main-d92445381fae45bd \
  --name xfusion-vm \
  --command-id RunShellScript \
  --scripts "systemctl status nginx && curl -s http://localhost:80"
```

---

**curl returns connection refused or timeout**

Verify the NSG rule exists and is applied to the correct NIC:

```bash
az network nsg rule list \
  --resource-group kml_rg_main-d92445381fae45bd \
  --nsg-name xfusion-vmNSG \
  --output table
```

---

**VM not appearing as a healthy backend target**

Verify the NIC IP config is registered to the backend pool:

```bash
az network nic ip-config show \
  --resource-group kml_rg_main-d92445381fae45bd \
  --nic-name xfusion-vmVMNic \
  --name ipconfigxfusion-vm \
  --query "loadBalancerBackendAddressPools" \
  --output table
```

---

**Backend pool shows Succeeded but no backend instances listed**

Re-run Step 14 to re-register the NIC IP config to the pool. This can occur if the NIC was modified after initial pool registration.

---

*Region: East US | SKU: Standard | CLI: 2.67.0*






<img width="1401" height="719" alt="image" src="https://github.com/user-attachments/assets/dee4984d-241b-4e8d-a21c-5efd2d936b58" />
<img width="1401" height="719" alt="image" src="https://github.com/user-attachments/assets/10fa87e4-a7ff-4cd7-87ed-10ae4c4aad89" />

<img width="1401" height="785" alt="image" src="https://github.com/user-attachments/assets/2814a3ab-ab32-42f0-b6d3-6c6301efb4e2" />



