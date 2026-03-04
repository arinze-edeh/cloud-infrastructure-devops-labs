# Azure VM SSH Provisioning

> **Secure, password-less SSH access to an Azure Virtual Machine provisioned via Azure CLI from a designated landing host.**

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Problem Statement and Resolutions](#problem-statement-and-resolutions)
- [Implementation](#implementation)
  - [Phase 1 - SSH Key Generation](#phase-1---ssh-key-generation)
  - [Phase 2 - Azure CLI Authentication](#phase-2---azure-cli-authentication)
  - [Phase 3 - Virtual Machine Provisioning](#phase-3---virtual-machine-provisioning)
  - [Phase 4 - Network Security Configuration](#phase-4---network-security-configuration)
  - [Phase 5 - Connectivity Verification](#phase-5---connectivity-verification)
- [Infrastructure Summary](#infrastructure-summary)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Repository Structure](#repository-structure)

---

## Overview

This runbook documents the end-to-end provisioning of an Azure Virtual Machine (`devops-vm`) with hardened, password-less SSH access. The VM is deployed in the `westus` region using Azure CLI from a designated `azure-client` landing host, satisfying organizational policy constraints on disk SKU and storage configuration.

The solution demonstrates resilient infrastructure provisioning under restrictive Azure Policy environments, including a documented approach to recovering from policy-blocked deployments without portal access.

---

## Architecture

```
+-------------------+          SSH (Port 22)         +----------------------+
|   azure-client    | -----------------------------> |     devops-vm        |
|  (Landing Host)   |    Key: ~/.ssh/id_rsa           |  Ubuntu 22.04 LTS    |
|                   |    User: azureuser              |  westus              |
|  RSA 4096 Key     |    IP:   104.45.223.126         |  Standard_B1s        |
+-------------------+                                +----------------------+
                                                             |
                                               +------------+-------------+
                                               |    Azure Infrastructure  |
                                               |  NSG: devops-vmNSG       |
                                               |  VNET: devops-vmVNET     |
                                               |  NIC: devops-vmVMNic     |
                                               |  Disk: Standard_LRS 30GB |
                                               +--------------------------+
```

***Screenshot Placeholder: Architecture diagram in Azure Portal showing VM, NIC, NSG, and VNET topology***

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Azure CLI | >= 2.x (`az --version`) |
| Azure Subscription | Active, with Contributor role on resource group |
| OS on `azure-client` | Linux (bash shell) |
| Region | `westus` |
| Azure Policy | Standard_LRS or Standard_RAGRS disk SKU, max 128 GB OS disk |

---

## Problem Statement and Resolutions

This deployment encountered three distinct Azure Policy enforcement failures. Each is documented with root cause and resolution.

---

### Problem 1 - Default OS Disk SKU Blocked by Policy

**Symptom**

```
RequestDisallowedByPolicy: Resource 'devops-vm_OsDisk_1_...' was disallowed by policy.
Reasons: This storage configuration is not allowed. Ensure the disk size is 128 GB
or less and the SKU is not Premium for Compute disks.
```

**Root Cause**

`az vm create` defaults to `Premium_LRS` disk SKU for `Standard_B1s` VMs. The lab subscription enforces a global policy requiring `Standard_LRS` or `Standard_RAGRS` only.

**Resolution**

Explicitly pass `--storage-sku Standard_LRS` and `--os-disk-size-gb 30` flags to override the default disk profile.

```bash
az vm create \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30
```

***Screenshot: Terminal showing the RequestDisallowedByPolicy error in red***

---

### Problem 2 - Disk Storage Type Cannot Be Changed on Existing Resource

**Symptom**

```
PropertyChangeNotAllowed: Changing property 'osDisk.managedDisk.storageAccountType'
is not allowed.
```

**Root Cause**

The first failed deployment left a partially provisioned VM resource in Azure. The second `az vm create` attempt tried to update the existing resource's disk type, which Azure does not permit in-place.

**Resolution**

Delete the partial VM using `--force-deletion none` to bypass the policy-blocked disk deletion, then recreate from clean state.

```bash
az vm delete \
  --resource-group "kml_rg_main-c77b2ff0bcbd4ec5" \
  --name "devops-vm" \
  --force-deletion none \
  --yes 2>/dev/null
```

***Screenshot: Terminal showing `az resource list` output with only 4 networking resources remaining after cleanup***

---

### Problem 3 - Policy Blocks Disk Deletion via Standard `az vm delete`

**Symptom**

```
RequestDisallowedByPolicy: Resource 'devops-vm_OsDisk_1_...' was disallowed by policy
during vm delete operation.
```

**Root Cause**

`az vm delete` without `--force-deletion none` attempts to validate and delete the attached OS disk. Since the disk itself was policy-non-compliant, Azure blocked the deletion call.

**Resolution**

Use `--force-deletion none` which detaches and releases the VM without attempting to delete associated disk resources. The orphaned disk eventually garbage-collects or can be removed via Azure REST API.

***Screenshot: Terminal showing successful `az vm delete --force-deletion none` followed by clean `az resource list` output***

---

## Implementation

### Phase 1 - SSH Key Generation

Check for an existing key pair on `azure-client`. If none exists, generate one.

**Check for existing key**

```bash
ls -la ~/.ssh/id_rsa ~/.ssh/id_rsa.pub 2>/dev/null
```

**Generate new RSA 4096-bit key pair (if no existing key)**

```bash
ssh-keygen -t rsa -b 4096 -C "azureuser@devops-vm" -f ~/.ssh/id_rsa -N ""
```

| Flag | Value | Purpose |
|---|---|---|
| `-t rsa` | RSA algorithm | Widely supported, Azure-compatible |
| `-b 4096` | 4096-bit | Strong key length |
| `-C` | `azureuser@devops-vm` | Identifying comment |
| `-f` | `~/.ssh/id_rsa` | Explicit output path |
| `-N ""` | Empty passphrase | Required for password-less automation |

**Load key into variable and verify**

```bash
PUB_KEY=$(cat ~/.ssh/id_rsa.pub) && echo $PUB_KEY
```

Expected output begins with `ssh-rsa AAAA...`

***Screenshot: Terminal showing ssh-keygen randomart image output and successful PUB_KEY echo***

---

### Phase 2 - Azure CLI Authentication

**Authenticate with lab service principal credentials**

```bash
az login \
  -u "kk_lab_user_main-c77b2ff0bcbd4ec5@azurefreekmlprod.onmicrosoft.com" \
  -p "pdw&ah77"
```

**Verify active subscription**

```bash
az account show
```

Expected fields:

```json
{
  "name": "Azure Free Labs",
  "state": "Enabled",
  "user": {
    "type": "servicePrincipal"
  }
}
```

**List available resource groups**

```bash
az group list --output table
```

***Screenshot Placeholder: Terminal showing `az account show` JSON output with `"state": "Enabled"`***

---

### Phase 3 - Virtual Machine Provisioning

> **Note:** Due to Azure Policy enforcement (see [Problem Statement](#problem-statement-and-resolutions)), networking resources must be reused from the prior failed attempt using the `--nics` flag.

**Final working provisioning command**

```bash
az vm create \
  --resource-group "kml_rg_main-c77b2ff0bcbd4ec5" \
  --name "devops-vm" \
  --location "westus" \
  --image "Ubuntu2204" \
  --size "Standard_B1s" \
  --admin-username "azureuser" \
  --ssh-key-values "$PUB_KEY" \
  --authentication-type ssh \
  --nics "devops-vmVMNic" \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS \
  --output json
```

**Expected successful output**

```json
{
  "fqdns": "",
  "location": "westus",
  "macAddress": "60-45-BD-06-4C-87",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "104.45.223.126",
  "resourceGroup": "kml_rg_main-c77b2ff0bcbd4ec5"
}
```

***Screenshot Placeholder: Terminal showing full `az vm create` JSON success output with `"powerState": "VM running"`***

---

### Phase 4 - Network Security Configuration

Open port 22 on the VM's Network Security Group to allow inbound SSH.

```bash
az vm open-port \
  --resource-group "kml_rg_main-c77b2ff0bcbd4ec5" \
  --name "devops-vm" \
  --port 22
```

**Verify NSG rule was applied**

Confirm `securityRules` in the output contains:

```json
{
  "name": "open-port-22",
  "access": "Allow",
  "destinationPortRange": "22",
  "direction": "Inbound",
  "priority": 900,
  "provisioningState": "Succeeded"
}
```

***Screenshot Placeholder: Azure Portal NSG inbound rules view showing `open-port-22` rule at priority 900***

---

### Phase 5 - Connectivity Verification

**SSH from `azure-client` to `devops-vm`**

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@104.45.223.126
```

**Verify hostname inside the VM**

```bash
hostname
```

Expected output: `devops-vm`

**Exit the session**

```bash
exit
```

***Screenshot Placeholder: Terminal showing successful SSH login banner (Ubuntu 22.04 welcome message) and `azureuser@devops-vm:~$` prompt with no password prompt***

***Screenshot Placeholder: Terminal showing `hostname` command returning `devops-vm` and `exit` / `logout` / connection closed sequence***

---

## Infrastructure Summary

| Resource | Name | Value |
|---|---|---|
| Resource Group | `kml_rg_main-c77b2ff0bcbd4ec5` | `eastus` |
| Virtual Machine | `devops-vm` | `westus` |
| VM Size | `Standard_B1s` | 1 vCPU, 1 GB RAM |
| OS Image | Ubuntu 22.04.5 LTS | `Ubuntu2204` |
| OS Disk | `Standard_LRS` | 30 GB |
| Admin User | `azureuser` | SSH key auth only |
| Public IP | `devops-vmPublicIP` | `104.45.223.126` |
| Private IP | eth0 | `10.0.0.4` |
| NSG | `devops-vmNSG` | Port 22 open inbound |
| Virtual Network | `devops-vmVNET` | `10.0.0.0/x` |
| Network Interface | `devops-vmVMNic` | Reused from failed deployment |
| SSH Key Type | RSA 4096-bit | `~/.ssh/id_rsa` |
| Authentication | Key-based only | Password auth disabled |

---

## Troubleshooting

| Error Code | Message | Resolution |
|---|---|---|
| `RequestDisallowedByPolicy` on disk create | Premium SKU not allowed | Add `--storage-sku Standard_LRS --os-disk-size-gb 30` |
| `PropertyChangeNotAllowed` on disk type | Cannot modify existing disk | Delete VM with `--force-deletion none`, then recreate |
| `RequestDisallowedByPolicy` on disk delete | Policy blocks disk deletion | Use `--force-deletion none` to bypass disk cleanup |
| `No such file or directory` on `id_rsa.pub` | SSH key does not exist | Run `ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""` |
| SSH connection timeout | Port 22 blocked | Run `az vm open-port --port 22` |
| `PUB_KEY` variable empty | Key not loaded | Re-run `PUB_KEY=$(cat ~/.ssh/id_rsa.pub)` and verify with `echo $PUB_KEY` |

---

## Security Considerations

* **Password authentication is disabled.** The VM accepts only RSA key-based SSH connections, eliminating brute-force password attack surface.
* **NSG restricts inbound traffic.** Only port 22 is explicitly opened. All other inbound traffic is denied by the default `DenyAllInBound` rule at priority 65500.
* **Private key never leaves `azure-client`.** Only the public key (`id_rsa.pub`) is transmitted to Azure during VM provisioning.
* **Service principal credentials are lab-scoped.** Credentials used during `az login` are time-bound to the lab session window (1 hour) and should be rotated or invalidated post-session.
* **`StrictHostKeyChecking=no` used only for initial connection.** The host fingerprint is persisted to `~/.ssh/known_hosts` on first connect. For production workloads, pre-populate known hosts or use certificate-based verification.

---


<img width="1032" height="432" alt="image" src="https://github.com/user-attachments/assets/3fb90aa4-4ba0-4e9d-8327-324bb472772c" />
<img width="1032" height="562" alt="image" src="https://github.com/user-attachments/assets/3fd240c9-2ee7-4e43-8493-294ee4ee00e6" />
<img width="576" height="522" alt="image" src="https://github.com/user-attachments/assets/9891b20e-b314-40df-8bdc-520737e507df" />
<img width="1027" height="564" alt="image" src="https://github.com/user-attachments/assets/7f718b40-02e5-4f1c-8231-fbb506effb31" />
<img width="1033" height="528" alt="image" src="https://github.com/user-attachments/assets/6734a0fd-7101-4eb2-96cf-4b8bc3660ba8" />
<img width="1035" height="614" alt="image" src="https://github.com/user-attachments/assets/969cd261-345f-40a9-8957-824d75c35844" />
<img width="1043" height="398" alt="image" src="https://github.com/user-attachments/assets/40b7c17b-29d0-4987-8f5a-47a042b3e50a" />
<img width="1028" height="309" alt="image" src="https://github.com/user-attachments/assets/1a0391fe-7ee7-4ecf-bf12-c2cad298f581" />
<img width="1033" height="371" alt="image" src="https://github.com/user-attachments/assets/9e26dd43-46e3-4dc9-9464-c084c4e6f979" />
<img width="1039" height="730" alt="image" src="https://github.com/user-attachments/assets/cb9b5a36-2ff6-4bf7-8bd7-41f50f352c5d" />
<img width="1035" height="872" alt="image" src="https://github.com/user-attachments/assets/9d7a455e-bd43-41fa-bef7-ccd2ea2e19c4" />

<img width="1036" height="855" alt="image" src="https://github.com/user-attachments/assets/c8b2f549-b486-4201-b163-74085d700a86" />

<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/3f039a0c-5f86-4917-977f-6da037f19168" />

<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/9edbf25b-165f-4f54-b322-c7c5f5a02e09" />

<img width="1039" height="868" alt="image" src="https://github.com/user-attachments/assets/f3b4ebfd-8efa-4341-81dc-aa718d16bb20" />

<img width="1034" height="868" alt="image" src="https://github.com/user-attachments/assets/7b11a134-8f8f-420b-a407-9a0548af5e95" />

<img width="1041" height="861" alt="image" src="https://github.com/user-attachments/assets/e091408d-b667-49cb-b577-74769146a66b" />

<img width="1018" height="785" alt="image" src="https://github.com/user-attachments/assets/0916fc94-dbd1-4043-bee6-2a841361b710" />

<img width="1034" height="857" alt="image" src="https://github.com/user-attachments/assets/4d8bd659-a04f-4ccd-b0d8-ca799390b3e8" />



