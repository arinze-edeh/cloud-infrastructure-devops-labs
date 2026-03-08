# Azure VM Nginx Public Accessibility Remediation

![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?style=flat-square&logo=microsoftazure&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-1.18.0-009639?style=flat-square&logo=nginx&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=flat-square)

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Root Cause Analysis](#root-cause-analysis)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Resolution Walkthrough](#resolution-walkthrough)
- [Verification](#verification)
- [Screenshots](#screenshots)
- [Key Takeaways](#key-takeaways)
- [References](#references)

---

## Overview

This document details the end-to-end investigation and resolution of a production-blocking network accessibility issue affecting an Azure Virtual Machine (`nautilus-vm`) running an Nginx web server. The VM was deployed within a Virtual Network (`nautilus-vnet`) in the `westus` region under the resource group `kml_rg_main-464891cf7c214ddc`. Despite a running Nginx instance, the server was completely unreachable from the public internet.

This runbook serves as an incident post-mortem and a reusable remediation guide for Azure infrastructure engineers dealing with similar network misconfiguration scenarios.

---

## Problem Statement

### Symptom

An Nginx web server deployed on `nautilus-vm` was inaccessible from the public internet on port 80, despite the server being confirmed operational within the VM.

### Impact

| Dimension | Detail |
|---|---|
| **Severity** | High (P1) |
| **Affected Resource** | `nautilus-vm` (Azure VM, West US) |
| **Affected Service** | Nginx HTTP Server (Port 80) |
| **Public IP** | `20.237.243.56` (nautilus-pip) |
| **Start Time** | Sun Mar 08 06:24:47 UTC 2026 |
| **Resolution Time** | Sun Mar 08 07:24:47 UTC 2026 |

---

## Root Cause Analysis

Three compounding misconfigurations were identified, each independently sufficient to block public internet access:

### Issue 1: Misconfigured Route Table (Primary Cause)

The route table `nautilus-rtb` contained a route named `Block-Internet` with `addressPrefix: 0.0.0.0/0` and `nextHopType: None`. This silently dropped all outbound traffic destined for the internet, effectively blackholing all external communication.

```json
{
  "name": "Block-Internet",
  "addressPrefix": "0.0.0.0/0",
  "nextHopType": "None"
}
```

### Issue 2: No Inbound NSG Rule for HTTP (Secondary Cause)

The Network Security Group `nautilus-vmNSG` had only one inbound rule (`default-allow-ssh` on port 22). There was no rule permitting inbound TCP traffic on port 80, so even if routing was correct, HTTP requests would be blocked at the security layer.

### Issue 3: Public IP Not Attached to NIC (Tertiary Cause)

The public IP address `nautilus-pip` (`20.237.243.56`) existed as a standalone resource but was not associated with the VM's network interface (`nautilus-vmVMNic`). As a result, the VM had no publicly routable address, regardless of NSG or routing configuration.

---

## Architecture

```
Internet
    |
    | HTTP :80
    v
[ Public IP: nautilus-pip ]
    |
    | (was detached -- FIXED)
    v
[ NIC: nautilus-vmVMNic ]
    |
    | Private IP: 10.0.0.4
    v
[ Subnet: nautilus-subnet ]
    |
    |-- [ NSG: nautilus-vmNSG ]      <-- Port 80 rule was missing (FIXED)
    |-- [ Route Table: nautilus-rtb ] <-- nextHopType was None (FIXED)
    |
    v
[ VM: nautilus-vm ]
    |
    v
[ Nginx: listening on :80 ]
```

---

## Prerequisites

Before executing this remediation, ensure you have:

- Azure CLI (`az`) installed and authenticated
- `Contributor` or `Network Contributor` RBAC role on the target resource group
- SSH access to the target VM (port 22 must be open)
- Target resource group name exported as an environment variable

```bash
export RESOURCE_GROUP="kml_rg_main-464891cf7c214ddc"
export NIC_NAME="nautilus-vmVMNic"
```

---

## Resolution Walkthrough

### Step 1: Inventory and Discovery

List all VMs and confirm the target resource group:

```bash
az group list --output table
az vm list --output table
```

Retrieve the NIC ID attached to the VM:

```bash
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name nautilus-vm \
  --query "networkProfile.networkInterfaces[].id" \
  --output tsv
```

### Step 2: Audit Existing Network Configuration

Inspect the route table and confirm the blocking route:

```bash
az network route-table show \
  --resource-group $RESOURCE_GROUP \
  --name nautilus-rtb \
  --query "routes" \
  --output json
```

**Expected problematic output:**

```json
[
  {
    "name": "Block-Internet",
    "nextHopType": "None",
    "addressPrefix": "0.0.0.0/0"
  }
]
```

Inspect NSG rules to confirm absence of HTTP rule:

```bash
az network nsg rule list \
  --resource-group $RESOURCE_GROUP \
  --nsg-name nautilus-vmNSG \
  --output table
```

Verify public IP attachment status:

```bash
az network public-ip show \
  --resource-group $RESOURCE_GROUP \
  --name nautilus-pip \
  --query "{Name:name, IP:ipAddress, AttachedTo:ipConfiguration}" \
  --output json
```

### Step 3: Fix Route Table (Restore Internet Routing)

Update the `Block-Internet` route to forward traffic to the internet instead of dropping it:

```bash
az network route-table route update \
  --resource-group $RESOURCE_GROUP \
  --route-table-name nautilus-rtb \
  --name Block-Internet \
  --next-hop-type Internet
```

**Verified output:**

```json
{
  "name": "Block-Internet",
  "nextHopType": "Internet",
  "provisioningState": "Succeeded"
}
```

### Step 4: Add NSG Inbound Rule for HTTP Port 80

Create an explicit allow rule for inbound HTTP traffic:

```bash
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name nautilus-vmNSG \
  --name Allow-HTTP-80 \
  --protocol Tcp \
  --direction Inbound \
  --priority 100 \
  --source-address-prefix "*" \
  --source-port-range "*" \
  --destination-address-prefix "*" \
  --destination-port-range 80 \
  --access Allow
```

### Step 5: Associate Public IP with NIC

Attach `nautilus-pip` to the VM's IP configuration:

```bash
az network nic ip-config update \
  --resource-group $RESOURCE_GROUP \
  --nic-name $NIC_NAME \
  --name ipconfignautilus-vm \
  --public-ip-address nautilus-pip
```

### Step 6: Install and Enable Nginx on the VM

SSH into the VM and deploy Nginx:

```bash
ssh -o StrictHostKeyChecking=no azureuser@20.237.243.56
```

```bash
sudo apt-get update -y && \
sudo apt-get install -y nginx && \
sudo systemctl start nginx && \
sudo systemctl enable nginx
```

Confirm Nginx is responding locally:

```bash
curl http://localhost
```

---

## Verification

### Local Verification (inside VM)

```bash
curl http://localhost
# Expected: Nginx default HTML page with HTTP 200
```

### External Verification (from internet)

```bash
curl --connect-timeout 15 -o /dev/null -s -w "HTTP Status: %{http_code}\n" http://20.237.243.56
# Expected output: HTTP Status: 200
```

---

## Screenshots

### Network Topology and Resource Group Overview

> ***Screenshot Placeholder***
> *Caption: Azure Portal showing `kml_rg_main-464891cf7c214ddc` resource group with all network components: `nautilus-vm`, `nautilus-vnet`, `nautilus-vmNSG`, `nautilus-rtb`, and `nautilus-pip`.*

---

### Route Table Before Fix (Block-Internet with nextHopType: None)

> ***Screenshot Placeholder***
> *Caption: Azure Portal route table view showing the `Block-Internet` route with `nextHopType` set to `None`, confirming the root cause that silently dropped all outbound traffic.*

---

### Route Table After Fix (nextHopType: Internet)

> ***Screenshot Placeholder***
> *Caption: Updated route table showing `Block-Internet` route with `nextHopType` changed to `Internet`, restoring outbound connectivity for the subnet.*

---

### NSG Rules Before Remediation (Port 80 Missing)

> ***Screenshot Placeholder***
> *Caption: NSG `nautilus-vmNSG` inbound rules showing only the default `default-allow-ssh` rule on port 22, with no HTTP rule present.*

---

### NSG Rules After Remediation (Allow-HTTP-80 Added)

> ***Screenshot Placeholder***
> *Caption: NSG `nautilus-vmNSG` inbound rules after fix, showing the newly created `Allow-HTTP-80` rule at priority 100 permitting TCP traffic on port 80 from all sources.*

---

### Public IP Association (nautilus-pip Attached to NIC)

> ***Screenshot Placeholder***
> *Caption: NIC IP configuration view showing `nautilus-pip` (20.237.243.56) successfully attached to `ipconfignautilus-vm` on `nautilus-vmVMNic`.*

---

### Nginx Default Page Accessible from Internet

> ***Screenshot Placeholder***
> *Caption: Browser or curl output confirming the Nginx default welcome page is reachable at `http://20.237.243.56` with HTTP 200, validating full end-to-end resolution.*

---

## Key Takeaways

### What Went Wrong

| # | Issue | Resource | Impact |
|---|---|---|---|
| 1 | Route `nextHopType` set to `None` | `nautilus-rtb` | All outbound traffic silently dropped |
| 2 | No inbound NSG rule for port 80 | `nautilus-vmNSG` | HTTP requests blocked at security layer |
| 3 | Public IP detached from NIC | `nautilus-pip` | VM had no routable public address |

### Remediation Summary

| # | Action Taken | Command |
|---|---|---|
| 1 | Updated route `nextHopType` to `Internet` | `az network route-table route update` |
| 2 | Added inbound NSG rule `Allow-HTTP-80` | `az network nsg rule create` |
| 3 | Attached `nautilus-pip` to NIC IP config | `az network nic ip-config update` |

### Prevention Recommendations

* Apply Infrastructure as Code (IaC) using Terraform or Bicep to enforce consistent, reviewable network configurations at deployment time.
* Implement Azure Policy to prevent route tables with `nextHopType: None` for default routes from being applied to production subnets.
* Use Azure Network Watcher and IP Flow Verify during deployment validation to catch connectivity issues before they reach production.
* Tag all public IP resources with their intended NIC association to prevent orphaned IP configurations.
* Include port 80 and port 443 NSG rules in all baseline VM deployment templates serving HTTP workloads.

---

## References

* [Azure Route Tables Documentation](https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table)
* [Azure Network Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
* [Associate a Public IP Address to a VM](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/associate-public-ip-address-vm)
* [Azure CLI Network Reference](https://learn.microsoft.com/en-us/cli/azure/network)
* [Nginx on Ubuntu 22.04](https://nginx.org/en/docs/install.html)

---

**Author:** DevOps Engineering Team
**Last Updated:** March 2026
**Region:** West US | **Resource Group:** `kml_rg_main-464891cf7c214ddc`

<img width="1037" height="389" alt="image" src="https://github.com/user-attachments/assets/d3880703-da5e-4c55-b445-065a8cf5018d" />
<img width="1028" height="413" alt="image" src="https://github.com/user-attachments/assets/8aae56e7-eb35-4a36-a21f-152648c4a64c" />
<img width="1035" height="522" alt="image" src="https://github.com/user-attachments/assets/e9b9ee12-1e75-4a4c-815c-2136d8fb7859" />
<img width="1035" height="522" alt="image" src="https://github.com/user-attachments/assets/b9363021-76eb-4869-a02f-84ca060fab9a" />
<img width="1183" height="654" alt="image" src="https://github.com/user-attachments/assets/3d9a7ec4-f633-446a-b5d4-cfa12ae23eba" />
<img width="1186" height="794" alt="image" src="https://github.com/user-attachments/assets/c236a4d2-37a9-4b4e-a3e7-9667bc49caae" />
<img width="1188" height="735" alt="image" src="https://github.com/user-attachments/assets/ece3556b-2dd9-4bfc-a1cb-ad0933a89803" />
<img width="1185" height="768" alt="image" src="https://github.com/user-attachments/assets/13695bff-9557-4549-9743-342d44ea8def" />
<img width="1188" height="467" alt="image" src="https://github.com/user-attachments/assets/3eeb447e-3001-4d74-884e-92429715f55c" />
<img width="1186" height="440" alt="image" src="https://github.com/user-attachments/assets/7f076aef-68b0-4724-bb27-246eb91fe62f" />
<img width="1186" height="582" alt="image" src="https://github.com/user-attachments/assets/7bc66bab-002e-4465-a0f0-24b04789c7fe" />
<img width="1183" height="865" alt="image" src="https://github.com/user-attachments/assets/58f43c13-0ee8-4177-b687-7d13492bf1a7" />
<img width="1184" height="601" alt="image" src="https://github.com/user-attachments/assets/734e9263-8474-46e1-9562-fa201c92c4af" />
<img width="1179" height="732" alt="image" src="https://github.com/user-attachments/assets/1cd12e0e-cbbc-411b-b22a-0825cec2ff1f" />
<img width="1183" height="844" alt="image" src="https://github.com/user-attachments/assets/30808382-80de-4c2a-9fba-72451bcb5c3e" />
<img width="1184" height="860" alt="image" src="https://github.com/user-attachments/assets/8be3fc75-0dc0-4a1a-b378-9e891b0b8a9e" />
<img width="1179" height="857" alt="image" src="https://github.com/user-attachments/assets/a7577aef-33b6-4ff2-8da4-30173f493f85" />
<img width="1181" height="605" alt="image" src="https://github.com/user-attachments/assets/5671a45a-64c3-4516-b3a1-490c1b4ac6e7" />
<img width="1184" height="665" alt="image" src="https://github.com/user-attachments/assets/b485062f-5ab7-4303-8685-29441b093fed" />
