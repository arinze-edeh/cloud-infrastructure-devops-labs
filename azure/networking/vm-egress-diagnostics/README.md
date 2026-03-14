# Azure VM Egress Diagnostics: NSG Outbound Block Remediation

![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=for-the-badge)
![Region](https://img.shields.io/badge/Region-East_US-blue?style=for-the-badge)

---

## Table of Contents

* [Problem Statement](#problem-statement)
* [Environment Overview](#environment-overview)
* [Root Cause](#root-cause)
* [Prerequisites](#prerequisites)
* [Resolution Workflow](#resolution-workflow)
  * [Phase 0: Authentication and Context](#phase-0-authentication-and-context)
  * [Phase 1: Inventory and Variable Capture](#phase-1-inventory-and-variable-capture)
  * [Phase 2: NSG Diagnosis and SSH Validation](#phase-2-nsg-diagnosis-and-ssh-validation)
  * [Phase 3: Remediation](#phase-3-remediation)
  * [Phase 4: Post-Fix Verification](#phase-4-post-fix-verification)
* [Verification Results](#verification-results)
* [Key Lessons](#key-lessons)
* [Repository Structure](#repository-structure)

---

## Problem Statement

The DevOps team reported that an Azure Virtual Machine was **unable to install or update any packages**. All `apt-get` operations were silently failing, and outbound internet connectivity was completely broken. The VM appeared online from the portal but was operationally isolated from the internet.

**Symptoms observed:**

* `apt-get update` returning `Ign` (ignored) for all package sources
* `ping 8.8.8.8` resulting in `100% packet loss`
* SSH response packets blocked, causing connection timeouts on private IP

**Objectives:**

1. Investigate the connectivity issue preventing package installation on the Azure VM
2. Implement a solution to restore full outbound connectivity and package installation capabilities

---

## Environment Overview

| Property | Value |
|---|---|
| **Cloud Provider** | Microsoft Azure |
| **Region** | East US (`eastus`) |
| **Resource Group** | `KML_RG_MAIN-ABFB64E535324D83` |
| **Target VM** | `nautilus-vm` |
| **VM OS** | Ubuntu 22.04.5 LTS |
| **NIC** | `nautilus-vmVMNic` |
| **NSG** | `nautilus-nsg` |
| **VM Private IP** | `10.0.0.4` |
| **VM Public IP** | `20.121.33.102` |
| **Access Method** | SSH via `/root/.ssh/id_rsa` |

> **Architecture note:** The lab shell environment serves as the `azure-client` jump host. There is no separate `azure-client` VM in the subscription. SSH access to `nautilus-vm` is performed directly from this shell using the pre-provisioned RSA key.

---

## Root Cause

A deliberately injected Network Security Group (NSG) outbound rule named **`Block-All-Outbound`** was configured at **priority 200** on `nautilus-nsg`, denying all outbound traffic (`*`) before any allow rule could be evaluated.

**NSG rule evaluation is strictly priority-ordered (lowest number wins).** The malicious rule at priority 200 fired before the default `AllowInternetOutBound` rule at priority 65001, effectively blackholing all egress traffic from the VM.

```
Priority    Name                   Access    Effect
--------    ----                   ------    ------
200         Block-All-Outbound     DENY  *   <-- Root cause: fires first, blocks everything
65000       AllowVnetOutBound      Allow *   <-- Never reached
65001       AllowInternetOutBound  Allow *   <-- Never reached
65500       DenyAllOutBound        Deny  *   <-- Default deny
```

**Secondary impact:** Blocked outbound traffic also prevented SSH TCP response packets from returning to the client, causing `Connection timed out` errors even though inbound port 22 was explicitly allowed at priority 100.

---

## Prerequisites

* Azure CLI (`az`) installed and accessible in your shell
* SSH private key present at `/root/.ssh/id_rsa` with permissions `600`
* Sufficient Azure RBAC permissions: `Network Contributor` or higher on the resource group

---

## Resolution Workflow

### Phase 0: Authentication and Context

Authenticate to Azure and set the default region to East US.

```bash
az login \
  --username "<LAB_USERNAME>" \
  --password "<LAB_PASSWORD>"
```

```bash
az account set --subscription $(az account list --query "[0].id" --output tsv)
az configure --defaults location=eastus
```

**Verify active subscription:**

```bash
az account show --output table
```

> ***Screenshot Placeholder:*** `01-az-account-show.png` -- Terminal output confirming `AzureCloud` subscription is active with `IsDefault: True` and `State: Enabled`

---

### Phase 1: Inventory and Variable Capture

Capture all required resource identifiers in a single block to avoid variable loss between commands.

```bash
# Capture resource group
RG=$(az vm list --query "[0].resourceGroup" --output tsv)
echo "Resource Group: $RG"

# Capture VM public IP directly (skip private IP -- unreachable from outside VNet)
VM_PIP=$(az network public-ip list \
  --resource-group $RG \
  --query "[0].ipAddress" \
  --output tsv)
echo "VM Public IP: $VM_PIP"

# Capture NIC name
NIC_NAME=$(az vm show \
  --name nautilus-vm \
  --resource-group $RG \
  --query "networkProfile.networkInterfaces[0].id" \
  --output tsv | xargs basename)
echo "NIC: $NIC_NAME"

# Capture NSG name
NSG_NAME=$(az network nic show \
  --name $NIC_NAME \
  --resource-group $RG \
  --query "networkSecurityGroup.id" \
  --output tsv | xargs basename)
echo "NSG: $NSG_NAME"
```

**Expected output:**

```
Resource Group: KML_RG_MAIN-ABFB64E535324D83
VM Public IP:   20.121.33.102
NIC:            nautilus-vmVMNic
NSG:            nautilus-nsg
```

> ***Screenshot Placeholder:*** `02-variable-capture.png` -- Terminal output showing all four variables successfully resolved with correct values

---

### Phase 2: NSG Diagnosis and SSH Validation

Enumerate all NSG rules (outbound and inbound) and validate SSH access simultaneously.

```bash
echo "=== OUTBOUND RULES ===" && \
az network nsg rule list \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --include-default \
  --output table \
  --query "[?direction=='Outbound'].{Priority:priority, Name:name, Access:access, DestPort:destinationPortRange}" && \
echo "=== INBOUND RULES ===" && \
az network nsg rule list \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --include-default \
  --output table \
  --query "[?direction=='Inbound'].{Priority:priority, Name:name, Access:access, DestPort:destinationPortRange}" && \
echo "=== TESTING SSH ===" && \
ssh -i /root/.ssh/id_rsa \
    -o StrictHostKeyChecking=no \
    -o ConnectTimeout=15 \
    azureuser@$VM_PIP
```

**NSG outbound rules observed (pre-fix):**

```
Priority    Name                   Access    DestPort
----------  ---------------------  --------  ----------
200         Block-All-Outbound     Deny      *          <-- CULPRIT
65000       AllowVnetOutBound      Allow     *
65001       AllowInternetOutBound  Allow     *
65500       DenyAllOutBound        Deny      *
```

**NSG inbound rules observed:**

```
Priority    Name                           Access    DestPort
----------  -----------------------------  --------  ----------
100         Allow-SSH                      Allow     22
110         Allow-HTTP                     Allow     80
65000       AllowVnetInBound               Allow     *
65001       AllowAzureLoadBalancerInBound  Allow     *
65500       DenyAllInBound                 Deny      *
```

> ***Screenshot Placeholder:*** `03-nsg-rules-pre-fix.png` -- Terminal output showing both outbound and inbound rule tables with `Block-All-Outbound` at priority 200 highlighted

**Connectivity validation inside `nautilus-vm` (pre-fix, confirms breakage):**

```bash
# Run from inside the SSH session
ping -c 2 8.8.8.8
sudo apt-get update 2>&1 | head -5
```

**Pre-fix output confirming the problem:**

```
2 packets transmitted, 0 received, 100% packet loss

Ign:1 http://azure.archive.ubuntu.com/ubuntu jammy InRelease
Ign:2 http://azure.archive.ubuntu.com/ubuntu jammy-updates InRelease
```

> ***Screenshot Placeholder:*** `04-pre-fix-connectivity-failure.png` -- Terminal output inside `nautilus-vm` showing `100% packet loss` on ping and `Ign` responses from `apt-get update`

Exit the SSH session before applying the fix:

```bash
exit
```

---

### Phase 3: Remediation

Delete the offending NSG rule and immediately verify it has been removed.

```bash
az network nsg rule delete \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --name "Block-All-Outbound" && \
echo "=== RULE DELETED -- VERIFYING ===" && \
az network nsg rule list \
  --nsg-name $NSG_NAME \
  --resource-group $RG \
  --include-default \
  --output table \
  --query "[?direction=='Outbound'].{Priority:priority, Name:name, Access:access, DestPort:destinationPortRange}"
```

**Expected post-deletion outbound rule state:**

```
Priority    Name                   Access    DestPort
----------  ---------------------  --------  ----------
65000       AllowVnetOutBound      Allow     *
65001       AllowInternetOutBound  Allow     *
65500       DenyAllOutBound        Deny      *
```

> ***Screenshot Placeholder:*** `05-nsg-rule-deleted.png` -- Terminal output confirming `Block-All-Outbound` is absent from the outbound rule list, with only the three default rules remaining

**What changed:** Traffic destined for the internet now falls through to `AllowInternetOutBound` at priority 65001, which permits all outbound connections. The blackhole is eliminated.

---

### Phase 4: Post-Fix Verification

Run the complete verification suite in a single non-interactive SSH command. This tests internet reachability, DNS resolution, package list refresh, package installation, and binary execution in one operation.

```bash
ssh -i /root/.ssh/id_rsa \
    -o StrictHostKeyChecking=no \
    azureuser@$VM_PIP \
    "ping -c 4 8.8.8.8 && sudo apt-get update && sudo apt-get install -y curl && curl --version"
```

> ***Screenshot Placeholder:*** `06-post-fix-ping-success.png` -- Terminal output showing `4 packets transmitted, 4 received, 0% packet loss` with sub-2ms round-trip times

> ***Screenshot Placeholder:*** `07-apt-get-update-success.png` -- Terminal output showing all 42 package sources resolving with `Hit` and `Get` responses, fetching `44.0 MB` successfully

> ***Screenshot Placeholder:*** `08-curl-install-success.png` -- Terminal output showing `curl` and `libcurl4` downloaded and installed at `7.81.0-1ubuntu1.23`

> ***Screenshot Placeholder:*** `09-curl-version-confirmed.png` -- Terminal output showing `curl 7.81.0 (x86_64-pc-linux-gnu)` with full protocol and feature list confirming successful binary execution

---

## Verification Results

| Test | Pre-Fix | Post-Fix |
|---|---|---|
| `ping 8.8.8.8` (4 packets) | `100% packet loss` | `0% packet loss`, avg `1.85ms` |
| `apt-get update` | All sources `Ign` (ignored) | `44.0 MB` fetched in `9s` |
| `apt-get install curl` | Failed (no connectivity) | `1 upgraded, 1 newly installed` |
| `curl --version` | Command not found | `curl 7.81.0` confirmed |
| SSH via private IP `10.0.0.4` | `Connection timed out` | N/A (public IP used) |
| SSH via public IP `20.121.33.102` | Connected (inbound allowed) | Connected |

---

## Key Lessons

**1. Capture all variables upfront in a single block.**
Variable loss between separate terminal commands causes cascading failures. Setting `RG`, `NIC_NAME`, `NSG_NAME`, and `VM_PIP` in one atomic block eliminates this risk entirely.

**2. Always SSH via public IP, not private IP.**
Private IP (`10.0.0.4`) is only reachable from within the same VNet. When the jump host is outside that VNet or is the lab shell itself, the private IP will always time out. Resolve the public IP first and use it exclusively.

**3. NSG rule priority is absolute and linear.**
Lower priority number always wins. A `Deny *` rule at priority 200 overrides every `Allow` rule at 65000+, regardless of how permissive those rules are. Always sort and inspect rules by priority when diagnosing connectivity.

**4. Blocked outbound traffic also breaks SSH response paths.**
Even when inbound port 22 is explicitly allowed, a blanket outbound deny prevents TCP response packets from leaving the VM, making SSH appear broken from the inbound side when the real problem is egress.

**5. Non-interactive SSH for verification is faster and cleaner.**
Chaining verification commands as a single SSH argument (`ssh user@host "cmd1 && cmd2 && cmd3"`) eliminates interactive session overhead and produces a single, clean output block suitable for documentation.

---

## Repository Structure

```
networking/
  vm-egress-diagnostics/
    README.md                  # This document
    datacenter-vm/             # Lab 1: datacenter-vm connectivity remediation
    nautilus-vm/               # Lab 2: nautilus-vm connectivity remediation (this lab)
```

---

*Maintained by the Nautilus DevOps Team | Region: East US | Classification: Runbook*



<img width="1031" height="711" alt="image" src="https://github.com/user-attachments/assets/9b83a617-b655-4551-addc-3302f89144f4" />
<img width="1035" height="756" alt="image" src="https://github.com/user-attachments/assets/d80b5406-79e3-4314-9636-2784f42bbb11" />
<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/b74b9fe2-bb83-4c61-b34e-c4bc328c5923" />
<img width="1028" height="862" alt="image" src="https://github.com/user-attachments/assets/cf0d47bb-2644-417b-93ba-77ac1f2c4b5a" />
<img width="1030" height="671" alt="image" src="https://github.com/user-attachments/assets/2d60dab8-45e7-4b6d-8dca-b2862f409e3b" />
<img width="1029" height="856" alt="image" src="https://github.com/user-attachments/assets/6931d647-5f26-4311-be53-2c7d323bdacb" />
<img width="1032" height="857" alt="image" src="https://github.com/user-attachments/assets/ea3b7a5b-46fe-454b-946f-b0e9273e5e5d" />
<img width="1031" height="869" alt="image" src="https://github.com/user-attachments/assets/c69d18bf-ad23-4847-889e-d3adbb41a431" />
