# Provisioning and Validating an Azure Virtual Machine via Portal
**Platform:** Microsoft Azure | **Interface:** Azure Portal (UI) | **OS:** Ubuntu 22.04 LTS | **Scope:** Compute Infrastructure

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Specifications](#architecture-and-specifications)
- [Prerequisites](#prerequisites)
- [High-Level Workflow](#high-level-workflow)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Login to Azure Portal](#step-1-login-to-azure-portal)
  - [Step 2: Create Virtual Machine](#step-2-create-virtual-machine)
  - [Step 3: Configure Basics](#step-3-configure-basics)
  - [Step 4: Configure Networking](#step-4-configure-networking)
  - [Step 5: Configure Disk](#step-5-configure-disk)
  - [Step 6: Review and Deploy](#step-6-review-and-deploy)
  - [Step 7: SSH Verification](#step-7-ssh-verification)
- [Validation Checklist](#validation-checklist)
- [Key Decisions](#key-decisions)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Lessons Learned](#lessons-learned)
- [Conclusion](#conclusion)

---

## Overview

This document details the end-to-end provisioning of an Azure Virtual Machine (VM) using the Azure Portal UI, executed as part of the **Nautilus DevOps team's incremental cloud migration strategy**. The VM serves as a foundational compute resource supporting subsequent infrastructure workloads, including containerized applications, monitoring agents, and integration pipelines.

This implementation follows a **problem-solution-implementation-validation** structure and is intended to serve as onboarding documentation for engineers new to Azure compute provisioning, as well as a handoff reference for production environments.

---

## Problem Statement

The Nautilus DevOps team requires a reproducible, validated process for provisioning Azure Linux VMs that meet specific compliance and operational constraints:

- Controlled VM sizing to manage cost (`Standard_B1s`)
- Standardized OS image for consistency across environments (`Ubuntu 22.04 LTS`)
- Defined disk tier and size to align with I/O expectations (`Standard HDD, 30 GB`)
- Secure remote access enforced at the network layer (SSH via NSG)
- SSH connectivity verified before handoff to downstream teams

The absence of a documented provisioning workflow increases the risk of configuration drift, inconsistent environments, and failed onboarding experiences for new engineers.

---

## Architecture and Specifications

| Parameter | Value |
|---|---|
| **VM Name** | `xfusion-vm` |
| **Region** | `Central US (centralus)` |
| **OS Image** | Ubuntu Server 22.04 LTS |
| **VM Size** | Standard_B1s (1 vCPU, 1 GB RAM) |
| **OS Disk Type** | Standard HDD |
| **OS Disk Size** | 30 GB |
| **Network** | Default VNet and Subnet |
| **NSG Inbound Rule** | Port 22 (SSH) allowed |
| **Authentication** | SSH key pair |
| **Interface** | Azure Portal (UI) |

---

## Prerequisites

- An active Azure subscription with sufficient quota for `Standard_B1s` VMs in the target region
- Access to the Azure Portal with appropriate RBAC role (Contributor or higher on the target Resource Group)
- A pre-existing Resource Group (do not create a new one if working in a managed lab or restricted environment)
- An SSH client available on the local machine (`ssh` on Linux/macOS or Windows Terminal with OpenSSH)
- Permissions to download and store SSH private key files securely

---

## High-Level Workflow

```
Login to Azure Portal
    --> Navigate to Virtual Machines
    --> Click "+ Create" --> Azure Virtual Machine
    --> Configure Basics (name, region, image, size, authentication)
    --> Configure Networking (VNet, subnet, NSG with SSH rule)
    --> Configure Disk (30 GB Standard HDD)
    --> Review + Create (validate and deploy)
    --> Download SSH private key
    --> SSH into VM to verify connectivity
```

---

## Step-by-Step Implementation

---

### Step 1: Login to Azure Portal

Navigate to [https://portal.azure.com](https://portal.azure.com) and authenticate with your assigned credentials. Confirm that the correct subscription and directory context are active before proceeding.

> **Operational Note:** If working in a managed or shared environment, verify the active subscription using the directory selector in the top-right corner of the portal to avoid provisioning resources in the wrong tenant.

**Screenshot: Azure Portal login page**
<img width="1740" height="949" alt="Azure Portal login page" src="https://github.com/user-attachments/assets/4abac257-12d7-4c63-9e74-abfed162ac04" />

---

### Step 2: Create Virtual Machine

From the Azure Portal home, navigate to **Virtual Machines** using the search bar or the left-hand services menu. Click **"+ Create"** and select **"Azure Virtual Machine"** from the dropdown.

> **Operational Note:** The portal may suggest "Azure Virtual Machine (Spot)" for reduced-cost options. Always select the standard Azure Virtual Machine option unless preemptible workloads are explicitly acceptable for the use case.

**Screenshot: Virtual Machines dashboard with Create option**
<img width="1781" height="950" alt="Virtual Machines dashboard showing the + Create button" src="https://github.com/user-attachments/assets/38a332d6-c73e-4323-9a68-6753e13a661b" />

---

### Step 3: Configure Basics

On the **Basics** tab, configure the following parameters:

- **Subscription:** Select the target subscription
- **Resource Group:** Select the pre-existing resource group (do not create a new one in managed environments)
- **Virtual Machine Name:** `xfusion-vm`
- **Region:** `(US) East US`
- **Availability Options:** No infrastructure redundancy required (for this implementation)
- **Image:** `Ubuntu Server 22.04 LTS - x64 Gen2`
- **VM Architecture:** x64
- **Size:** `Standard_B1s` (1 vcpu, 1 GiB memory)
- **Authentication Type:** SSH public key
- **Username:** Set a non-default admin username (e.g., `azureuser`)
- **SSH public key source:** Generate a new key pair (download and store the private key securely)
- **Public inbound ports:** Allow selected ports (SSH port 22)

> **Security Note:** Azure generates an RSA key pair during this step. The private key (`.pem` file) is available for download only once. Store it immediately in a secure location such as a secrets manager or encrypted local vault. Losing the private key means losing SSH access to the VM.

**Screenshots: Basics tab configuration**
<img width="1761" height="951" alt="Basics tab showing VM name, region, and image selection" src="https://github.com/user-attachments/assets/8bc3d40a-0529-4f80-9f17-1e6fbf4b368a" />
<img width="1749" height="945" alt="Basics tab showing VM size selection and authentication type" src="https://github.com/user-attachments/assets/d0b77cc1-9876-4330-a833-ad82eb1ab5a5" />
<img width="1747" height="943" alt="Basics tab showing SSH key configuration and inbound port rules" src="https://github.com/user-attachments/assets/69e781e6-d20c-42cb-9924-7e36faa93c59" />

---

### Step 4: Configure Networking

Navigate to the **Networking** tab. Configure the following:

- **Virtual Network:** Use the default VNet auto-created for the resource group, or select an existing one
- **Subnet:** Use the default subnet
- **Public IP:** Allow Azure to create a new public IP (required for direct SSH access)
- **NIC Network Security Group:** Select **"Basic"**
- **Public inbound ports:** `Allow selected ports`
- **Select inbound ports:** `SSH (22)`
- **Accelerated Networking:** Disabled (not supported on `Standard_B1s`)
- **Load balancing:** None

> **Operational Note:** The NSG is the primary network access control layer in Azure. Inbound rules at the NSG level take precedence over in-VM firewall configurations. For production workloads, scope SSH access to known CIDR ranges rather than allowing `0.0.0.0/0` (any source).

> **Edge Case:** If a VNet does not exist in the selected region, the portal will prompt to create one. Ensure the address space does not overlap with existing peered networks or on-premises ranges if VNet peering or VPN connectivity is planned.

**Screenshots: Networking tab configuration**
<img width="1783" height="943" alt="Networking tab showing VNet and subnet selection" src="https://github.com/user-attachments/assets/77949321-80e8-4872-88b0-ba7574d026ea" />
<img width="1760" height="947" alt="Networking tab showing public IP and NIC NSG settings" src="https://github.com/user-attachments/assets/29d18b57-43f2-46d6-9c7a-8a99fd67a63d" />
<img width="1758" height="948" alt="Networking tab showing inbound port rule for SSH" src="https://github.com/user-attachments/assets/748f9958-3c5a-46a9-9ee7-45ef484f5563" />
<img width="1734" height="951" alt="Networking tab showing accelerated networking and load balancing settings" src="https://github.com/user-attachments/assets/92942e70-9fd8-4a72-95a4-7df876050060" />

---

### Step 5: Configure Disk

Navigate to the **Disks** tab. Configure the following:

- **OS Disk Size:** `30 GiB` (default or custom)
- **OS Disk Type:** `Standard HDD (LRS)`
- **Delete with VM:** Enabled (recommended for non-persistent lab and non-critical workloads)
- **Encryption:** Platform-managed keys (default)

> **Operational Note:** Standard HDD is appropriate for dev/test or low-IOPS workloads. For production workloads requiring consistent disk throughput (such as databases or high-traffic web servers), upgrade to **Premium SSD** or **Ultra Disk** depending on latency requirements. Changing disk type post-deployment requires VM deallocation.

> **Edge Case:** The minimum OS disk size enforced by the image may differ from the specified 30 GB. Verify the image's minimum disk size requirement during provisioning. Azure will prevent deployment if the specified size is below the image minimum.

**Screenshot: Disk configuration tab**
<img width="1760" height="951" alt="Disks tab showing OS disk type set to Standard HDD and disk size set to 30 GiB" src="https://github.com/user-attachments/assets/f5be68e5-085e-4ee5-99c4-b913e6e86175" />

---

### Step 6: Review and Deploy

Leave all remaining tabs (**Management**, **Monitoring**, **Advanced**, **Tags**) at their default values unless organizational policy requires specific configurations (e.g., enabling boot diagnostics, assigning tags for cost allocation).

Navigate to the **Review + Create** tab. Azure performs an automatic validation pass against the configured parameters.

**Validation must pass before deployment proceeds.** If validation fails, review the error details and correct the affected configuration tab before retrying.

Once validation passes:

1. Click **"Create"**
2. On the **"Generate new key pair"** dialog, click **"Download private key and create resource"**
3. Save the `.pem` file securely
4. Monitor the deployment progress on the portal until all resources show **"Succeeded"** status

> **Risk:** If the browser window or session is closed during deployment, the private key download prompt is not recoverable. Always complete the key download before navigating away.

**Screenshots: Review, validation, and deployment progression**
<img width="1792" height="947" alt="Review + Create tab showing validation in progress" src="https://github.com/user-attachments/assets/808b1bd5-23ff-45d2-bad4-0babc2e0dda0" />
<img width="1754" height="954" alt="Validation passed confirmation on Review + Create tab" src="https://github.com/user-attachments/assets/0d03d98a-ba47-4622-bfc8-ed42c05d8a08" />
<img width="1762" height="951" alt="Generate new key pair dialog with download prompt" src="https://github.com/user-attachments/assets/87f58cf9-271b-486c-8bfb-58b94e25a516" />
<img width="1756" height="945" alt="Deployment in progress screen showing resource creation status" src="https://github.com/user-attachments/assets/c64c2007-d8a1-44dd-a519-0a2e26b3f1e1" />
<img width="1870" height="952" alt="Deployment progress showing VM and associated resources being created" src="https://github.com/user-attachments/assets/b93cbadc-5683-4966-ac97-87888c3ca912" />
<img width="1779" height="947" alt="Deployment succeeded confirmation with resource links" src="https://github.com/user-attachments/assets/b761740b-30b5-4eb2-8f4e-3ff9d63ed9c7" />
<img width="1917" height="904" alt="VM overview page showing running state and public IP address" src="https://github.com/user-attachments/assets/7ccf9e53-b4a8-44d6-9c8b-424b2666d618" />
<img width="1797" height="947" alt="VM resource overview confirming name, region, size, and OS details" src="https://github.com/user-attachments/assets/42c71886-3210-4224-9fd6-de1305170fb8" />

---

### Step 7: SSH Verification

After deployment completes, retrieve the VM's **Public IP Address** from the VM overview page. Use the downloaded private key to authenticate via SSH.

Set correct permissions on the private key file and connect:

```bash
chmod 400 xfusion-vm_key.pem
ssh -i xfusion-vm_key.pem azureuser@<PUBLIC_IP_ADDRESS>
```

A successful login confirms:

- The VM is running and reachable over the public internet on port 22
- The NSG inbound rule for SSH is correctly applied
- The SSH key pair was correctly generated and associated with the VM
- The OS has fully booted and the SSH daemon is active

> **Troubleshooting:** If the SSH connection times out, verify that:
> 1. The NSG inbound rule for port 22 is applied to the correct network interface
> 2. The VM is in a **"Running"** state (not deallocated or stopped)
> 3. The public IP is correctly copied from the portal (no trailing spaces)
> 4. The private key file permissions are set to `400` (world-readable keys are rejected by SSH clients)

**Screenshots: SSH private key download and successful terminal connection**
<img width="1829" height="949" alt="Azure portal prompt to download the SSH private key before VM creation completes" src="https://github.com/user-attachments/assets/fb929034-c46b-43da-9585-765f3d847868" />
<img width="1031" height="875" alt="Terminal window showing successful SSH login to xfusion-vm as azureuser" src="https://github.com/user-attachments/assets/e852c320-8da5-4c9c-9c6f-dbc18400b2e6" />

---

## Validation Checklist

| Check | Expected Value | Status |
|---|---|---|
| VM name | `xfusion-vm` | Verify in portal |
| Region | `centralus` | Verify in VM overview |
| OS image | Ubuntu Server 22.04 LTS | Verify in VM overview |
| VM size | `Standard_B1s` | Verify in VM overview |
| OS disk type | Standard HDD | Verify in Disks blade |
| OS disk size | 30 GiB | Verify in Disks blade |
| NSG SSH rule | Port 22 inbound allowed | Verify in Networking blade |
| SSH connectivity | Successful login as admin user | Verify via terminal |

---

## Key Decisions

- **Standard HDD over SSD:** Chosen to align with cost constraints for this workload tier. Acceptable given that the VM is not running I/O-intensive applications. For production compute, Premium SSD should be evaluated.
- **Basic NSG vs. Advanced:** Basic NSG provides sufficient access control for this implementation. Advanced NIC NSG configuration would be required if multiple NICs or subnet-level segmentation is needed.
- **Default VNet:** Using the auto-generated VNet simplifies provisioning for isolated workloads. Future implementations connecting to shared infrastructure should use a pre-planned VNet with defined address spaces to avoid CIDR conflicts.
- **Portal UI vs. CLI/IaC:** The Portal was used to enable visual validation of each configuration step. For repeatable or automated provisioning, ARM templates, Bicep, or Terraform should be used to eliminate manual configuration risk.

---

## Risks, Edge Cases, and Troubleshooting

**Private key loss:**
The `.pem` file is downloadable only once during VM creation. If lost, SSH access cannot be recovered via the standard key mechanism. Mitigation options include resetting the SSH key via the Azure Portal (VM blade > "Reset password") or using the Azure Serial Console for emergency access.

**Region quota exhaustion:**
`Standard_B1s` quota may be exhausted in high-demand regions. If deployment fails with a quota error, request a quota increase via the Subscriptions blade or redeploy to an alternate region.

**NSG misconfiguration:**
Assigning an NSG to a subnet rather than the NIC may produce unexpected results in environments with overlapping rules. Always verify effective security rules via the VM's **"Networking"** blade under **"Effective security rules"**.

**VM size unavailability:**
Certain VM sizes may not be available in all regions. If `Standard_B1s` is unavailable in the selected region, the portal will surface an error during validation. Resolve by selecting an available size or switching to an alternative region.

**SSH connection refused (port 22 not reachable):**
This typically indicates the NSG rule was not correctly applied, the VM is still booting, or the SSH daemon encountered an error on first boot. Wait 60 to 90 seconds after deployment completes before attempting the initial SSH connection.

---

## Best Practices and Operational Considerations

- **Tag all resources** at creation time with environment, owner, and project identifiers to support cost allocation and governance at scale.
- **Enable boot diagnostics** for all production VMs to facilitate troubleshooting of OS-level boot failures via the Azure Serial Console.
- **Restrict SSH source IPs** in the NSG inbound rule to known CIDR ranges rather than `0.0.0.0/0`. This is a critical security hardening step for production environments.
- **Store private keys in a secrets manager** (such as Azure Key Vault) immediately after download. Do not store keys in version control, shared drives, or unencrypted local directories.
- **Use managed identities** instead of SSH key-based access for VM-to-service authentication in production workflows.
- **Plan disk sizing upfront.** Expanding an OS disk post-deployment requires VM deallocation and manual filesystem extension. Over-provisioning slightly at creation is preferable to emergency resizing under load.
- **Automate provisioning** using Terraform, Bicep, or ARM templates for any VM configuration that will be replicated across environments. Portal-based provisioning does not produce auditable, version-controlled infrastructure definitions.

---

## Lessons Learned

- **NSG rules control access at the network boundary, not at the VM level.** An OS-level firewall (`ufw`, `iptables`) may be inactive by default on Azure Linux images. Relying solely on NSG rules is acceptable for basic access control, but layered defense requires in-VM firewall configuration as well.
- **SSH verification is a mandatory delivery gate**, not an optional post-deployment step. A VM that is deployed but unreachable over SSH has not been successfully provisioned from an operational standpoint.
- **Region selection has downstream implications.** Choosing `East US` versus `Central US` affects latency, available VM sizes, compliance boundaries, and potential peering costs. Always confirm the target region with the infrastructure or platform team before provisioning.
- **Key pair management discipline must be established before first deployment.** Retrofitting secure key storage practices after provisioning creates operational risk during the gap period.

---

## Conclusion

This document provides a complete, validated walkthrough for provisioning an Azure Virtual Machine via the Portal UI, from initial configuration through SSH connectivity verification. The implementation demonstrates core Azure compute provisioning skills including VM sizing, disk selection, network security group configuration, and key-based authentication.

The provisioned `xfusion-vm` serves as a validated foundational compute resource for subsequent infrastructure workloads within the Nautilus DevOps migration program. For repeatable provisioning at scale, the patterns established here should be codified into Infrastructure as Code using Terraform or Bicep.


ccess an Azure Virtual Machine using best practices, forming a core building block for cloud infrastructure migration.
