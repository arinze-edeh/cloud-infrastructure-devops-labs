# Azure Network Security Group (NSG) Provisioning and Hardening via Azure CLI

> **Environment:** Microsoft Azure | **Region:** East US | **Auth Method:** Service Principal (Azure CLI)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Security Model](#architecture-and-security-model)
- [Environment Details](#environment-details)
- [Prerequisites](#prerequisites)
- [Step 1: Verify Azure Authentication](#step-1-verify-azure-authentication)
- [Step 2: Identify Target Resource Group](#step-2-identify-target-resource-group)
- [Step 3: Create the Network Security Group](#step-3-create-the-network-security-group)
- [Step 4: Add Inbound Rule - Allow HTTP (Port 80)](#step-4-add-inbound-rule---allow-http-port-80)
- [Step 5: Add Inbound Rule - Allow SSH (Port 22)](#step-5-add-inbound-rule---allow-ssh-port-22)
- [Step 6: Validate NSG Rules](#step-6-validate-nsg-rules)
- [Automation Script](#automation-script)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Outcome](#outcome)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This document details the end-to-end provisioning and configuration of an **Azure Network Security Group (NSG)** using the **Azure CLI** as part of the Nautilus infrastructure migration initiative. The NSG enforces inbound traffic control by explicitly permitting **HTTP (port 80)** and **SSH (port 22)** while relying on Azure's implicit default-deny posture for all other inbound traffic.

The implementation adheres to **cloud-native**, **least-privilege**, and **automation-first** engineering principles, making it suitable for production environments, repeatable deployments, and infrastructure handoff.

---

## Problem Statement

As part of the Nautilus infrastructure migration to Azure, virtual machines and subnets require a consistent, auditable, and programmable network access control layer. Without a properly configured NSG:

- **Inbound traffic is unrestricted** by default in permissive VNet configurations
- **Audit trails and rule management** become fragmented without centralized configuration
- **Manual portal-based configuration** introduces drift, lacks repeatability, and cannot be version-controlled

**Solution:** Programmatically provision an NSG via Azure CLI, define explicit inbound allow rules for required services (HTTP and SSH), and automate the full workflow via a reusable Bash script.

---

## Architecture and Security Model

Azure NSGs operate as stateful Layer 4 packet filters attached to subnets or individual network interfaces (NICs). Rules are evaluated by **priority order** (lower number = higher precedence). Azure always appends three immutable default rules:

| Priority | Name | Action | Purpose |
|----------|------|--------|---------|
| 65000 | AllowVNetInBound | Allow | Permits traffic within the VNet |
| 65001 | AllowAzureLoadBalancerInBound | Allow | Permits Azure Load Balancer health probes |
| 65500 | DenyAllInBound | Deny | Blocks all other inbound traffic |

Custom rules defined in this project are assigned priorities **100** and **110**, ensuring they are evaluated before Azure's default rules.

---

## Environment Details

| Item | Value |
|------|-------|
| Cloud Provider | Microsoft Azure |
| Subscription | Azure Free |
| Region | eastus |
| Resource Group | kml_rg_main-f56fa9d90d7842e1 |
| NSG Name | nautilus-nsg |
| Auth Method | Azure CLI (Service Principal) |

---

## Prerequisites

- **Azure CLI** installed and accessible in `$PATH`
- **Active Azure subscription** with sufficient RBAC permissions (Network Contributor or higher)
- **Authenticated Azure CLI session** via Service Principal or interactive login
- **Bash shell** for executing automation scripts

---

## Step 1: Verify Azure Authentication

**Intent:** Confirm that the CLI session is authenticated to the correct subscription and tenant before executing any resource operations. Running commands against the wrong subscription is a common and costly mistake in multi-tenant environments.

```bash
az account show
```

**Expected output fields to verify:**

- `"state": "Enabled"` confirms the subscription is active
- `"name": "Azure Free"` confirms the correct subscription context
- `"type": "servicePrincipal"` confirms non-interactive, automation-compatible authentication
- `"isDefault": true` confirms this is the active context for all subsequent CLI commands

> **Operational Note:** In environments with multiple subscriptions, always explicitly set context with `az account set --subscription <subscription-id>` before provisioning resources.

**Screenshot: Azure CLI authenticated session displaying active subscription and service principal context**

<img width="1034" height="556" alt="image" src="https://github.com/user-attachments/assets/6665d16e-1a21-4a8a-a45c-f257e0430949" />

---

## Step 2: Identify Target Resource Group

**Intent:** Dynamically resolve the target resource group name to avoid hardcoding values in interactive sessions, reducing configuration errors across environments.

```bash
RG_NAME=$(az group list --query "[0].name" -o tsv)
echo "Your Resource Group is: $RG_NAME"
```

**Expected output:**

```
Your Resource Group is: kml_rg_main-f56fa9d90d7842e1
```

> **Best Practice:** In production pipelines, resource group names should be passed as environment variables or pipeline parameters rather than dynamically queried via index. This prevents unintended deployments if resource group ordering changes.

**Screenshot: Dynamic resource group name resolution via Azure CLI query**

<img width="1033" height="629" alt="image" src="https://github.com/user-attachments/assets/15612c68-a301-48ff-bbdf-7a7899224078" />

---

## Step 3: Create the Network Security Group

**Intent:** Provision the NSG resource within the target resource group. At creation time, Azure automatically attaches three immutable default rules to the NSG. No custom rules are active until explicitly defined in subsequent steps.

```bash
az network nsg create \
  --resource-group $RG_NAME \
  --name nautilus-nsg
```

**Expected result:**

- `"provisioningState": "Succeeded"` confirms successful NSG creation
- `"name": "nautilus-nsg"` confirms correct naming
- `"location": "eastus"` confirms correct region placement
- `"securityRules": []` confirms no custom rules are active yet
- Default rules (`AllowVNetInBound`, `AllowAzureLoadBalancerInBound`, `DenyAllInBound`) are visible under `"defaultSecurityRules"`

> **Operational Note:** NSGs at this stage are not yet associated with any subnet or NIC. They must be explicitly attached after configuration to enforce traffic policies.

**Screenshot: NSG creation command and JSON output showing default security rules provisioned by Azure**

<img width="1034" height="870" alt="image" src="https://github.com/user-attachments/assets/1a4497d3-66a0-4072-b6c0-c4549f39f23f" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/8c18d50c-366a-4d35-87da-34ef00b3c2bc" />
<img width="1034" height="869" alt="image" src="https://github.com/user-attachments/assets/7c5520f2-0b7d-4d10-aace-8abf58acafc4" />
<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/1e462635-2045-4919-9727-2cc31023c396" />

---

## Step 4: Add Inbound Rule - Allow HTTP (Port 80)

**Intent:** Define an explicit inbound allow rule for HTTP traffic on TCP port 80. This enables web server reachability from any source, which is the expected posture for public-facing application tiers. Priority 100 ensures this rule is evaluated first among all custom rules.

```bash
az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-HTTP \
  --priority 100 \
  --destination-port-ranges 80 \
  --access Allow \
  --protocol Tcp
```

**Key rule attributes confirmed in response:**

- `"name": "Allow-HTTP"`
- `"priority": 100`
- `"protocol": "Tcp"`
- `"destinationPortRange": "80"`
- `"direction": "Inbound"`
- `"access": "Allow"`
- `"provisioningState": "Succeeded"`

> **Security Note:** `sourceAddressPrefix: "*"` permits HTTP traffic from any IP address. For internal-only applications, restrict the source to a specific CIDR range or VNet service tag (e.g., `VirtualNetwork`).

**Screenshot: Allow-HTTP NSG rule creation with full JSON response confirming rule parameters and provisioning success**

<img width="1038" height="866" alt="image" src="https://github.com/user-attachments/assets/05009582-c13c-40de-bd1b-ec2d892ddbbc" />

---

## Step 5: Add Inbound Rule - Allow SSH (Port 22)

**Intent:** Define an explicit inbound allow rule for SSH traffic on TCP port 22, enabling secure remote administrative access to virtual machines. Priority 110 places this rule immediately after HTTP, maintaining clear priority ordering.

```bash
az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-SSH \
  --priority 110 \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp
```

**Key rule attributes confirmed in response:**

- `"name": "Allow-SSH"`
- `"priority": 110`
- `"protocol": "Tcp"`
- `"destinationPortRange": "22"`
- `"direction": "Inbound"`
- `"access": "Allow"`
- `"provisioningState": "Succeeded"`

> **Critical Security Warning:** This rule allows SSH from `0.0.0.0/0` (any source). In production environments, SSH access **must** be restricted to trusted CIDR ranges, corporate VPN egress IPs, or Azure Bastion. Exposing port 22 to the internet without IP restrictions is a high-severity security risk and violates most enterprise security baselines.

**Screenshot: Allow-SSH NSG rule creation with JSON response confirming priority 110 and inbound TCP access on port 22**

<img width="1033" height="861" alt="image" src="https://github.com/user-attachments/assets/8b72b5b8-61e0-46d8-9f9a-8c58b39f7724" />

---

## Step 6: Validate NSG Rules

**Intent:** Perform a final authoritative verification of all active custom security rules. This step confirms that both rules were applied with the correct priority, protocol, direction, and access settings before the NSG is associated with any network resource.

```bash
az network nsg rule list \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --output table
```

**Expected output:**

```
Name        ResourceGroup                    Priority  SourcePortRanges  SourceAddressPrefixes  SourceASG  Access  Protocol  Direction
----------  -------------------------------  --------  ----------------  ---------------------  ---------  ------  --------  ---------
Allow-HTTP  kml_rg_main-f56fa9d90d7842e1    100       *                 *                      None       Allow   Tcp       Inbound
Allow-SSH   kml_rg_main-f56fa9d90d7842e1    110       *                 *                      None       Allow   Tcp       Inbound
```

**Validation checklist:**

- `Allow-HTTP` at priority **100** targeting port **80** via TCP, direction **Inbound**
- `Allow-SSH` at priority **110** targeting port **22** via TCP, direction **Inbound**
- No unintended rules present
- Priority ordering is correct and non-conflicting

**Screenshot: Final NSG rule list in tabular format confirming both Allow-HTTP (priority 100) and Allow-SSH (priority 110) are active and correctly configured**

<img width="1191" height="627" alt="image" src="https://github.com/user-attachments/assets/acafbb42-bef3-4e13-ba85-83f4f0fbf464" />

---

## Automation Script

To ensure repeatability, idempotency, and consistency across environments, the entire NSG provisioning workflow was encoded in a reusable Bash automation script. This enables one-command deployment as part of a CI/CD pipeline or infrastructure bootstrapping process.

**File:** `az-nsg-nautilus.sh`

```bash
#!/bin/bash
# Azure Network Security Group Automation Script
# Project: Nautilus Infrastructure Migration

RG_NAME="kml_rg_main-f56fa9d90d7842e1"
NSG_NAME="nautilus-nsg"

echo "Creating NSG..."
az network nsg create -g $RG_NAME -n $NSG_NAME

echo "Adding Inbound Rules..."
az network nsg rule create -g $RG_NAME --nsg-name $NSG_NAME \
  -n Allow-HTTP --priority 100 --destination-port-ranges 80 --access Allow --protocol Tcp

az network nsg rule create -g $RG_NAME --nsg-name $NSG_NAME \
  -n Allow-SSH --priority 110 --destination-port-ranges 22 --access Allow --protocol Tcp

echo "Final Configuration:"
az network nsg rule list -g $RG_NAME --nsg-name $NSG_NAME --output table
```

**Granting executable permissions and verifying script content:**

```bash
chmod +x az-nsg-nautilus.sh
cat az-nsg-nautilus.sh
```

**Screenshot: Script authored using heredoc and saved to az-nsg-nautilus.sh, followed by NSG rule table confirming correct output**

<img width="1183" height="822" alt="image" src="https://github.com/user-attachments/assets/e62d9a05-72c5-4bce-a648-7945093ca168" />

**Screenshot: Script content confirmed via cat after heredoc creation**

<img width="1187" height="436" alt="image" src="https://github.com/user-attachments/assets/d2ad0bab-6fb5-4b7b-8fc3-c0feddb64280" />

**Screenshot: Executable permissions granted via chmod +x and final script content verified**

<img width="1187" height="438" alt="image" src="https://github.com/user-attachments/assets/db576d75-3954-470a-bc18-1a66cd11ac68" />

> **Pipeline Integration Note:** For CI/CD environments, externalize `RG_NAME` and `NSG_NAME` as environment variables or pipeline parameters. Consider using `az deployment group create` with ARM/Bicep templates for full Infrastructure as Code (IaC) compliance and state management.

---

## Security Considerations

**Rule Design**

- Custom rules are assigned priorities **100 and 110**, well below Azure's default rules (65000+), ensuring they are always evaluated first
- Azure's implicit `DenyAllInBound` rule at priority 65500 blocks all traffic not explicitly permitted, enforcing a default-deny posture without requiring an explicit deny rule

**Production Hardening Requirements**

- SSH access (`Allow-SSH`) should be restricted to trusted CIDR ranges or Azure Bastion service, not `0.0.0.0/0`
- HTTP traffic should be fronted by an **Azure Application Gateway** or **Azure Front Door** with WAF policies enabled in production
- NSGs should be applied at the **subnet level** for broad control, with additional NIC-level NSGs for per-VM granularity where required
- Enable **NSG Flow Logs** via Azure Network Watcher for traffic auditing and anomaly detection
- Review and audit NSG rules on a scheduled cadence as part of cloud security posture management (CSPM)

> **Warning:** Exposing SSH port 22 to `0.0.0.0/0` in production is a critical security risk. Restrict source IPs immediately upon production association.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| `az account show` returns wrong subscription | Multiple subscriptions configured | Run `az account set --subscription <id>` to set correct context |
| `az group list` returns empty or wrong group | Subscription mismatch or no groups in region | Verify subscription with `az account show`; list all groups with `az group list -o table` |
| NSG creation fails with authorization error | Insufficient RBAC permissions | Ensure the service principal has **Network Contributor** role on the resource group |
| Rule creation fails with priority conflict | Duplicate priority value | List existing rules with `az network nsg rule list` and choose an unused priority |
| Rules show in list but traffic is still blocked | NSG not associated with subnet or NIC | Associate NSG: `az network vnet subnet update --nsg nautilus-nsg` |
| SSH blocked despite Allow-SSH rule | Host-level firewall (iptables/ufw) blocking port 22 | Verify OS-level firewall allows inbound SSH on the VM |

---

## Outcome

- Network Security Group **nautilus-nsg** successfully provisioned in **eastus**
- Inbound HTTP (port 80) access explicitly permitted at **priority 100**
- Inbound SSH (port 22) access explicitly permitted at **priority 110**
- All other inbound traffic implicitly denied by Azure default rules
- Configuration validated via `az network nsg rule list --output table`
- Full workflow automated and packaged in `az-nsg-nautilus.sh` for repeatable deployment
- NSG is ready for subnet or NIC association to enforce traffic policies

---

## Skills Demonstrated

- **Azure Networking Fundamentals** - NSG architecture, rule priority model, default deny posture
- **Azure CLI Proficiency** - Resource provisioning, querying, and validation via CLI
- **Infrastructure as Code Mindset** - Bash automation, parameterized scripts, repeatable deployments
- **Secure Network Design** - Least-privilege access, explicit allow rules, production hardening awareness
- **Operational Documentation** - Enterprise-style handoff documentation with validation and troubleshooting guidance






























# Azure Network Security Group (NSG) - HTTP & SSH Access

## Overview
- This project demonstrates the creation and configuration of an **Azure Network Security Group (NSG)** using the **Azure CLI** as part of the Nautilus infrastructure migration strategy.  
- The NSG enforces inbound access control by explicitly allowing **HTTP (port 80)** and **SSH (port 22)** traffic while relying on Azure’s default deny rules for all other inbound traffic.

- The implementation follows **cloud-native, least-privilege, and automation-first** principles.

---

## Environment Details

| Item           | Value                         |
| -------------- | ----------------------------- |
| Cloud Provider | Microsoft Azure               |
| Subscription   | Azure Free Labs               |
| Region         | eastus                        |
| Resource Group | kml_rg_main-f56fa9d90d7842e1  |
| NSG Name       | nautilus-nsg                  |
| Auth Method    | Azure CLI (Service Principal) |

## Prerequisites

- Azure CLI installed

- Active Azure subscription

- Authenticated Azure CLI session

## Step 1: Verify Azure Authentication

- Confirm the active Azure account and subscription context.

- `az account show`

Expected Outcome

- Subscription state is Enabled

- Correct tenant and subscription are active

📸 SCREENSHOT: `Azure CLI authenticated account details`
<img width="1034" height="556" alt="image" src="https://github.com/user-attachments/assets/6665d16e-1a21-4a8a-a45c-f257e0430949" />

## Step 2: Identify Target Resource Group

- Retrieve the resource group used for NSG deployment.

`RG_NAME=$(az group list --query "[0].name" -o tsv)
echo "Your Resource Group is: $RG_NAME"`

📸 SCREENSHOT: `Resource group resolution output`
<img width="1033" height="629" alt="image" src="https://github.com/user-attachments/assets/15612c68-a301-48ff-bbdf-7a7899224078" />

## Step 3: Create the Network Security Group

- Create the NSG within the identified resource group.

- `az network nsg create \
  --resource-group $RG_NAME \
  --name nautilus-nsg`

Result

- NSG is created with Azure default inbound and outbound rules

- Provisioning state is Succeeded

📸 SCREENSHOTS: `NSG creation success output`
<img width="1034" height="870" alt="image" src="https://github.com/user-attachments/assets/1a4497d3-66a0-4072-b6c0-c4549f39f23f" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/8c18d50c-366a-4d35-87da-34ef00b3c2bc" />
<img width="1034" height="869" alt="image" src="https://github.com/user-attachments/assets/7c5520f2-0b7d-4d10-aace-8abf58acafc4" />
<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/1e462635-2045-4919-9727-2cc31023c396" />

## Step 4: Add Inbound Rule - Allow HTTP (Port 80)

- Allow inbound HTTP traffic from any source.

- `az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-HTTP \
  --priority 100 \
  --destination-port-ranges 80 \
  --access Allow \
  --protocol Tcp`

📸 SCREENSHOT: `Allow-HTTP inbound rule configuration`
<img width="1038" height="866" alt="image" src="https://github.com/user-attachments/assets/05009582-c13c-40de-bd1b-ec2d892ddbbc" />

## Step 5: Add Inbound Rule - Allow SSH (Port 22)

- Allow inbound SSH access for secure remote administration.

- `az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-SSH \
  --priority 110 \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp`

📸 SCREENSHOT: `Allow-SSH inbound rule configuration`
<img width="1033" height="861" alt="image" src="https://github.com/user-attachments/assets/8b72b5b8-61e0-46d8-9f9a-8c58b39f7724" />

## Step 6: Validate NSG Rules

- Verify that the inbound rules are correctly applied and prioritized.

- `az network nsg rule list \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --output table`

Expected Output

- `Allow-HTTP → Priority 100 → Port 80`

- `Allow-SSH → Priority 110 → Port 22`

📸 SCREENSHOT: `Final NSG inbound rules table`
<img width="1191" height="627" alt="image" src="https://github.com/user-attachments/assets/acafbb42-bef3-4e13-ba85-83f4f0fbf464" />
## Automation Script

- To ensure repeatability and consistency, the entire configuration was automated using a Bash script.

File: `az-nsg-nautilus.sh`

- `#!/bin/bash`
- `# Azure Network Security Group Automation Script`
- `# Project: Nautilus Infrastructure Migration`

- `RG_NAME="kml_rg_main-f56fa9d90d7842e1"`
- `NSG_NAME="nautilus-nsg"`

- `echo "Creating NSG..."`
- `az network nsg create -g $RG_NAME -n $NSG_NAME`

- `echo "Adding Inbound Rules..."`
- `az network nsg rule create -g $RG_NAME --nsg-name $NSG_NAME -n Allow-HTTP --priority 100 --destination-port-ranges 80 --access Allow --protocol Tcp`
- `az network nsg rule create -g $RG_NAME --nsg-name $NSG_NAME -n Allow-SSH --priority 110 --destination-port-ranges 22 --access Allow --protocol Tcp`

- `echo "Final Configuration:"`
- `az network nsg rule list -g $RG_NAME --nsg-name $NSG_NAME --output table`

📸 SCREENSHOTS: `Script execution and final rule verification`
<img width="1183" height="822" alt="image" src="https://github.com/user-attachments/assets/e62d9a05-72c5-4bce-a648-7945093ca168" />

`Granting Executable Permissions to Deployment Script`
<img width="1187" height="436" alt="image" src="https://github.com/user-attachments/assets/d2ad0bab-6fb5-4b7b-8fc3-c0feddb64280" />

`Verification`
<img width="1187" height="438" alt="image" src="https://github.com/user-attachments/assets/db576d75-3954-470a-bc18-1a66cd11ac68" />

## Security Considerations

- Explicit allow rules are defined before Azure’s default deny rule

- Rule priorities follow industry best practices

- NSG is designed to be attached to a subnet or NIC as needed

⚠️ `In production environments, SSH access should be restricted to trusted CIDR ranges instead of 0.0.0.0/0.`

## Outcome

- Network Security Group successfully created
  
- HTTP and SSH inbound access explicitly allowed

- Configuration validated and automated

- Ready for subnet or NIC association

## Skills Demonstrated

- Azure Networking Fundamentals

- Azure CLI Automation

- Infrastructure as Code mindset

- Secure network design
