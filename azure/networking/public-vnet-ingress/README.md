# Azure Public VNet Infrastructure Deployment

![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Deployment-Production_Ready-28a745?style=for-the-badge)
![Region](https://img.shields.io/badge/Region-East_US-0078D4?style=for-the-badge)

---

## Table of Contents

* [Project Overview](#project-overview)
* [Architecture](#architecture)
* [Problem Statement](#problem-statement)
* [Prerequisites](#prerequisites)
* [Infrastructure Components](#infrastructure-components)
* [Deployment Guide](#deployment-guide)
* [Known Issues and Resolutions](#known-issues-and-resolutions)
* [Verification and Validation](#verification-and-validation)
* [Security Considerations](#security-considerations)
* [Contributing](#contributing)

---

## Project Overview

This repository documents the end-to-end deployment of a **public-facing Azure Virtual Network (VNet)** infrastructure designed to host internet-accessible services. The setup provisions a dedicated VNet, a public subnet with automatic IP assignment, and a Linux virtual machine accessible over SSH, all within the Azure **East US** region.

This implementation was executed as part of a DevOps networking task assigned to the **Nautilus DevOps Team** by the **Networking Team**, with the objective of enabling scalable, managed deployment of public-facing applications.

---

## Architecture

```
Azure Subscription (Azure Free Labs)
+
+-- Resource Group: kml_rg_main-562bad02c40d498b
    |
    +-- Virtual Network: devops-pub-vnet (10.0.0.0/16)
    |   |
    |   +-- Subnet: devops-pub-subnet (10.0.0.0/24)
    |       |
    |       +-- VM NIC: devops-pub-vm530
    |       +-- Private IP: 10.0.0.5
    |
    +-- Virtual Machine: devops-pub-vm
    |   +-- OS: Ubuntu 24.04 LTS (x64)
    |   +-- Size: Standard_B1s (1 vCPU, 1 GiB RAM)
    |   +-- OS Disk: Standard HDD (LRS)
    |   +-- Security Type: Standard
    |
    +-- Public IP: 13.68.188.64
    +-- NSG: devops-pub-vm-nsg
        +-- Inbound Rule: TCP Port 22 (SSH) -- Allow
```

---

## Problem Statement

The Networking Team required a publicly accessible virtual network environment on Azure to support a set of internet-facing services. The key requirements were:

* A named public VNet (`devops-pub-vnet`) in the **East US** region
* A subnet (`devops-pub-subnet`) with automatic public IP assignment enabled for resources
* A virtual machine (`devops-pub-vm`) hosted within the VNet
* SSH access on **port 22** open and reachable from the internet
* All resources provisioned under the existing resource group

---

## Prerequisites

### Azure Access

| Requirement | Detail |
|---|---|
| Azure Portal Access | https://portal.azure.com |
| Subscription | Azure Free Labs |
| Resource Group | `kml_rg_main-562bad02c40d498b` |
| Required Role | Contributor or Owner on the subscription |

### Technical Requirements

* A modern web browser (Chrome, Edge, Firefox)
* Active Azure account credentials
* Basic familiarity with Azure Portal navigation

---

## Infrastructure Components

### Virtual Network

| Property | Value |
|---|---|
| Name | `devops-pub-vnet` |
| Region | East US |
| Address Space | `10.0.0.0/16` |
| DNS Servers | Default (Azure-provided) |
| Azure Bastion | Disabled |
| Azure Firewall | Disabled |
| DDoS Protection | Disabled |

### Subnet

| Property | Value |
|---|---|
| Name | `devops-pub-subnet` |
| Address Range | `10.0.0.0/24` |
| Available Addresses | 256 |
| Private Subnet | Disabled (public outbound access enabled) |
| NAT Gateway | None |

### Virtual Machine

| Property | Value |
|---|---|
| Name | `devops-pub-vm` |
| Operating System | Ubuntu Server 24.04 LTS |
| VM Generation | V2 |
| Architecture | x64 |
| Size | Standard_B1s (1 vCPU, 1 GiB RAM) |
| OS Disk Type | Standard HDD (Locally Redundant Storage) |
| Authentication | Password-based |
| Admin Username | `azureuser` |
| Public IP | `13.68.188.64` (Dynamic) |
| Private IP | `10.0.0.5` |
| Security Type | Standard |

### Network Security Group (NSG)

| Rule | Priority | Port | Protocol | Direction | Action |
|---|---|---|---|---|---|
| Allow SSH | 300 | 22 | TCP | Inbound | Allow |

---

## Deployment Guide

### Phase 1 -- Create the Virtual Network

**Step 1: Navigate to Virtual Networks**

1. Log in to the Azure Portal at https://portal.azure.com
2. In the top search bar, type **Virtual Networks** and select it
3. Click **+ Create**

**Step 2: Configure the Basics Tab**

Fill in the following values:

* **Subscription:** Azure Free Labs
* **Resource Group:** `kml_rg_main-562bad02c40d498b`
* **Name:** `devops-pub-vnet`
* **Region:** (US) East US

***Screenshot: VNet Basics Tab***

<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/3201f81c-9115-4dd7-9c0f-ee73e6387e31" />

**Step 3: Configure IP Addresses Tab**

1. Retain the default address space `10.0.0.0/16`
2. Click **+ Add a subnet** or edit the default subnet
3. Set the subnet name to `devops-pub-subnet`
4. Set starting address to `10.0.0.0` and size to `/24`
5. Ensure **Enable private subnet** is **unchecked**
6. Leave NAT gateway as **None**
7. Click **Save**

***Screenshot: Subnet Configuration Panel***

<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/0ffe4f27-6e09-4b58-ab6c-5cf734164f7d" />

**Step 4: Review and Create**

1. Skip Security and Tags tabs (leave defaults)
2. Click **Review + create**
3. Verify validation passes
4. Click **Create**

***Screenshot: VNet Review Page (Validation Passed)***
<img width="1919" height="948" alt="image" src="https://github.com/user-attachments/assets/682bf119-ebe9-4a1b-83a7-9dbc2d2bd652" />

***Screenshot: VNet Deployment Complete***
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/87a4930c-0d0e-4815-a36a-edb3900cfe19" />

---

### Phase 2 -- Create the Virtual Machine

**Step 1: Navigate to Virtual Machines**

1. Search for **Virtual machines** in the top search bar
2. Click **+ Create** then select **Azure virtual machine**

**Step 2: Configure the Basics Tab**

| Field | Value |
|---|---|
| Subscription | Azure Free Labs |
| Resource Group | `kml_rg_main-562bad02c40d498b` |
| Virtual machine name | `devops-pub-vm` |
| Region | (US) East US |
| Availability options | No infrastructure redundancy required |
| Security type | **Standard** |
| Image | Ubuntu Server 24.04 LTS x64 Gen2 |
| Size | Standard_B1s |
| Authentication type | Password |
| Username | `azureuser` |
| Public inbound ports | Allow selected ports |
| Select inbound ports | SSH (22) |

***Screenshot: VM Basics Tab (Upper Section)***
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/7592064a-d7f0-4e71-98d8-7736de2b4783" />

***Screenshot: VM Administrator Account and Inbound Ports***
<img width="1919" height="902" alt="image" src="https://github.com/user-attachments/assets/8e1584b8-b8f8-4042-9130-74afcd280bd6" />

**Step 3: Configure the Disks Tab**

* **OS disk size:** Image default (30 GiB)
* **OS disk type:** `Standard HDD (locally-redundant storage)`
* **Delete with VM:** Checked

***Screenshot: VM Disks Tab (Standard HDD Selected)***
<img width="1919" height="948" alt="image" src="https://github.com/user-attachments/assets/818611a0-a6ea-4a86-b3fe-93052f53e9d2" />

**Step 4: Configure the Networking Tab**

| Field | Value |
|---|---|
| Virtual network | `devops-pub-vnet` |
| Subnet | `devops-pub-subnet (10.0.0.0/24)` |
| Public IP | (new) auto-assigned |
| NIC network security group | Basic |
| Public inbound ports | Allow selected ports |
| Select inbound ports | SSH (22) |

***Screenshot: VM Networking Tab (All Fields Populated)***

<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/6273cafa-d2ee-42a0-ae3d-52cf278b3501" />
<img width="1919" height="953" alt="image" src="https://github.com/user-attachments/assets/920bb18c-ad40-4103-9be8-69dcb5f9af63" />

**Step 5: Review and Create**

1. Click **Review + create**
2. Confirm validation passes
3. Click **Create**
4. Wait approximately 2 to 3 minutes for deployment to complete

***Screenshot: VM Deployment Complete***
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/65e5fded-5902-42d5-9200-f180aa837555" />

---

## Known Issues and Resolutions

### Issue 1 -- VM Deployment Failed with `RequestDisallowedByPolicy`

**Symptom**

The VM deployment failed at the disk provisioning stage with the following error:

```json
{
  "status": "Failed",
  "error": {
    "code": "RequestDisallowedByPolicy",
    "message": "Resource 'devops-pub-vm-disk1_...' was disallowed by policy."
  }
}
```

The deployment details showed the VM itself in **Conflict** state while the NIC, NSG, and Public IP all succeeded.

> ***Screenshot Placeholder -- Deployment Failed Error Screen***
> `![Deploy Failed](screenshots/10-deployment-failed-error.png)`

**Root Cause**

The Azure Free Labs subscription enforces a policy (`global-limits-free_...`) that **disallows Premium SSD and Standard SSD** disk types. The default disk type selected during VM creation was incompatible with this policy constraint.

**Resolution**

1. Delete only the failed VM resource (retain NIC, NSG, and Public IP)
2. Recreate the VM with **OS disk type set to Standard HDD (locally-redundant storage)**
3. Reuse the existing networking resources to avoid naming conflicts

> ***Screenshot Placeholder -- Delete VM Confirmation Dialog***
> `![Delete VM](screenshots/11-delete-vm-confirmation.png)`

> ***Screenshot Placeholder -- Disks Tab with Standard HDD Selected***
> `![Standard HDD Fix](screenshots/12-disk-type-standard-hdd.png)`

**Outcome**

Deployment succeeded on the second attempt with Standard HDD selected.

---

### Issue 2 -- Azure Spot Discount Error

**Symptom**

After navigating back to the VM creation form, the **"Run with Azure Spot discount"** checkbox was accidentally enabled, causing the error:

```
This size does not support Azure Spot.
```

**Root Cause**

Standard_B1s does not support Azure Spot pricing in this subscription tier.

**Resolution**

Scroll up on the Basics tab and uncheck **"Run with Azure Spot discount"**. The size field will return to normal and the error will clear.

---

### Issue 3 -- OS Disk Type Field Shows Loading State

**Symptom**

On the Disks tab, the **OS disk type** dropdown showed a blank loading state immediately after navigating from the Basics tab.

**Root Cause**

Azure Portal requires a brief moment to resolve available disk SKUs based on VM size and region policy constraints.

**Resolution**

Wait 3 to 5 seconds for the field to populate, then manually select **Standard HDD** before proceeding.

---

## Verification and Validation

### VM Health Check

After deployment, navigate to **Virtual Machines** and open `devops-pub-vm`. Verify the following:

| Property | Expected Value | Verified |
|---|---|---|
| Status | Running | [x] |
| Agent Status | Ready | [x] |
| Public IP | Assigned (non-null) | [x] |
| VNet / Subnet | `devops-pub-vnet / devops-pub-subnet` | [x] |
| OS Disk | Present (named disk, not `-`) | [x] |
| Security Type | Standard | [x] |

> ***Screenshot Placeholder -- VM Overview Page (Running State)***
> `![VM Overview](screenshots/13-vm-overview-running.png)`

### NSG Inbound Rules Check

Navigate to **Networking** on the VM blade and verify:

| Priority | Port | Protocol | Action |
|---|---|---|---|
| 300 (or lower) | 22 | TCP | Allow |

> ***Screenshot Placeholder -- NSG Inbound Rules (Port 22 Visible)***
> `![NSG Rules](screenshots/14-nsg-inbound-rules-port22.png)`

### SSH Connectivity Test (Optional)

From Azure Cloud Shell (Bash):

```bash
ssh azureuser@13.68.188.64
# Enter the password configured during VM creation
# Expected: Ubuntu 24.04 LTS welcome banner
```

> ***Screenshot Placeholder -- Successful SSH Login via Cloud Shell***
> `![SSH Login](screenshots/15-ssh-login-success.png)`

---

## Resource Summary

| Resource | Name | Type | Region | Status |
|---|---|---|---|---|
| Virtual Network | `devops-pub-vnet` | Microsoft.Network/virtualNetworks | East US | Active |
| Subnet | `devops-pub-subnet` | Subnet (10.0.0.0/24) | East US | Active |
| Virtual Machine | `devops-pub-vm` | Microsoft.Compute/virtualMachines | East US | Running |
| Public IP | `devops-pub-vm-ip` | Microsoft.Network/publicIpAddresses | East US | Associated |
| Network Interface | `devops-pub-vm530` | Microsoft.Network/networkInterfaces | East US | Active |
| NSG | `devops-pub-vm-nsg` | Microsoft.Network/networkSecurityGroups | East US | Active |

> ***Screenshot Placeholder -- Resource Group Overview (All 6 Resources Listed)***
> `![Resource Group](screenshots/16-resource-group-final-state.png)`

---

## Security Considerations

* **SSH access is open to all IP addresses (0.0.0.0/0).** This configuration is acceptable for lab and testing environments. For production workloads, restrict the SSH inbound rule source to a specific IP range or use Azure Bastion.
* **Password authentication is enabled.** For production environments, SSH key-based authentication is strongly recommended.
* **Private subnet is disabled.** Resources in `devops-pub-subnet` have default outbound internet access. After March 31, 2026, Azure will default new subnets to private. Review your outbound connectivity requirements before that date.
* **Standard HDD** was used due to subscription policy constraints. For production workloads requiring higher IOPS and SLA guarantees, evaluate Premium SSD availability in the target subscription.

---

## Contributing

1. Fork this repository
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Commit your changes: `git commit -m "Add: description of change"`
4. Push the branch: `git push origin feature/your-feature-name`
5. Open a Pull Request with a clear description of the change and its purpose


<img width="1919" height="947" alt="image" src="https://github.com/user-attachments/assets/d3feb2a8-92fd-48e1-8e92-993791efb602" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/4da9c40e-acd9-4f5e-8bce-919d61160627" />

<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/6ba4df7c-d5bd-4fca-8d3d-1a6d97a134a7" />

<img width="1919" height="953" alt="image" src="https://github.com/user-attachments/assets/30cf003d-d89a-4dfd-b508-0d724536dd4f" />

<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/1802dde2-e889-4763-9e87-7c53fa2c6019" />

<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/19790da7-62d3-40a9-8092-756c4d721caf" />
<img width="1919" height="946" alt="image" src="https://github.com/user-attachments/assets/cb74f92c-5806-4469-83a9-092d1e9747f1" />
<img width="1919" height="951" alt="image" src="https://github.com/user-attachments/assets/9466a0e7-0a48-48c2-8663-e0e755aa81a8" />
<img width="1919" height="954" alt="image" src="https://github.com/user-attachments/assets/ac8c2b5d-e56a-4483-a9f3-7b0b105640d5" />
<img width="1919" height="951" alt="image" src="https://github.com/user-attachments/assets/ef2e0629-e747-41b7-80fc-1213e7b8edfc" />
<img width="1919" height="950" alt="image" src="https://github.com/user-attachments/assets/45554f67-c080-466c-9097-0d98506665e3" />
<img width="1919" height="952" alt="image" src="https://github.com/user-attachments/assets/008de4de-7a81-4198-b965-de788d6ee9fb" />
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/dc344488-33a8-4af0-be05-1e5af9120a16" />
<img width="1919" height="951" alt="image" src="https://github.com/user-attachments/assets/1996e2d2-4a6c-4505-975b-7fe9fefd7a2d" />
<img width="1919" height="953" alt="image" src="https://github.com/user-attachments/assets/b883d42d-788e-4869-b66f-7465f0e6b3cf" />
<img width="1919" height="951" alt="image" src="https://github.com/user-attachments/assets/dd060c8b-1605-4a72-851f-d3ad9df455fe" />


<img width="1919" height="946" alt="image" src="https://github.com/user-attachments/assets/1ccaf041-a59a-4824-82c4-6ef61bd737e2" />
<img width="1919" height="948" alt="image" src="https://github.com/user-attachments/assets/1f0305de-3824-433b-8a57-86dc2346f4d0" />
<img width="1919" height="951" alt="image" src="https://github.com/user-attachments/assets/1134b819-e90c-45a7-9aff-2a048cc0d3ba" />
<img width="1919" height="955" alt="image" src="https://github.com/user-attachments/assets/96c3e60c-3090-4642-9831-e6141454d18c" />

<img width="1919" height="948" alt="image" src="https://github.com/user-attachments/assets/3ef018bf-f1e6-4ee9-9f3d-2f59e9486483" />
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/a7c94edb-d22e-456e-bead-c8a4f5e6f509" />
<img width="1919" height="954" alt="image" src="https://github.com/user-attachments/assets/7185ffba-831c-4d46-8777-ee8f006bd0b8" />


