# Azure VM Provisioning with Standard Static Public IP via CLI

> **Provisioning an Azure Virtual Machine with a persistent Standard SKU Static Public IP using Azure CLI -- following infrastructure-as-code principles for repeatable, auditable, and production-aligned deployments.**

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Step 1: Verify Existing Resource Group](#step-1-verify-existing-resource-group)
- [Step 2: Generate SSH Key Pair for VM Access](#step-2-generate-ssh-key-pair-for-vm-access)
- [Step 3: Retrieve Azure Environment Credentials](#step-3-retrieve-azure-environment-credentials)
- [Step 4: Create a Standard Static Public IP Address](#step-4-create-a-standard-static-public-ip-address)
- [Step 5: Create the Azure Virtual Machine](#step-5-create-the-azure-virtual-machine)
- [Step 6: Validate SSH Access to the VM](#step-6-validate-ssh-access-to-the-vm)
- [Step 7: Add an Additional SSH Key (Access Hardening)](#step-7-add-an-additional-ssh-key-access-hardening)
- [Step 8: Revalidate SSH Access After Key Update](#step-8-revalidate-ssh-access-after-key-update)
- [Step 9: Confirm Static Public IP Persistence](#step-9-confirm-static-public-ip-persistence)
- [Final State Summary](#final-state-summary)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Lessons Learned](#lessons-learned)

---

## Overview

This document details the end-to-end provisioning of an **Azure Virtual Machine** with a **Standard SKU Static Public IP address**, deployed entirely via the **Azure CLI**. The implementation demonstrates infrastructure-as-code principles commonly adopted in enterprise DevOps workflows -- ensuring repeatability, auditability, and alignment with production security standards.

The deployment delivers:

- A Ubuntu 22.04 LTS VM (`devops-vm`) in the **Central US** region
- A Standard SKU Static Public IP (`devops-pip`) attached at creation time
- Key-based SSH access using a dedicated RSA key pair
- A secondary SSH key injected post-deployment to demonstrate key rotation capability

---

## Problem Statement

Dynamic public IP allocation is the default behavior for Azure VMs. In production workloads, this creates operational risk: a VM restart or reallocation can change the public IP, breaking DNS records, firewall allow-lists, monitoring integrations, and application-level connectivity dependencies.

**Solution:** Provision a **Standard SKU Static Public IP** before VM creation and explicitly attach it at deployment time, guaranteeing IP persistence across VM restarts, stop/deallocate cycles, and maintenance events.

---

## Architecture Summary

| Component | Value |
|---|---|
| Cloud Provider | Microsoft Azure |
| Region | Central US |
| Compute | Azure Virtual Machine (Standard_B1s) |
| OS Image | Ubuntu 22.04 LTS |
| Networking | Standard SKU Static Public IP |
| Authentication | SSH key-based (RSA 2048-bit) |
| Deployment Method | Azure CLI |

---

## Prerequisites

- Azure CLI installed and authenticated on the local or jump-host machine
- Valid Azure subscription with contributor-level access
- Access to `azure-client` host (or equivalent CLI environment)
- SSH client available in the terminal session

---

## Step 1: Verify Existing Resource Group

Before provisioning any resources, confirm the target resource group exists and is in a healthy state. This prevents deployment failures caused by targeting a non-existent or failed resource group.

```bash
az group list --output table
```

**Expected Outcome:**

- Resource group is listed with `Status: Succeeded`
- Region reflects the intended deployment location

> **Operational Note:** In sandbox and lab environments, resource groups are pre-provisioned. Attempting to create a new resource group in these environments will result in a permission error. Always validate the existing group before proceeding.

**Screenshot: Resource Group Validation**

![Resource Group Validation](https://github.com/user-attachments/assets/fec68cb4-d0d9-4da0-bd24-0f6f4548f329)

*`az group list --output table` confirms resource group `kml_rg_main-6b2c2be9b65c46eb` exists in `eastus` with `Status: Succeeded`.*

---

## Step 2: Generate SSH Key Pair for VM Access

Create a dedicated RSA 2048-bit SSH key pair scoped exclusively to this VM. Using a named, purpose-specific key (rather than the default `~/.ssh/id_rsa`) isolates access and simplifies key lifecycle management.

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/devops-key -N ""
```

**Artifacts Created:**

- **Private Key:** `~/.ssh/devops-key` (never shared, never committed)
- **Public Key:** `~/.ssh/devops-key.pub` (uploaded to the VM during provisioning)

> **Security Note:** The `-N ""` flag sets an empty passphrase, appropriate for automated pipeline access. For interactive use in production, always protect private keys with a passphrase and store them in a secrets manager or hardware security module.

**Screenshot: SSH Key Generation**

![SSH Key Generation](https://github.com/user-attachments/assets/65a92e57-f870-4de3-8159-8a869ccb5903)

*RSA 2048-bit key pair generated successfully. Public key saved to `/root/.ssh/devops-key.pub` and private key to `/root/.ssh/devops-key`. The randomart fingerprint confirms key uniqueness.*

---

## Step 3: Retrieve Azure Environment Credentials

Confirm the active Azure authentication context. This validates that the CLI session is operating under the correct subscription and user identity before any resource provisioning begins.

```bash
showcreds
```

**Validated Information:**

- Azure Portal URL
- Authenticated username
- Active subscription and session window

> **Operational Note:** Always verify the active credential context before provisioning, particularly in shared or multi-subscription environments. An incorrect subscription context can result in resources being deployed to the wrong environment with potential cost and security implications.

**Screenshot: Azure Credentials Validation**

![Azure Credentials](https://github.com/user-attachments/assets/546df4c2-9d37-4ec0-844f-01cc1b6e13b0)

*`showcreds` confirms the active Azure session including portal URL, username, Application Client ID, and session expiry.*

---

## Step 4: Create a Standard Static Public IP Address

Provision the public IP **before** creating the VM. This decouples the IP resource from the VM lifecycle, allowing it to persist independently and be reassigned if the VM is replaced.

```bash
az network public-ip create \
  --resource-group <RESOURCE_GROUP> \
  --name devops-pip \
  --location centralus \
  --allocation-method Static \
  --sku Standard
```

**Key Properties Confirmed in Output:**

- `publicIPAllocationMethod`: `Static`
- `publicIPAddressVersion`: `IPv4`
- `sku.name`: `Standard`
- `provisioningState`: `Succeeded`
- `ipAddress`: `52.176.21.19`

> **SKU Note:** Standard SKU is required for zone-redundant and zone-specific deployments. Basic SKU public IPs are being retired and do not support association with Standard Load Balancers or Virtual Machine Scale Sets. Always use Standard SKU for production workloads.

> **Breaking Change Advisory:** The Azure CLI output includes a notice that in a coming release, Standard SKU IPs in zonal regions will default to zone-redundant allocation (`zones: ["1","2","3"]`). For non-zonal regions, allocation remains `zones: null`. Explicitly specifying the zone at creation time is recommended to avoid unintended behavior changes post-release.

**Screenshot: Static Public IP Creation**

![Public IP Creation](https://github.com/user-attachments/assets/f78d97be-84a3-41c0-b9dc-81e795ecf825)

*`az network public-ip create` provisions `devops-pip` with `Static` allocation and `Standard` SKU in `centralus`. IP `52.176.21.19` is immediately assigned and confirmed in the JSON output.*

---

## Step 5: Create the Azure Virtual Machine

Deploy the VM with the static IP explicitly attached at creation time. Associating the IP during provisioning rather than post-creation avoids a window during which the VM may be accessible via a transient dynamic IP.

```bash
az vm create \
  --resource-group <RESOURCE_GROUP> \
  --name devops-vm \
  --location centralus \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/devops-key.pub \
  --public-ip-address devops-pip \
  --public-ip-sku Standard \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS \
  --nsg-rule SSH
```

**Provisioning Results Confirmed in Output:**

- `powerState`: `VM running`
- `publicIpAddress`: `52.176.21.19` (matches the pre-created static IP)
- `privateIpAddress`: `10.0.0.4`
- `macAddress`: `00-0D-3A-94-FD-B9`
- `resourceGroup`: confirmed target group

> **Flag Reference:**
> - `--nsg-rule SSH`: Creates an NSG allowing inbound TCP/22 from any source. Restrict to known IP ranges in production.
> - `--storage-sku Standard_LRS`: Locally redundant storage for the OS disk.
> - `--os-disk-size-gb 30`: Minimum recommended size; increase as needed for workload data.
> - `--public-ip-sku Standard`: Must match the SKU of the pre-created IP resource to avoid attachment failure.

**Screenshot: VM Creation Output**

![VM Creation Output](https://github.com/user-attachments/assets/056ba576-455c-466f-96b9-80bbbcef4e61)

*`az vm create` returns the full provisioning result. `powerState: VM running` and `publicIpAddress: 52.176.21.19` confirm the VM is live and the static IP is correctly attached.*

---

## Step 6: Validate SSH Access to the VM

Test end-to-end connectivity using the private key generated in Step 2 against the static IP confirmed in Step 5. This validates that the key pair is correctly installed, the NSG allows inbound SSH, and the VM is reachable.

```bash
ssh -i ~/.ssh/devops-key -o StrictHostKeyChecking=no azureuser@52.176.21.19
```

**Verification Checks:**

- Successful login without password prompt
- Ubuntu 22.04 LTS MOTD banner displayed
- VM hostname resolves to `devops-vm`
- System load, memory usage, and disk usage metrics visible
- Internal IPv4 address `10.0.0.4` confirmed on `eth0`

> **Operational Note:** `StrictHostKeyChecking=no` suppresses the host key verification prompt on first connection. In production pipelines, use `ssh-keyscan` to pre-populate `~/.ssh/known_hosts` rather than disabling host key checking globally.

**Screenshot: Initial SSH Login**

![Initial SSH Login](https://github.com/user-attachments/assets/770ff500-1fe4-4d55-92ca-ec13d1e5a1dc)

*SSH login to `azureuser@52.176.21.19` succeeds using the dedicated private key. Ubuntu 22.04 LTS MOTD confirms the VM is operational. Internal IP `10.0.0.4` on `eth0` is visible in system metrics.*

---

## Step 7: Add an Additional SSH Key (Access Hardening)

Generate a second RSA key pair and inject the public key into the VM's authorized keys for `azureuser`. This demonstrates key rotation and multi-key access patterns, which are standard in enterprise environments where access is shared across team members or automated systems.

**Generate the secondary key pair:**

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
```

**Inject the public key into the VM user profile:**

```bash
az vm user update \
  --resource-group <RESOURCE_GROUP> \
  --name devops-vm \
  --username azureuser \
  --ssh-key-value ~/.ssh/id_rsa.pub
```

**Purpose:**

- Enables seamless key rotation without disrupting active sessions
- Supports multiple engineers or automated systems accessing the same VM independently
- Demonstrates the `VMAccessForLinux` extension deployment, which Azure uses to update VM user configurations post-provisioning

> **Extension Note:** `az vm user update` deploys the `VMAccessForLinux` extension (`Microsoft.OSTCExtensions`) internally. The `provisioningState: Succeeded` in the output confirms the key was successfully written to `~/.ssh/authorized_keys` on the VM.

**Screenshot: Secondary Key Generation**

![Secondary SSH Key Generation](https://github.com/user-attachments/assets/f39f547d-2fee-43cf-aba1-ac8d80e74a9f)

*Second RSA 2048-bit key pair generated at `/root/.ssh/id_rsa` and `/root/.ssh/id_rsa.pub`.*

**Screenshot: VM User SSH Key Update**

![VM User SSH Key Update](https://github.com/user-attachments/assets/1ef78f27-88ad-417c-96c3-6fb328fca3bb)

*`az vm user update` injects the secondary public key via the `VMAccessForLinux` extension. `provisioningState: Succeeded` confirms the key is written to the VM's `authorized_keys`.*

---

## Step 8: Revalidate SSH Access After Key Update

Confirm that SSH access remains uninterrupted after the user key update. This validates that the key injection did not modify or replace existing keys, and that both access paths remain active simultaneously.

```bash
ssh azureuser@52.176.21.19
```

**Expected Outcome:**

- SSH session established without password authentication
- `Last login` timestamp reflects the previous session from Step 6
- System MOTD and metrics confirm VM operational state is unchanged

> **Validation Logic:** If the key update had accidentally overwritten the original `authorized_keys` entry, this step would fail. Successful login confirms the `VMAccessForLinux` extension appends rather than replaces authorized keys.

**Screenshot: Post-Update SSH Login**

![Post-Update SSH Login](https://github.com/user-attachments/assets/770ff500-1fe4-4d55-92ca-ec13d1e5a1dc)

*SSH session re-established after the key update. `Last login` timestamp and `10.0.0.4` private IP confirm continuous access and unmodified VM state.*

---

## Step 9: Confirm Static Public IP Persistence

Explicitly verify that the public IP address has not changed throughout the deployment lifecycle. This is the definitive validation that the static allocation is functioning as intended.

```bash
az network public-ip show \
  --resource-group <RESOURCE_GROUP> \
  --name devops-pip \
  --query "{Name:name, IPAddress:ipAddress, Allocation:publicIPAllocationMethod}" \
  --output table
```

**Validation Result:**

| Name | IPAddress | Allocation |
|---|---|---|
| devops-pip | 52.176.21.19 | Static |

- IP address `52.176.21.19` is unchanged from initial allocation in Step 4
- `Allocation: Static` confirmed post-VM-creation and post-SSH-session

> **Production Implication:** This IP can now be safely referenced in DNS A records, firewall rules, monitoring agents, and application configuration without risk of address change across VM lifecycle events.

**Screenshot: Static IP Persistence Verification**

![Static Public IP Verification](https://github.com/user-attachments/assets/df63d106-0926-4e45-afe8-c217d784c2f5)

*`az network public-ip show` query confirms `devops-pip` retains IP `52.176.21.19` with `Static` allocation after the full deployment sequence. The `exit` and reconnect cycle visible in the terminal further validates persistence across session boundaries.*

---

## Final State Summary

| Component | Status |
|---|---|
| Virtual Machine | Running |
| OS | Ubuntu 22.04 LTS |
| VM Size | Standard_B1s |
| Public IP | 52.176.21.19 (Static, Standard SKU) |
| Private IP | 10.0.0.4 |
| SSH Access | Key-based (multi-key configured) |
| Region | Central US |
| Resource Group | kml_rg_main-6b2c2be9b65c46eb |

---

## Key Decisions

| Decision | Rationale |
|---|---|
| **Standard SKU over Basic SKU** | Standard SKU is required for zone-redundant deployments, Load Balancer integration, and future-proofing against Basic SKU retirement |
| **Pre-create IP before VM** | Decouples IP lifecycle from VM lifecycle; IP persists even if VM is deleted and recreated |
| **Named SSH key (`devops-key`) over default** | Isolates VM-specific credentials; simplifies rotation without impacting other resources using the default key |
| **`--nsg-rule SSH` at creation** | Automatically creates an NSG with TCP/22 inbound; avoids a no-access state immediately after provisioning |
| **`Standard_B1s` size** | Minimum viable compute for a Linux workload; appropriate for validation and development-tier deployments |
| **Secondary key injection via `az vm user update`** | Validates `VMAccessForLinux` extension behavior and demonstrates non-disruptive key rotation |

---

## Best Practices and Operational Considerations

- **Restrict NSG inbound SSH to known IP ranges.** The `--nsg-rule SSH` flag opens TCP/22 from `0.0.0.0/0` by default. In production, immediately update the NSG rule to allow only specific source CIDRs (e.g., VPN gateway ranges, jump-host IPs, or CI/CD runner IPs).

- **Store private keys in a secrets manager.** Keys stored in `~/.ssh` on a shared host are accessible to any user with filesystem access. Use Azure Key Vault, HashiCorp Vault, or an equivalent secrets management system for production private key storage.

- **Tag all resources at provisioning time.** Add `--tags` to all `az` commands for cost allocation, ownership tracking, and automated governance enforcement.

- **Use `--no-wait` for long-running operations in pipelines.** For CI/CD workflows, combine `--no-wait` with `az vm wait --created` to parallelize provisioning steps.

- **Validate IP persistence after stop/deallocate cycles.** While Standard Static IPs persist across restarts, confirm behavior after `az vm deallocate` in your specific environment -- particularly in environments with subscription-level policies that may override standard behavior.

- **Prefer `~/.ssh/config` for multi-VM access management.** Define host aliases, key paths, and usernames in `~/.ssh/config` to simplify SSH commands and reduce the risk of using the wrong key for the wrong host.

---

## Lessons Learned

- **Static IP must be created before VM provisioning** if IP attachment at creation time is required. Attempting to attach a Basic SKU IP to a Standard SKU VM -- or mismatching SKUs between the `--public-ip-address` and `--public-ip-sku` flags -- results in a provisioning error.

- **`VMAccessForLinux` appends, not replaces, authorized keys.** The `az vm user update` command adds the new public key to `~/.ssh/authorized_keys` without removing existing entries. This is critical to understand before executing key rotation in production to avoid accidental lockout.

- **Resource group creation is restricted in sandbox environments.** The resource group is pre-created by the lab environment. Any attempt to create a new resource group results in a permission error. Always list existing groups first and target the pre-provisioned one.

- **The `--location` flag must match the region of the resource group.** Mismatched regions between the resource group and child resources cause deployment failures. Confirm the resource group region with `az group list` before specifying `--location` in downstream commands.

- **Standard SKU IPs are always zone-aware.** The CLI warning about upcoming default zone behavior is a reminder that explicitly specifying `--zone` at IP creation time is the safest approach for predictable zone placement in production deployments.






























# Azure VM Deployment with Static Public IP (DevOps Lab)

## Project Overview

This project documents the end-to-end provisioning of an **Azure Virtual Machine** with a **Standard Static Public IP address** to guarantee stable and consistent external access for application workloads. The deployment was executed entirely via the **Azure CLI**, following infrastructure-as-code principles commonly adopted by FAANG-scale DevOps teams.

The final setup delivers:
- A Ubuntu-based VM (`devops-vm`)
- A Standard SKU Static Public IP (`devops-pip`)
- Secure SSH access using public key authentication
- Deployment scoped to the **Central US** Azure region

---

## Architecture Summary

- **Cloud Provider:** Microsoft Azure  
- **Region:** Central US  
- **Compute:** Azure Virtual Machine (Standard_B1s)  
- **OS Image:** Ubuntu 22.04 LTS  
- **Networking:** Standard Static Public IP  
- **Access Method:** SSH (Key-based authentication)

---

## Prerequisites

- Azure CLI installed and authenticated
- Valid Azure subscription credentials
- Access to `azure-client` host
- SSH client available

---

## Step 1: Verify Existing Resource Group

Validate the target resource group to ensure it is available and healthy.

```
az group list --output table
```
### Expected Outcome

- Resource group exists

- Status shows Succeeded

Screenshot: `Resource Group Validation Output`
<img width="1030" height="622" alt="image" src="https://github.com/user-attachments/assets/fec68cb4-d0d9-4da0-bd24-0f6f4548f329" />

## Step 2: Generate SSH Key Pair for VM Access

- Create a dedicated SSH key pair to securely access the VM.
```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/devops-key -N ""
```

Artifacts Created
```
Private Key: ~/.ssh/devops-key

Public Key: ~/.ssh/devops-key.pub
```
Screenshot: `SSH Key Generation Output`
<img width="1031" height="634" alt="image" src="https://github.com/user-attachments/assets/65a92e57-f870-4de3-8159-8a869ccb5903" />

## Step 3: Retrieve Azure Environment Credentials

- Confirm Azure authentication context using provided lab credentials.
```
showcreds
```
- Validated Information

- Azure Portal URL

- Username

- Active session window

Screenshot: `Azure Credentials Output`


<img width="968" height="278" alt="556260051-de7cfe57-b167-403f-8047-dd1bdced1ccc" src="https://github.com/user-attachments/assets/546df4c2-9d37-4ec0-844f-01cc1b6e13b0" />

## Step 4: Create a Standard Static Public IP Address

- Provision a Standard SKU Static Public IP to ensure IP persistence across VM restarts.
```
az network public-ip create \
  --resource-group <RESOURCE_GROUP> \
  --name devops-pip \
  --location centralus \
  --allocation-method Static \
  --sku Standard
```

### Key Properties

- Allocation Method: `Static`

- IP Version: `IPv4`

- SKU: `Standard`

Screenshot: `Public IP Creation Output`
<img width="1032" height="709" alt="image" src="https://github.com/user-attachments/assets/f78d97be-84a3-41c0-b9dc-81e795ecf825" />

## Step 5: Create the Azure Virtual Machine

- Deploy the VM and explicitly associate it with the previously created static public IP.
```
az vm create \
  --resource-group <RESOURCE_GROUP> \
  --name devops-vm \
  --location centralus \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/devops-key.pub \
  --public-ip-address devops-pip \
  --public-ip-sku Standard \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS \
  --nsg-rule SSH
```
- Provisioning Results

- VM successfully created

- Public IP attached at creation time

- SSH allowed via Network Security Group

Screenshot: `VM Creation Output`
<img width="1037" height="862" alt="image" src="https://github.com/user-attachments/assets/056ba576-455c-466f-96b9-80bbbcef4e61" />

## Step 6: Validate SSH Access to the VM

- Test secure connectivity using the generated SSH private key.
```
ssh -i ~/.ssh/devops-key azureuser@<STATIC_PUBLIC_IP>
```
- Verification Checks

- Successful login

- Ubuntu 22.04 LTS banner visible

- VM hostname resolves correctly

Screenshot: `Initial SSH Login to VM`


## Step 7: Add an Additional SSH Key (User Access Hardening)

- Generate a secondary SSH key and update the VM user profile.
```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
az vm user update \
  --resource-group <RESOURCE_GROUP> \
  --name devops-vm \
  --username azureuser \
  --ssh-key-value ~/.ssh/id_rsa.pub
```

### Purpose

- Enables key rotation

- Supports multiple secure access paths

Screenshots: `VM User SSH Key Update`
<img width="1030" height="409" alt="image" src="https://github.com/user-attachments/assets/f39f547d-2fee-43cf-aba1-ac8d80e74a9f" />
<img width="1037" height="576" alt="image" src="https://github.com/user-attachments/assets/1ef78f27-88ad-417c-96c3-6fb328fca3bb" />

## Step 8: Revalidate SSH Access Using Updated Configuration

- Confirm uninterrupted access after user key update.

- ssh azureuser@<STATIC_PUBLIC_IP>

### Expected Outcome

- SSH access remains functional

- No password authentication required

Screenshot: `Post-Update SSH Login`
<img width="1028" height="845" alt="image" src="https://github.com/user-attachments/assets/770ff500-1fe4-4d55-92ca-ec13d1e5a1dc" />

## Step 9: Confirm Static Public IP Persistence

- Verify that the public IP remains static and unchanged.
```
az network public-ip show \
  --resource-group <RESOURCE_GROUP> \
  --name devops-pip \
  --query "{Name:name, IPAddress:ipAddress, Allocation:publicIPAllocationMethod}" \
  --output table
```

### Validation Result

- Allocation Method: `Static`

- IP Address unchanged

Screenshot: `Static Public IP Verification`
<img width="1036" height="859" alt="image" src="https://github.com/user-attachments/assets/df63d106-0926-4e45-afe8-c217d784c2f5" />


## Final State Summary

| Component       | Status                |
| --------------- | --------------------- |
| Virtual Machine | Running               |
| OS              | Ubuntu 22.04 LTS      |
| VM Size         | Standard_B1s          |
| Public IP       | Static (Standard SKU) |
| SSH Access      | Key-Based             |
| Region          | Central US            |


## Key Takeaways

- Static Public IPs ensure consistent external access for applications

- Standard SKU is required for production-grade workloads

- SSH key-based authentication aligns with DevOps security best practices

- CLI-driven deployments promote repeatability and auditability
