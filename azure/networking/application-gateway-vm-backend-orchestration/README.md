# Azure VM + Application Gateway Deployment

> **Enterprise HTTP Load Balancing on Azure**
> Deploying a Ubuntu 22.04 VM running Nginx behind an Azure Application Gateway (Basic SKU) using Azure CLI, with full NSG configuration, VNet/subnet segmentation, and public IP routing.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Step-by-Step Deployment](#step-by-step-deployment)
  - [Step 1: Verify Azure Account and Configure Defaults](#step-1-verify-azure-account-and-configure-defaults)
  - [Step 2: Identify Resource Group](#step-2-identify-resource-group)
  - [Step 3: Create Network Security Group](#step-3-create-network-security-group)
  - [Step 4: Add Inbound HTTP Rule to NSG](#step-4-add-inbound-http-rule-to-nsg)
  - [Step 5: Verify NSG Rule List](#step-5-verify-nsg-rule-list)
  - [Step 6: Create Virtual Network](#step-6-create-virtual-network)
  - [Step 7: Create VM Subnet](#step-7-create-vm-subnet)
  - [Step 8: Create Application Gateway Subnet](#step-8-create-application-gateway-subnet)
  - [Step 9: Verify Subnets](#step-9-verify-subnets)
  - [Step 10: Generate SSH Key Pair](#step-10-generate-ssh-key-pair)
  - [Step 11: Create VM Cloud-Init Startup Script](#step-11-create-vm-cloud-init-startup-script)
  - [Step 12: Create the Virtual Machine - First Attempt Error](#step-12-create-the-virtual-machine---first-attempt-error)
  - [Step 13: Create the Virtual Machine - Corrected](#step-13-create-the-virtual-machine---corrected)
  - [Step 14: Retrieve VM Private IP](#step-14-retrieve-vm-private-ip)
  - [Step 15: Create Public IP - First Attempt Error](#step-15-create-public-ip---first-attempt-error)
  - [Step 16: Delete Stale Public IP and Recreate with Standard SKU](#step-16-delete-stale-public-ip-and-recreate-with-standard-sku)
  - [Step 17: Confirm Public IP Address](#step-17-confirm-public-ip-address)
  - [Step 18: Deploy Application Gateway - Attempt 1 Standard_v2 Policy Blocked](#step-18-deploy-application-gateway---attempt-1-standard_v2-policy-blocked)
  - [Step 19: Deploy Application Gateway - Attempt 2 Basic via CLI Blocked](#step-19-deploy-application-gateway---attempt-2-basic-via-cli-blocked)
  - [Step 20: Deploy Application Gateway - Attempt 3 Standard_Small Policy Blocked](#step-20-deploy-application-gateway---attempt-3-standard_small-policy-blocked)
  - [Step 21: Deploy Application Gateway - Resolution Basic SKU via az rest](#step-21-deploy-application-gateway---resolution-basic-sku-via-az-rest)
  - [Step 22: Monitor Provisioning State](#step-22-monitor-provisioning-state)
  - [Step 23: Validate Application Gateway Configuration](#step-23-validate-application-gateway-configuration)
  - [Step 24: Test Public HTTP Endpoint](#step-24-test-public-http-endpoint)
  - [Step 25: Verify Backend Health](#step-25-verify-backend-health)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Resource Summary](#resource-summary)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)

---

## Project Overview

**Problem Statement:**
The Nautilus Development Team requires a new Azure Virtual Machine running a web server, fronted by an Azure Application Gateway for high availability and improved traffic management.

**Solution:**
Deploy a lightweight Ubuntu 22.04 VM with Nginx auto-configured via cloud-init `custom-data`, placed in a private subnet with no public IP. An Azure Application Gateway (Basic SKU) is provisioned in a dedicated subnet and exposes the service publicly via a static Standard public IP. All traffic flows through the AGW, and NSG rules restrict inbound access appropriately.

**Scope of Work:**

| Component | Resource Name | Notes |
|---|---|---|
| Network Security Group | `xfusion-nsg` | Allows TCP 80 inbound |
| Virtual Network | `xfusion-vnet` | Address space `10.0.0.0/16` |
| VM Subnet | `xfusion-vm-subnet` | `10.0.1.0/24`, NSG attached |
| AGW Subnet | `xfusion-agw-subnet` | `10.0.2.0/24`, no NSG (AGW requirement) |
| Virtual Machine | `xfusion-vm` | Ubuntu 22.04, Standard_B1s, private IP only |
| Public IP | `xfusion-agw-ip` | Standard SKU, Static |
| Application Gateway | `xfusion-agw` | Basic SKU (policy-enforced), capacity 1 |
| Backend Pool | `xfusion-backendpool` | Target: `10.0.1.4` |
| HTTP Settings | `xfusion-http-settings` | Port 80, HTTP |
| Listener | `xfusion-listener` | Frontend port 80, HTTP |
| Routing Rule | `xfusion-routing-rule` | Basic rule, priority 100 |

---

## Architecture

```
Internet
    |
    v
[Public IP: xfusion-agw-ip]  20.25.49.151
    |
    v
[Application Gateway: xfusion-agw]  (Basic SKU)
    Subnet: xfusion-agw-subnet (10.0.2.0/24)
    Listener: xfusion-listener  (port 80, HTTP)
    Routing Rule: xfusion-routing-rule
    Backend Pool: xfusion-backendpool
    |
    v
[VM: xfusion-vm]  (private: 10.0.1.4)
    Subnet: xfusion-vm-subnet (10.0.1.0/24)
    NSG: xfusion-nsg  (Allow TCP 80 inbound)
    OS: Ubuntu 22.04
    Web Server: Nginx (auto-started via cloud-init)
```

---

## Prerequisites

* Azure CLI installed and authenticated (`az login` or service principal)
* Active Azure subscription with sufficient quota in `eastus`
* `bash` shell environment
* `curl` for endpoint validation
* `ssh-keygen` for key pair generation

---

## Environment Setup

Export these variables before executing any step:

```bash
export RG=$(az group list --query "[0].name" -o tsv)
export VM_PRIVATE_IP="10.0.1.4"
export SUBSCRIPTION="f0c3bcdd-5ce2-4fa0-8cf3-41559747512b"
```

Set the default region:

```bash
az configure --defaults location=eastus
```

---

## Step-by-Step Deployment

---

### Step 1: Verify Azure Account and Configure Defaults

Confirm the authenticated Azure session and subscription before any resource operations. Set `eastus` as the default location to avoid specifying `--location` on every command.

```bash
az account show
```

**Expected output (key fields):**

```json
{
  "name": "Azure Free Labs",
  "state": "Enabled",
  "user": { "type": "servicePrincipal" }
}
```

```bash
az configure --defaults location=eastus
```

> **Note:** On Azure Free Labs, authentication is performed via a pre-configured service principal. Verify `"state": "Enabled"` and `"isDefault": true` before proceeding.

**Screenshot: az account show output & az configure defaults**

<img width="1057" height="467" alt="image" src="https://github.com/user-attachments/assets/eda006d6-f1c7-4d99-9638-1fc75d7a5a54" />

> Confirms subscription identity, tenant ID, and enabled state.

> Terminal confirming `az configure --defaults location=eastus` executed with no error output.

---

### Step 2: Identify Resource Group

Dynamically capture the pre-existing resource group name into `$RG` for reuse across all subsequent commands.

```bash
RG=$(az group list --query "[0].name" -o tsv)
echo "Resource Group: $RG"
```

**Expected output:**

```
Resource Group: kml_rg_main-de7d6381a5594d46
```

> **Note:** Always query the resource group dynamically on Free Labs. The suffix is session-specific and changes between lab instances. Hardcoding the name will cause failures in a new session.

**Screenshot: Resource group query output**
> Terminal showing `echo "Resource Group: $RG"` resolved to `kml_rg_main-de7d6381a5594d46`.

<img width="1055" height="475" alt="image" src="https://github.com/user-attachments/assets/1428b368-8b77-4059-a322-6b488be7aac0" />

---

### Step 3: Create Network Security Group

Create the NSG `xfusion-nsg` that will be associated with the VM subnet. Azure automatically populates six default security rules upon creation.

```bash
az network nsg create \
  --name xfusion-nsg \
  --resource-group $RG
```

**Default rules auto-created:**

| Rule Name | Direction | Access | Priority |
|---|---|---|---|
| AllowVnetInBound | Inbound | Allow | 65000 |
| AllowAzureLoadBalancerInBound | Inbound | Allow | 65001 |
| DenyAllInBound | Inbound | Deny | 65500 |
| AllowVnetOutBound | Outbound | Allow | 65000 |
| AllowInternetOutBound | Outbound | Allow | 65001 |
| DenyAllOutBound | Outbound | Deny | 65500 |

> **Note:** `securityRules` (custom rules) will be an empty array at this stage. Default rules cannot be deleted but are overridden by custom rules with lower priority numbers.

**Screenshots: NSG creation JSON response**
>  `az network nsg create` response showing `"provisioningState": "Succeeded"`, `"name": "xfusion-nsg"`, and the populated `defaultSecurityRules` array with all six default entries.

<img width="1062" height="872" alt="image" src="https://github.com/user-attachments/assets/9c5a13dd-af35-43ba-ab0a-70a397e9b387" />
<img width="1062" height="883" alt="image" src="https://github.com/user-attachments/assets/52ca01ad-1f4f-4b2e-8323-f10b84fa375c" />
<img width="1063" height="883" alt="image" src="https://github.com/user-attachments/assets/177930c2-d3c2-4979-b896-b8c46f5df905" />

---

### Step 4: Add Inbound HTTP Rule to NSG

Add a custom inbound security rule allowing HTTP traffic (TCP port 80) from any source. Priority `100` ensures this rule is evaluated before the `DenyAllInBound` default at priority `65500`.

```bash
az network nsg rule create \
  --nsg-name xfusion-nsg \
  --resource-group $RG \
  --name Allow-HTTP \
  --protocol Tcp \
  --direction Inbound \
  --priority 100 \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-range 80 \
  --access Allow
```

**Expected key fields in response:**

```json
{
  "name": "Allow-HTTP",
  "priority": 100,
  "protocol": "Tcp",
  "direction": "Inbound",
  "access": "Allow",
  "destinationPortRange": "80",
  "provisioningState": "Succeeded"
}
```

**Screenshot: NSG rule create JSON response**
> `az network nsg rule create` output confirming `Allow-HTTP` with priority 100, protocol Tcp, direction Inbound, destinationPortRange 80, access Allow, and provisioningState Succeeded.

<img width="1066" height="735" alt="image" src="https://github.com/user-attachments/assets/aea89e13-bf2d-4b1c-8e46-db13fa583fbc" />

---

### Step 5: Verify NSG Rule List

List all custom security rules on the NSG in table format to confirm the `Allow-HTTP` rule was applied correctly.

```bash
az network nsg rule list \
  --nsg-name xfusion-nsg \
  --resource-group $RG \
  -o table
```

**Expected output:**

```
Name        Priority    Protocol    Direction    DestinationPortRanges    Access
----------  ----------  ----------  -----------  -----------------------  ------
Allow-HTTP  100         Tcp         Inbound      80                       Allow
```

**Screenshot: NSG rule list table output**
> Terminal showing `az network nsg rule list -o table` with `Allow-HTTP` as the single custom rule entry at priority 100 on port 80.

<img width="1329" height="596" alt="image" src="https://github.com/user-attachments/assets/52e04450-4d06-408b-890c-9d3f8883d2ec" />

---

### Step 6: Create Virtual Network

Create the VNet `xfusion-vnet` with a `/16` address space that will contain both the VM and Application Gateway subnets.

```bash
az network vnet create \
  --name xfusion-vnet \
  --resource-group $RG \
  --address-prefix 10.0.0.0/16
```

**Expected key fields in response:**

```json
{
  "name": "xfusion-vnet",
  "addressSpace": { "addressPrefixes": ["10.0.0.0/16"] },
  "provisioningState": "Succeeded",
  "subnets": []
}
```

**Screenshot: VNet creation JSON response**
> `az network vnet create` output showing `xfusion-vnet` with `addressPrefixes: ["10.0.0.0/16"]`, `provisioningState: Succeeded`, and an empty `subnets` array confirming no subnets exist yet.

<img width="1298" height="602" alt="image" src="https://github.com/user-attachments/assets/b9513bb6-ff81-44f9-b581-2f7716edec6c" />

---

### Step 7: Create VM Subnet

Create `xfusion-vm-subnet` within the VNet and attach `xfusion-nsg` to it. All VM traffic will be governed by the NSG rules defined in Steps 3 and 4.

```bash
az network vnet subnet create \
  --name xfusion-vm-subnet \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --address-prefix 10.0.1.0/24 \
  --network-security-group xfusion-nsg
```

**Expected key fields in response:**

```json
{
  "name": "xfusion-vm-subnet",
  "addressPrefix": "10.0.1.0/24",
  "networkSecurityGroup": { "resourceGroup": "kml_rg_main-de7d6381a5594d46" },
  "provisioningState": "Succeeded"
}
```

**Screenshot: VM subnet creation JSON response**
> Output confirming `xfusion-vm-subnet` with `addressPrefix: 10.0.1.0/24`, `networkSecurityGroup` reference populated (confirming NSG was attached), and `provisioningState: Succeeded`.

<img width="1298" height="860" alt="image" src="https://github.com/user-attachments/assets/8968e87f-23f6-4e20-b1d0-38998a4a6ce6" />

---

### Step 8: Create Application Gateway Subnet

Create the dedicated `xfusion-agw-subnet` for the Application Gateway. No NSG is attached to this subnet.

```bash
az network vnet subnet create \
  --name xfusion-agw-subnet \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --address-prefix 10.0.2.0/24
```

**Expected key fields in response:**

```json
{
  "name": "xfusion-agw-subnet",
  "addressPrefix": "10.0.2.0/24",
  "provisioningState": "Succeeded"
}
```

> **Important:** Azure Application Gateway subnets must NOT have an NSG attached unless the NSG explicitly allows AGW management traffic on ports `65200-65535`. Omitting the NSG on this subnet avoids health probe failures and provisioning errors.

**Screenshot: AGW subnet creation JSON response**
> Output confirming `xfusion-agw-subnet` with `addressPrefix: 10.0.2.0/24`, no `networkSecurityGroup` field present, and `provisioningState: Succeeded`.

<img width="1294" height="863" alt="image" src="https://github.com/user-attachments/assets/9f4e2790-2853-4f7a-aa68-3c8dd91336b5" />

---

### Step 9: Verify Subnets

Confirm both subnets exist with correct CIDR allocations and provisioning state before proceeding to VM and AGW creation.

```bash
az network vnet subnet list \
  --vnet-name xfusion-vnet \
  --resource-group $RG \
  -o table
```

**Expected output:**

```
AddressPrefix    Name                ProvisioningState    ResourceGroup
---------------  ------------------  -------------------  ----------------------------
10.0.1.0/24      xfusion-vm-subnet   Succeeded            kml_rg_main-de7d6381a5594d46
10.0.2.0/24      xfusion-agw-subnet  Succeeded            kml_rg_main-de7d6381a5594d46
```

**Screenshot: Subnet list table output**
> Terminal showing both `xfusion-vm-subnet` (`10.0.1.0/24`) and `xfusion-agw-subnet` (`10.0.2.0/24`) with `ProvisioningState: Succeeded` for both entries.

<img width="1294" height="863" alt="image" src="https://github.com/user-attachments/assets/9f4e2790-2853-4f7a-aa68-3c8dd91336b5" />

---

### Step 10: Generate SSH Key Pair

Generate an RSA 2048-bit SSH key pair for VM authentication. The public key will be injected at VM creation time via `--ssh-key-values`.

**Check for existing key:**

```bash
cat ~/.ssh/id_rsa.pub
```

Key file was absent on this lab environment:

```
cat: /root/.ssh/id_rsa.pub: No such file or directory
```

Generate a new key pair:

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub
```

**Expected output:**

```
Generating public/private rsa key pair.
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:ywll5VAceuZW3Ua6xh7gScIhG6us28Jz7/sRtyR8m9Y root@azure-client
```

> **Security Note:** The `-N ""` flag sets an empty passphrase, acceptable only for lab environments. In production, always protect private keys with strong passphrases and store them in Azure Key Vault.

**Screenshot: cat ~/.ssh/id_rsa.pub not found**
> Terminal showing `cat: /root/.ssh/id_rsa.pub: No such file or directory` confirming no pre-existing SSH key on this lab host.

<img width="1298" height="701" alt="image" src="https://github.com/user-attachments/assets/ca06e14d-fa51-4b95-828a-f4a20ee5fca0" />

**Screenshot: ssh-keygen output and public key**
> Full `ssh-keygen` terminal output including the randomart image, followed by `cat ~/.ssh/id_rsa.pub` displaying the generated RSA public key value beginning with `ssh-rsa AAAAB3...`.

<img width="1298" height="701" alt="image" src="https://github.com/user-attachments/assets/ca06e14d-fa51-4b95-828a-f4a20ee5fca0" />

---

### Step 11: Create VM Cloud-Init Startup Script

Write the Nginx installation script to `/tmp/userdata.sh`. This file is passed to the VM via `--custom-data` and executed by cloud-init on first boot.

```bash
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF
```

Verify the script was written correctly:

```bash
cat /tmp/userdata.sh
```

**Expected output:**

```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
```

> **Note:** `systemctl enable nginx` ensures the service restarts automatically after a VM reboot. Without this, a reboot would leave the backend unhealthy from the AGW health probe perspective.

**Screenshot: userdata.sh creation and verification**
> Terminal showing the heredoc write command followed by `cat /tmp/userdata.sh` confirming all five lines of the Nginx startup script are correctly written to disk.

<img width="1297" height="682" alt="image" src="https://github.com/user-attachments/assets/ab9a01b8-ac18-452a-b229-23accf7c58b2" />

---

### Step 12: Create the Virtual Machine - First Attempt Error

**Problem:** The first `az vm create` command used `--os-disk-sku`, which is not a valid parameter for this Azure CLI version.

```bash
az vm create \
  --name xfusion-vm \
  --resource-group $RG \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --authentication-type ssh \
  --os-disk-sku Standard_LRS \
  --vnet-name xfusion-vnet \
  --subnet xfusion-vm-subnet \
  --nsg xfusion-nsg \
  --public-ip-address "" \
  --custom-data /tmp/userdata.sh
```

**Error received:**

```
unrecognized arguments: --os-disk-sku Standard_LRS
```

**Root cause:** `--os-disk-sku` does not exist as a parameter in `az vm create`. The correct parameter for specifying disk storage type is `--storage-sku`.

**Resolution:** Replace `--os-disk-sku Standard_LRS` with `--storage-sku Standard_LRS` and re-run. See Step 13.

**Screenshot: VM create error - unrecognized --os-disk-sku argument**
> Terminal showing the `az vm create` command with `--os-disk-sku Standard_LRS` and the resulting `unrecognized arguments: --os-disk-sku Standard_LRS` error message from the Azure CLI.

<img width="1294" height="617" alt="image" src="https://github.com/user-attachments/assets/c2a9b852-1f76-4c85-b751-1d0b7cde4abc" />

---

### Step 13: Create the Virtual Machine - Corrected

Replace `--os-disk-sku` with `--storage-sku`. The VM is deployed with `--public-ip-address ""` to ensure no public IP is attached, placing it exclusively on the private subnet accessible only through the Application Gateway.

```bash
az vm create \
  --name xfusion-vm \
  --resource-group $RG \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --authentication-type ssh \
  --storage-sku Standard_LRS \
  --vnet-name xfusion-vnet \
  --subnet xfusion-vm-subnet \
  --nsg xfusion-nsg \
  --public-ip-address "" \
  --custom-data /tmp/userdata.sh
```

**Expected output:**

```json
{
  "fqdns": "",
  "location": "eastus",
  "macAddress": "00-0D-3A-1C-9E-3C",
  "powerState": "VM running",
  "privateIpAddress": "10.0.1.4",
  "publicIpAddress": "",
  "resourceGroup": "kml_rg_main-de7d6381a5594d46",
  "zones": ""
}
```

> **Key Design Decision:** `--public-ip-address ""` explicitly disables public IP allocation. All public traffic must enter through the Application Gateway only, enforcing centralized traffic control.

**Screenshot: VM create success JSON response**
> `az vm create` output confirming `"powerState": "VM running"`, `"privateIpAddress": "10.0.1.4"`, and `"publicIpAddress": ""` for `xfusion-vm`, confirming the VM is up with no public IP.

<img width="1297" height="866" alt="image" src="https://github.com/user-attachments/assets/478a28f1-db91-4878-b915-bf88a9d7e872" />

---

### Step 14: Retrieve VM Private IP

Capture the VM private IP dynamically and store it in `$VM_PRIVATE_IP` for use as the Application Gateway backend pool target address.

```bash
VM_PRIVATE_IP=$(az vm show \
  --name xfusion-vm \
  --resource-group $RG \
  --show-details \
  --query privateIps \
  -o tsv)
echo "VM Private IP: $VM_PRIVATE_IP"
```

**Expected output:**

```
VM Private IP: 10.0.1.4
```

> **Note:** Always capture private IPs dynamically using `--show-details --query privateIps` rather than assuming the assigned address. Azure DHCP assigns IPs based on prior subnet allocations, and assumptions will break in shared or re-used environments.

**Screenshot: VM private IP echo output**
> Terminal showing `echo "VM Private IP: $VM_PRIVATE_IP"` resolved to `10.0.1.4` confirming the dynamic IP query succeeded.

<img width="1292" height="722" alt="image" src="https://github.com/user-attachments/assets/801c967d-3b42-45df-a4fe-09643136ee56" />

---

### Step 15: Create Public IP - First Attempt Error

**Context:** An initial Standard SKU public IP was created during the first deployment sequence, assigned address `172.174.27.169`. After the first AGW deployment failed (Step 18), this IP was deleted to clear the dependency. A subsequent attempt to create a Basic SKU IP to match AGW Basic tier also failed due to subscription quota.

**Initial Standard IP creation (later deleted):**

```bash
az network public-ip create \
  --name xfusion-agw-ip \
  --resource-group $RG \
  --sku Standard \
  --allocation-method Static
```

Assigned: `172.174.27.169`

**Deleted after initial AGW attempt failed:**

```bash
az network public-ip delete \
  --name xfusion-agw-ip \
  --resource-group $RG
```

**Attempt with Basic SKU (failed):**

```bash
az network public-ip create \
  --name xfusion-agw-ip \
  --resource-group $RG \
  --sku Basic \
  --allocation-method Dynamic
```

**Error received:**

```
(IPv4BasicSkuPublicIpCountLimitReached) Cannot create more than 0 IPv4 Basic SKU
public IP addresses for this subscription in this region.
Code: IPv4BasicSkuPublicIpCountLimitReached
```

**Root cause:** The Azure Free Labs subscription enforces a hard quota of zero for Basic SKU public IPs in `eastus`. Basic public IPs are deprecated and unavailable on this subscription tier.

**Screenshot: Initial Standard public IP creation showing 172.174.27.169**
> `az network public-ip create` response with `"ipAddress": "172.174.27.169"` and `"sku": {"name": "Standard"}` from the first provisioning attempt.

<img width="1301" height="868" alt="image" src="https://github.com/user-attachments/assets/ecafc348-e6e3-4d51-948f-40997665628f" />

**Screenshot: Basic SKU public IP quota error**
> Terminal showing the `IPv4BasicSkuPublicIpCountLimitReached` error: `Cannot create more than 0 IPv4 Basic SKU public IP addresses for this subscription in this region`.

```
[ SCREENSHOT PLACEHOLDER: 15b_public_ip_basic_quota_error.png ]
```

---

### Step 16: Delete Stale Public IP and Recreate with Standard SKU

Delete the existing public IP and recreate it with Standard SKU Static configuration, which is supported by this subscription and required by the Standard AGW public IP dependency.

```bash
az network public-ip delete \
  --name xfusion-agw-ip \
  --resource-group $RG
```

Recreate with Standard SKU:

```bash
az network public-ip create \
  --name xfusion-agw-ip \
  --resource-group $RG \
  --sku Standard \
  --allocation-method Static
```

**Expected key fields in response:**

```json
{
  "name": "xfusion-agw-ip",
  "ipAddress": "20.25.49.151",
  "publicIPAllocationMethod": "Static",
  "sku": { "name": "Standard" },
  "provisioningState": "Succeeded"
}
```

**Screenshot 16: Recreated Standard public IP with address 20.25.49.151**
> `az network public-ip create` response showing `"ipAddress": "20.25.49.151"`, `"sku": {"name": "Standard"}`, `"publicIPAllocationMethod": "Static"`, and `"provisioningState": "Succeeded"`.

```
[ SCREENSHOT PLACEHOLDER: 16_public_ip_standard_recreated.png ]
```

---

### Step 17: Confirm Public IP Address

Extract and confirm the final assigned public IP that will serve as the AGW frontend entry point.

```bash
az network public-ip show \
  --name xfusion-agw-ip \
  --resource-group $RG \
  --query ipAddress \
  -o tsv
```

**Expected output:**

```
20.25.49.151
```

**Screenshot 17: Public IP address confirmation**
> Terminal showing `az network public-ip show --query ipAddress -o tsv` returning the single value `20.25.49.151`.

```
[ SCREENSHOT PLACEHOLDER: 17_public_ip_tsv_confirm.png ]
```

---

### Step 18: Deploy Application Gateway - Attempt 1 Standard_v2 Policy Blocked

**Problem:** The first AGW deployment used `Standard_v2` SKU, which was immediately rejected by an Azure Policy enforcing `Basic` SKU only on this subscription.

```bash
az network application-gateway create \
  --name xfusion-agw \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --subnet xfusion-agw-subnet \
  --public-ip-address xfusion-agw-ip \
  --sku Standard_v2 \
  --capacity 1 \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --routing-rule-type Basic \
  --servers $VM_PRIVATE_IP \
  --priority 100
```

**Error received:**

```json
{
  "error": {
    "code": "InvalidTemplateDeployment",
    "details": [{
      "code": "RequestDisallowedByPolicy",
      "message": "Resource 'xfusion-agw' was disallowed by policy.
      Reasons: 'Only the Basic SKU is allowed for Azure Application Gateway.
      Please update the SKU to comply.'"
    }]
  }
}
```

**Root cause:** A subscription-level Azure Policy (`azure_application_gateway-tpm`) restricts AGW deployments to `Basic` SKU only. `Standard_v2` is explicitly denied.

**Screenshot 18: AGW Standard_v2 RequestDisallowedByPolicy error**
> Terminal showing the full `RequestDisallowedByPolicy` JSON error response for the `Standard_v2` AGW deployment attempt, including the policy definition ID, enforcement reason, and `policyDefinitionEffect: deny`.

```
[ SCREENSHOT PLACEHOLDER: 18_agw_standard_v2_policy_error.png ]
```

---

### Step 19: Deploy Application Gateway - Attempt 2 Basic via CLI Blocked

**Problem:** Passing `--sku Basic` directly to `az network application-gateway create` fails because the installed CLI version does not enumerate `Basic` as a valid `--sku` value, even though it is required by the subscription policy.

```bash
az network application-gateway create \
  --name xfusion-agw \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --subnet xfusion-agw-subnet \
  --public-ip-address xfusion-agw-ip \
  --sku Basic \
  --capacity 1 \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --routing-rule-type Basic \
  --servers $VM_PRIVATE_IP \
  --priority 100
```

**Error received:**

```
az network application-gateway create: 'Basic' is not a valid value for '--sku'.
Allowed values: Standard_Small, Standard_Medium, WAF_Medium, WAF_Large, Standard_v2, WAF_v2.
```

**Root cause:** The CLI's local argument validator hardcodes an allowed-values list for `--sku`. `Basic` exists in the ARM API and is required by the policy, but has not been added to this CLI version's enumeration. This creates the core conflict of this deployment: Azure Policy requires `Basic`, CLI refuses to pass it.

**Screenshot 19: AGW --sku Basic CLI validation error**
> Terminal showing `'Basic' is not a valid value for '--sku'` with the full allowed values list confirming `Basic` is absent and the conflict with the subscription policy is established.

```
[ SCREENSHOT PLACEHOLDER: 19_agw_cli_basic_sku_invalid.png ]
```

---

### Step 20: Deploy Application Gateway - Attempt 3 Standard_Small Policy Blocked

**Problem:** `Standard_Small`, the first value in the CLI's allowed-values list, was also blocked by the same subscription policy. This eliminated all CLI-accessible SKUs as viable options.

```bash
az network application-gateway create \
  --name xfusion-agw \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --subnet xfusion-agw-subnet \
  --public-ip-address xfusion-agw-ip \
  --sku Standard_Small \
  --capacity 1 \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --routing-rule-type Basic \
  --servers $VM_PRIVATE_IP \
  --priority 100
```

**Error received:**

```json
{
  "error": {
    "code": "RequestDisallowedByPolicy",
    "message": "Resource 'xfusion-agw' was disallowed by policy.
    Reasons: 'Only the Basic SKU is allowed for Azure Application Gateway.'"
  }
}
```

**Root cause:** Same policy as Attempt 1. All non-Basic SKUs are rejected. The full SKU elimination trail is now complete: `Standard_v2` blocked by policy, `Basic` blocked by CLI, `Standard_Small` blocked by policy. The only path forward is to bypass the CLI entirely.

**Screenshot 20: AGW Standard_Small policy violation error**
> Terminal showing the `RequestDisallowedByPolicy` JSON error for the `Standard_Small` attempt, completing the three-attempt SKU elimination trail that establishes the requirement for the `az rest` workaround.

```
[ SCREENSHOT PLACEHOLDER: 20_agw_standard_small_policy_error.png ]
```

---

### Step 21: Deploy Application Gateway - Resolution Basic SKU via az rest

**Resolution:** Bypass the CLI's local SKU argument validation entirely by submitting the complete ARM resource definition directly via `az rest --method PUT`. The ARM REST API accepts `Basic` as a valid SKU name, satisfying the subscription policy.

```bash
RG="kml_rg_main-de7d6381a5594d46"
SUBSCRIPTION="f0c3bcdd-5ce2-4fa0-8cf3-41559747512b"
VM_PRIVATE_IP="10.0.1.4"

az rest \
  --method PUT \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION}/resourceGroups/${RG}/providers/Microsoft.Network/applicationGateways/xfusion-agw?api-version=2023-05-01" \
  --body '{
    "location": "eastus",
    "properties": {
      "sku": {
        "name": "Basic",
        "tier": "Basic",
        "capacity": 1
      },
      "gatewayIPConfigurations": [
        {
          "name": "appGatewayIpConfig",
          "properties": {
            "subnet": {
              "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-de7d6381a5594d46/providers/Microsoft.Network/virtualNetworks/xfusion-vnet/subnets/xfusion-agw-subnet"
            }
          }
        }
      ],
      "frontendIPConfigurations": [
        {
          "name": "appGatewayFrontendIP",
          "properties": {
            "publicIPAddress": {
              "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-de7d6381a5594d46/providers/Microsoft.Network/publicIPAddresses/xfusion-agw-ip"
            }
          }
        }
      ],
      "frontendPorts": [
        {
          "name": "xfusion-frontend-port",
          "properties": { "port": 80 }
        }
      ],
      "backendAddressPools": [
        {
          "name": "xfusion-backendpool",
          "properties": {
            "backendAddresses": [
              { "ipAddress": "10.0.1.4" }
            ]
          }
        }
      ],
      "backendHttpSettingsCollection": [
        {
          "name": "xfusion-http-settings",
          "properties": {
            "port": 80,
            "protocol": "Http",
            "cookieBasedAffinity": "Disabled",
            "requestTimeout": 30
          }
        }
      ],
      "httpListeners": [
        {
          "name": "xfusion-listener",
          "properties": {
            "frontendIPConfiguration": {
              "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-de7d6381a5594d46/providers/Microsoft.Network/applicationGateways/xfusion-agw/frontendIPConfigurations/appGatewayFrontendIP"
            },
            "frontendPort": {
              "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-de7d6381a5594d46/providers/Microsoft.Network/applicationGateways/xfusion-agw/frontendPorts/xfusion-frontend-port"
            },
            "protocol": "Http"
          }
        }
      ],
      "requestRoutingRules": [
        {
          "name": "xfusion-routing-rule",
          "properties": {
            "ruleType": "Basic",
            "priority": 100,
            "httpListener": {
              "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-de7d6381a5594d46/providers/Microsoft.Network/applicationGateways/xfusion-agw/httpListeners/xfusion-listener"
            },
            "backendAddressPool": {
              "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-de7d6381a5594d46/providers/Microsoft.Network/applicationGateways/xfusion-agw/backendAddressPools/xfusion-backendpool"
            },
            "backendHttpSettings": {
              "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-de7d6381a5594d46/providers/Microsoft.Network/applicationGateways/xfusion-agw/backendHttpSettingsCollection/xfusion-http-settings"
            }
          }
        }
      ]
    }
  }'
```

**Expected response confirms AGW accepted and provisioning:**

```json
{
  "name": "xfusion-agw",
  "properties": {
    "sku": { "name": "Basic", "tier": "Basic", "capacity": 1 },
    "provisioningState": "Updating",
    "operationalState": "Stopped",
    "backendAddressPools": [{ "name": "xfusion-backendpool" }],
    "backendHttpSettingsCollection": [{ "name": "xfusion-http-settings" }],
    "httpListeners": [{ "name": "xfusion-listener" }],
    "requestRoutingRules": [{ "name": "xfusion-routing-rule" }]
  }
}
```

**Screenshot 21a: az rest PUT command submitted**
> Terminal showing the `az rest --method PUT` command being submitted with the ARM API URL (`api-version=2023-05-01`) and the opening of the full JSON request body.

```
[ SCREENSHOT PLACEHOLDER: 21a_az_rest_put_command.png ]
```

**Screenshot 21b: az rest successful AGW provisioning response**
> JSON response from `az rest` confirming `"sku": {"name": "Basic", "tier": "Basic"}`, `"provisioningState": "Updating"`, and all four component names correctly created: `xfusion-backendpool`, `xfusion-http-settings`, `xfusion-listener`, `xfusion-routing-rule`.

```
[ SCREENSHOT PLACEHOLDER: 21b_az_rest_agw_provisioning_response.png ]
```

---

### Step 22: Monitor Provisioning State

Application Gateway provisioning is asynchronous and typically takes 5 to 10 minutes. Poll the provisioning state every 30 seconds until `Succeeded` is returned.

```bash
watch -n 30 'az network application-gateway show \
  --name xfusion-agw \
  --resource-group kml_rg_main-de7d6381a5594d46 \
  --query provisioningState -o tsv'
```

**Target state:**

```
Succeeded
```

Press `Ctrl+C` once `Succeeded` is confirmed.

> **Note:** Do not proceed to Steps 23-25 while the state shows `Updating`. The AGW will not route traffic until `operationalState` transitions from `Stopped` to `Running`, which occurs when `provisioningState` reaches `Succeeded`.

**Screenshot 22: watch provisioningState output showing Succeeded**
> Terminal running `watch` with the repeated query showing `provisioningState` value `Succeeded`, confirming the AGW has fully provisioned and is ready to route traffic.

```
[ SCREENSHOT PLACEHOLDER: 22_agw_provisioning_succeeded.png ]
```

---

### Step 23: Validate Application Gateway Configuration

Query all AGW component names in a single structured table to confirm every resource was created with the correct naming and the correct SKU.

```bash
az network application-gateway show \
  --name xfusion-agw \
  --resource-group $RG \
  --query "{SKU:sku.name, BackendPool:backendAddressPools[0].name, HTTPSettings:backendHttpSettingsCollection[0].name, Listener:httpListeners[0].name, RoutingRule:requestRoutingRules[0].name}" \
  -o table
```

**Expected output:**

```
SKU    BackendPool          HTTPSettings           Listener          RoutingRule
-----  -------------------  ---------------------  ----------------  --------------------
Basic  xfusion-backendpool  xfusion-http-settings  xfusion-listener  xfusion-routing-rule
```

**Screenshot 23: AGW configuration summary table**
> Terminal showing the `az network application-gateway show -o table` output confirming all five fields: `SKU=Basic`, `BackendPool=xfusion-backendpool`, `HTTPSettings=xfusion-http-settings`, `Listener=xfusion-listener`, `RoutingRule=xfusion-routing-rule`.

```
[ SCREENSHOT PLACEHOLDER: 23_agw_config_summary_table.png ]
```

---

### Step 24: Test Public HTTP Endpoint

Retrieve the AGW public IP and issue an HTTP request to confirm the full traffic path is operational from the public internet through the gateway to the Nginx VM.

```bash
AGW_IP=$(az network public-ip show \
  --name xfusion-agw-ip \
  --resource-group $RG \
  --query ipAddress -o tsv)
echo "AGW Public IP: $AGW_IP"

curl -s -o /dev/null -w "%{http_code}" http://$AGW_IP
```

**Expected output:**

```
AGW Public IP: 20.25.49.151
200
```

An HTTP `200` confirms the complete traffic path is operational: public internet to AGW public IP (`20.25.49.151`), through the listener (`xfusion-listener`) and routing rule (`xfusion-routing-rule`), to the Nginx instance on the backend VM at `10.0.1.4:80`.

**Screenshot: curl HTTP 200 response via AGW public IP**
> Terminal showing `AGW Public IP: 20.25.49.151` on the first line immediately followed by the `curl` command output returning `200`, confirming successful end-to-end HTTP traffic flow through the Application Gateway.

<img width="1297" height="547" alt="image" src="https://github.com/user-attachments/assets/cfb6feee-a52a-4e19-8667-81efb0768cc4" />


---

### Step 25: Verify Backend Health

Confirm that the Application Gateway health probe independently reports the backend VM as `Healthy`. This is the definitive validation that the AGW can reach and receive valid responses from the Nginx backend, separate from the user-facing `curl` test.

```bash
az network application-gateway show-backend-health \
  --name xfusion-agw \
  --resource-group $RG \
  --query "backendAddressPools[0].backendHttpSettingsCollection[0].servers[0].health" \
  -o tsv
```

**Expected output:**

```
Healthy
```

> **Note:** If the backend reports `Unknown` or `Unhealthy`, verify: (1) Nginx is running on the VM with `systemctl status nginx`, (2) the NSG on `xfusion-vm-subnet` allows TCP 80 inbound from the AGW subnet `10.0.2.0/24`, and (3) the AGW `operationalState` is `Running` and not `Stopped`.

**Screenshot: Backend health status Healthy**
> Terminal showing `az network application-gateway show-backend-health` with the `--query` path returning the single value `Healthy`, confirming the AGW health probe is passing against the Nginx VM backend at `10.0.1.4`.

<img width="1294" height="577" alt="image" src="https://github.com/user-attachments/assets/53155d51-066a-4249-89d1-d887c025c229" />



---

## Errors Encountered and Resolutions

| # | Step | Command | Error | Root Cause | Resolution |
|---|---|---|---|---|---|
| 1 | 12 | `az vm create --os-disk-sku` | `unrecognized arguments: --os-disk-sku Standard_LRS` | Parameter does not exist in `az vm create` | Replace with `--storage-sku Standard_LRS` |
| 2 | 15 | `az network public-ip create --sku Basic` | `IPv4BasicSkuPublicIpCountLimitReached` | Subscription quota for Basic public IPs is zero in eastus | Use `--sku Standard --allocation-method Static` |
| 3 | 18 | `az network application-gateway create --sku Standard_v2` | `RequestDisallowedByPolicy: Only Basic SKU allowed` | Subscription Azure Policy blocks all non-Basic AGW SKUs | Must use `Basic` SKU; use `az rest` workaround |
| 4 | 19 | `az network application-gateway create --sku Basic` | `'Basic' is not a valid value for '--sku'` | CLI version does not enumerate Basic as an accepted value | Use `az rest --method PUT` with ARM API directly at `api-version=2023-05-01` |
| 5 | 20 | `az network application-gateway create --sku Standard_Small` | `RequestDisallowedByPolicy: Only Basic SKU allowed` | Same policy as error 3; eliminates all CLI-accessible SKUs | Use `az rest --method PUT` with `"name": "Basic", "tier": "Basic"` in SKU body |

---

## Resource Summary

| Resource | Name | Value |
|---|---|---|
| Resource Group | `kml_rg_main-de7d6381a5594d46` | Pre-existing |
| Location | `eastus` | Default configured |
| NSG | `xfusion-nsg` | Custom rule: Allow-HTTP TCP 80 inbound priority 100 |
| VNet | `xfusion-vnet` | `10.0.0.0/16` |
| VM Subnet | `xfusion-vm-subnet` | `10.0.1.0/24`, NSG: xfusion-nsg |
| AGW Subnet | `xfusion-agw-subnet` | `10.0.2.0/24`, no NSG |
| VM | `xfusion-vm` | Ubuntu 22.04, Standard_B1s, private IP: 10.0.1.4 |
| VM Admin User | `azureuser` | SSH key authentication |
| Web Server | Nginx | Auto-installed via cloud-init on first boot |
| Public IP | `xfusion-agw-ip` | Standard Static: 20.25.49.151 |
| Application Gateway | `xfusion-agw` | Basic SKU, capacity 1 |
| Backend Pool | `xfusion-backendpool` | IP: 10.0.1.4 |
| HTTP Settings | `xfusion-http-settings` | Port 80, HTTP, timeout 30s |
| Listener | `xfusion-listener` | Port 80, HTTP |
| Routing Rule | `xfusion-routing-rule` | Basic, priority 100 |

---

## Best Practices

### Networking

* Always use separate subnets for the Application Gateway and backend VMs. This isolates AGW management traffic and prevents NSG conflicts.
* Do NOT attach an NSG to the AGW subnet unless you explicitly allow AGW management traffic on ports `65200-65535`. Omitting the NSG is the safest default.
* Assign no public IP to backend VMs. All public access must route through the gateway to enforce centralized traffic control and single-point access management.
* Use `/24` or smaller subnets for the AGW subnet. Azure Application Gateway consumes IP addresses per instance plus standard subnet reservations.

### Security

* Apply NSG rules with least-privilege: allow only the specific ports required, not wildcard port ranges in production.
* Rotate SSH keys periodically and store private keys in Azure Key Vault rather than on the client filesystem.
* In production, front the Application Gateway with Azure Web Application Firewall (WAF) to protect against OWASP Top 10 threats.
* Enable HTTPS on the AGW listener and use TLS termination with certificates managed through Azure Key Vault.

### Infrastructure as Code

* Parameterize all resource names, IPs, and subscription IDs. Never hardcode values across commands, especially in lab environments where resource group names change per session.
* Export environment variables at the start of every session and validate them with `echo` before executing resource-creating commands.
* Use `az rest` deliberately as an escape hatch when CLI validation conflicts with ARM API capabilities. Always document the `api-version` used.

### VM Configuration

* Verify `--custom-data` scripts are idempotent. Cloud-init runs the script once at first boot; reimaging the VM will re-execute it.
* Always pair `systemctl start` with `systemctl enable` to ensure services survive reboots and remain available to AGW health probes.
* Prefer `cloud-config` YAML over raw shell scripts for complex initialization. Cloud-config produces structured logs in `/var/log/cloud-init-output.log` for easier debugging.

### Observability

* Poll `provisioningState` with `watch` rather than querying once and assuming completion. Application Gateway provisioning is asynchronous and takes 5 to 15 minutes.
* Run `az network application-gateway show-backend-health` immediately after provisioning as a mandatory post-deployment gate before marking the deployment complete.
* Configure Azure Monitor alerts on AGW backend health and unhealthy host count metrics for all production workloads.

---

## Lessons Learned

**1. Azure Policy enforcement and CLI SKU enumeration can be out of sync.**
The subscription policy enforced `Basic` SKU only, but the installed CLI version did not list `Basic` as an accepted `--sku` value. The resolution was to bypass the CLI entirely using `az rest` with the raw ARM API body. Always audit active Azure Policies before designing deployment commands for any subscription.

**2. Azure CLI parameter naming is version-sensitive.**
`--os-disk-sku` does not exist; the correct parameter is `--storage-sku`. Always validate parameter names against `az vm create --help` for the installed CLI version before executing. Do not copy commands from external tutorials without verifying them against current CLI help output.

**3. Basic SKU public IPs are quota-blocked and deprecated.**
The Azure Free Labs subscription had a hard quota of zero for Basic SKU public IPs in `eastus`. Standard SKU public IPs with Static allocation are the correct and future-proof choice for all new deployments. Azure has announced the deprecation of Basic SKU public IPs, and they should not be used in any new infrastructure.

**4. Application Gateway subnets must be dedicated and NSG-free by default.**
Attaching an NSG to the AGW subnet without explicitly allowing management traffic on ports `65200-65535` causes health probe failures and blocks traffic from reaching backends. The safest practice is to leave the AGW subnet without an NSG and apply all access control on the VM subnet instead.

**5. az rest is the correct and authoritative escape hatch when CLI and ARM Policy conflict.**
When the CLI argument validator rejects a value that Azure Policy requires, directly calling the ARM REST API via `az rest --method PUT` with a full resource body resolves the conflict cleanly. This requires constructing all nested resource ID paths manually and specifying the correct ARM `api-version` from the official reference.

**6. Dynamic IP assignment in small subnets is predictable but must never be assumed.**
The VM received `10.0.1.4` in the `10.0.1.0/24` subnet following Azure's standard address reservation pattern (`.1` gateway, `.2` DNS, `.3` broadcast). Always capture private IPs dynamically with `az vm show --show-details --query privateIps` rather than hardcoding assumed addresses into backend pool configurations.

**7. Both curl HTTP 200 and backend health Healthy are required validation gates.**
A `curl` returning `200` confirms user-facing traffic is flowing. `show-backend-health` returning `Healthy` confirms the AGW health probe is independently passing. Both checks together validate the complete operational path. Neither check alone is sufficient to sign off a deployment.

---




<img width="1330" height="626" alt="image" src="https://github.com/user-attachments/assets/c0d4856d-a6d3-4c98-bfd0-f588287f98ed" />








<img width="1303" height="860" alt="image" src="https://github.com/user-attachments/assets/23e2b348-15b5-43fe-8567-b5fb1a829f55" />

<img width="1297" height="823" alt="image" src="https://github.com/user-attachments/assets/bc1b417b-f53c-4f16-8c75-e2f97c985526" />
<img width="1299" height="866" alt="image" src="https://github.com/user-attachments/assets/82f14e2f-790c-4244-a8a0-d0bf12d71c35" />
<img width="1299" height="431" alt="image" src="https://github.com/user-attachments/assets/10e6d9ff-0452-40c1-8e47-cd7673991a78" />
<img width="1292" height="679" alt="image" src="https://github.com/user-attachments/assets/caa78f80-e534-48d4-ba67-84153fb0817f" />
<img width="1302" height="715" alt="image" src="https://github.com/user-attachments/assets/4f55a175-9192-4853-b19a-63826a585433" />
<img width="1299" height="866" alt="image" src="https://github.com/user-attachments/assets/781a4fef-f6fa-4152-9fc4-04fb93efd02a" />
<img width="1295" height="865" alt="image" src="https://github.com/user-attachments/assets/9f655645-26b7-4609-983a-ee4f8dc60d0a" />
<img width="1297" height="851" alt="image" src="https://github.com/user-attachments/assets/f32a6541-eb7d-46f0-9de5-d3e0d4028c9b" />
<img width="1297" height="857" alt="image" src="https://github.com/user-attachments/assets/aa38dbb0-2564-45f4-bbd2-ed6d98c7e1aa" />
<img width="1298" height="868" alt="image" src="https://github.com/user-attachments/assets/0483809a-e361-4496-b26f-103bc1528180" />
<img width="1296" height="655" alt="image" src="https://github.com/user-attachments/assets/1e6cc023-c60a-41bf-b1a7-b875a5013af8" />
<img width="1303" height="362" alt="image" src="https://github.com/user-attachments/assets/24b3eb83-96eb-4e0e-8cfa-ddc3540fa0fd" />
