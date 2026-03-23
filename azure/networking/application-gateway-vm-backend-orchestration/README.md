# Azure VM + Application Gateway Deployment

> **Enterprise HTTP Load Balancing on Azure Free Labs**
> Deploying a Ubuntu 22.04 VM running Nginx behind an Azure Application Gateway (Basic SKU) using Azure CLI, with full NSG configuration, VNet/subnet segmentation, and public IP routing.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Step-by-Step Deployment](#step-by-step-deployment)
  - [Step 1: Configure Defaults and Identify Resource Group](#step-1-configure-defaults-and-identify-resource-group)
  - [Step 2: Create Network Security Group (NSG)](#step-2-create-network-security-group-nsg)
  - [Step 3: Add Inbound HTTP Rule to NSG](#step-3-add-inbound-http-rule-to-nsg)
  - [Step 4: Create Virtual Network and Subnets](#step-4-create-virtual-network-and-subnets)
  - [Step 5: Generate SSH Key Pair](#step-5-generate-ssh-key-pair)
  - [Step 6: Create VM Startup Script (Nginx)](#step-6-create-vm-startup-script-nginx)
  - [Step 7: Create the Virtual Machine](#step-7-create-the-virtual-machine)
  - [Step 8: Retrieve VM Private IP](#step-8-retrieve-vm-private-ip)
  - [Step 9: Create Public IP for Application Gateway](#step-9-create-public-ip-for-application-gateway)
  - [Step 10: Deploy Application Gateway (Basic SKU via REST API)](#step-10-deploy-application-gateway-basic-sku-via-rest-api)
  - [Step 11: Monitor Provisioning State](#step-11-monitor-provisioning-state)
  - [Step 12: Validate Application Gateway Configuration](#step-12-validate-application-gateway-configuration)
  - [Step 13: Test Public HTTP Endpoint](#step-13-test-public-http-endpoint)
  - [Step 14: Verify Backend Health](#step-14-verify-backend-health)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Resource Summary](#resource-summary)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Screenshots](#screenshots)

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
    Subnet: xfusion-vm-subnet (10.0.2.0/24)
    NSG: xfusion-nsg (Allow TCP 80 inbound)
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

**Verify Azure CLI authentication:**

```bash
az account show
```

**Expected output fields to confirm:**

```
"name": "Azure Free Labs"
"state": "Enabled"
```

---

## Environment Setup

All commands below assume the following environment variables are set. Export these before executing any step:

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

### Step 1: Configure Defaults and Identify Resource Group

Set the default Azure location and capture the pre-existing resource group name into a shell variable for reuse throughout all commands.

```bash
az configure --defaults location=eastus

RG=$(az group list --query "[0].name" -o tsv)
echo "Resource Group: $RG"
```

**Expected output:**

```
Resource Group: kml_rg_main-de7d6381a5594d46
```

> **Note:** On Azure Free Labs, the resource group is pre-provisioned. Always query it dynamically rather than hardcoding to avoid stale references across lab sessions.

---

### Step 2: Create Network Security Group (NSG)

Create the NSG that will be associated with the VM subnet. Azure automatically attaches a set of default security rules covering inbound/outbound VNET and internet traffic.

```bash
az network nsg create \
  --name xfusion-nsg \
  --resource-group $RG
```

**Verify default rules were created:**

The response will contain `defaultSecurityRules` including:

| Rule Name | Direction | Access | Priority |
|---|---|---|---|
| AllowVnetInBound | Inbound | Allow | 65000 |
| AllowAzureLoadBalancerInBound | Inbound | Allow | 65001 |
| DenyAllInBound | Inbound | Deny | 65500 |
| AllowVnetOutBound | Outbound | Allow | 65000 |
| AllowInternetOutBound | Outbound | Allow | 65001 |
| DenyAllOutBound | Outbound | Deny | 65500 |

> **Note:** `securityRules` (custom rules) will be empty at this stage. Default rules cannot be deleted but can be overridden with lower priority numbers.

---

### Step 3: Add Inbound HTTP Rule to NSG

Add a custom inbound rule to allow HTTP (TCP port 80) traffic from any source. Priority `100` ensures this rule is evaluated before the `DenyAllInBound` default at `65500`.

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

**Verify the rule was added:**

```bash
az network nsg rule list --nsg-name xfusion-nsg --resource-group $RG -o table
```

**Expected output:**

```
Name        Priority    Protocol    Direction    DestinationPortRanges    Access
----------  ----------  ----------  -----------  -----------------------  ------
Allow-HTTP  100         Tcp         Inbound      80                       Allow
```

---

### Step 4: Create Virtual Network and Subnets

Create the virtual network with a `/16` address space, then provision two dedicated subnets: one for the VM and one for the Application Gateway.

**Create the VNet:**

```bash
az network vnet create \
  --name xfusion-vnet \
  --resource-group $RG \
  --address-prefix 10.0.0.0/16
```

**Create the VM subnet (with NSG attached):**

```bash
az network vnet subnet create \
  --name xfusion-vm-subnet \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --address-prefix 10.0.1.0/24 \
  --network-security-group xfusion-nsg
```

**Create the Application Gateway subnet (no NSG):**

```bash
az network vnet subnet create \
  --name xfusion-agw-subnet \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --address-prefix 10.0.2.0/24
```

> **Important:** Azure Application Gateway subnets must NOT have an NSG attached unless the NSG explicitly allows AGW management traffic (ports 65200-65535 for V2, or 65503-65534 for V1/Basic). For simplicity and to avoid provisioning issues, the AGW subnet is left NSG-free in this deployment.

**Verify both subnets:**

```bash
az network vnet subnet list \
  --vnet-name xfusion-vnet \
  --resource-group $RG \
  -o table
```

**Expected output:**

```
AddressPrefix    Name                ProvisioningState
---------------  ------------------  -------------------
10.0.1.0/24      xfusion-vm-subnet   Succeeded
10.0.2.0/24      xfusion-agw-subnet  Succeeded
```

---

### Step 5: Generate SSH Key Pair

Generate an RSA 2048-bit SSH key pair for authenticating to the VM. The public key will be injected during VM creation.

**Check for existing key:**

```bash
cat ~/.ssh/id_rsa.pub
```

If the file does not exist (expected on a fresh lab environment), generate a new key pair:

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub
```

**Expected output (truncated):**

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC... root@azure-client
```

> **Security Note:** The `-N ""` flag sets an empty passphrase, which is acceptable for lab environments. In production, always protect private keys with strong passphrases and store them in a secrets manager (e.g., Azure Key Vault).

---

### Step 6: Create VM Startup Script (Nginx)

Write a cloud-init compatible shell script that installs and enables Nginx on first boot. This is passed to the VM via `--custom-data`.

```bash
cat > /tmp/userdata.sh << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF
```

**Verify the script was written:**

```bash
cat /tmp/userdata.sh
```

> **Note:** Azure `--custom-data` accepts a raw shell script or a cloud-config YAML file. Azure passes the content to `cloud-init` on the VM. The script runs once at first boot as root.

---

### Step 7: Create the Virtual Machine

Deploy the VM into the VM subnet with no public IP. The private IP will be assigned dynamically from the `10.0.1.0/24` range. The Nginx installation script is passed as `--custom-data`.

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
  "powerState": "VM running",
  "privateIpAddress": "10.0.1.4",
  "publicIpAddress": "",
  "resourceGroup": "kml_rg_main-de7d6381a5594d46"
}
```

> **Key Design Decision:** `--public-ip-address ""` explicitly disables public IP allocation. All traffic must flow through the Application Gateway, enforcing the hub-and-spoke access pattern.

**Error encountered (and resolved):**

```
unrecognized arguments: --os-disk-sku Standard_LRS
```

**Root cause:** `--os-disk-sku` is not a valid parameter for `az vm create`.

**Fix:** Replace `--os-disk-sku` with `--storage-sku`:

```bash
# WRONG
--os-disk-sku Standard_LRS

# CORRECT
--storage-sku Standard_LRS
```

---

### Step 8: Retrieve VM Private IP

Capture the VM private IP for use as the Application Gateway backend target.

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

---

### Step 9: Create Public IP for Application Gateway

Provision a Standard SKU Static public IP for the Application Gateway frontend.

```bash
az network public-ip create \
  --name xfusion-agw-ip \
  --resource-group $RG \
  --sku Standard \
  --allocation-method Static
```

**Confirm the assigned IP:**

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

**Errors encountered (and resolved):**

**Attempt 1: Wrong SKU name**

```bash
--sku Basic --allocation-method Dynamic
```

**Error:**

```
(IPv4BasicSkuPublicIpCountLimitReached) Cannot create more than 0 IPv4 Basic SKU
public IP addresses for this subscription in this region.
```

**Root cause:** Azure Free Labs blocks Basic SKU public IPs in the `eastus` region at the subscription level.

**Fix:** Use `Standard` SKU with `Static` allocation:

```bash
--sku Standard --allocation-method Static
```

> **Note:** A Standard SKU public IP was also created and deleted prior to the above error. The deletion was necessary to recreate with correct settings after the initial AGW provisioning attempt failed.

---

### Step 10: Deploy Application Gateway (Basic SKU via REST API)

This step required multiple attempts due to Azure Policy enforcement and Azure CLI SKU validation conflicts. Full error trail and resolution are documented below.

**Attempt 1: Standard_v2 SKU (Policy Blocked)**

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

**Error:**

```
RequestDisallowedByPolicy: Resource 'xfusion-agw' was disallowed by policy.
Reasons: 'Only the Basic SKU is allowed for Azure Application Gateway.'
```

**Attempt 2: Basic SKU via CLI (CLI Validation Blocked)**

```bash
az network application-gateway create \
  --sku Basic \
  ...
```

**Error:**

```
'Basic' is not a valid value for '--sku'.
Allowed values: Standard_Small, Standard_Medium, WAF_Medium, WAF_Large, Standard_v2, WAF_v2.
```

**Root cause:** The Azure CLI version installed in the lab does not enumerate `Basic` as a valid `--sku` value, even though the Azure REST API and policy enforcement require it.

**Attempt 3: Standard_Small SKU (Policy Blocked)**

```bash
az network application-gateway create \
  --sku Standard_Small \
  ...
```

**Error:**

```
RequestDisallowedByPolicy: Only the Basic SKU is allowed for Azure Application Gateway.
```

**Resolution: Use az rest to call the ARM API directly**

Bypass the CLI's local SKU enumeration by submitting the full ARM resource definition via `az rest`:

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

**Expected response confirms:**

```json
{
  "name": "xfusion-agw",
  "properties": {
    "sku": { "name": "Basic", "tier": "Basic", "capacity": 1 },
    "provisioningState": "Updating",
    "operationalState": "Stopped"
  }
}
```

---

### Step 11: Monitor Provisioning State

Application Gateway provisioning typically takes 5 to 10 minutes. Poll the provisioning state until `Succeeded` is returned.

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

---

### Step 12: Validate Application Gateway Configuration

Confirm all AGW components were created correctly with a single structured query:

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

---

### Step 13: Test Public HTTP Endpoint

Retrieve the AGW public IP and send an HTTP request to confirm the web server is reachable through the gateway:

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

An HTTP `200` response confirms traffic successfully traversed the Application Gateway and reached the Nginx instance on the backend VM.

---

### Step 14: Verify Backend Health

Confirm that the Application Gateway health probe reports the backend VM as healthy:

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

> **Note:** If the backend reports `Unknown` or `Unhealthy`, check that Nginx is running on the VM (`systemctl status nginx`) and that the NSG on `xfusion-vm-subnet` allows TCP 80 from the AGW subnet `10.0.2.0/24`.

---

## Errors Encountered and Resolutions

| # | Command | Error | Root Cause | Resolution |
|---|---|---|---|---|
| 1 | `az vm create` | `unrecognized arguments: --os-disk-sku` | Parameter does not exist in this CLI version | Replace with `--storage-sku Standard_LRS` |
| 2 | `az network application-gateway create --sku Standard_v2` | `RequestDisallowedByPolicy: Only the Basic SKU is allowed` | Subscription-level Azure Policy enforces Basic SKU only | Use `--sku Basic` or bypass via `az rest` |
| 3 | `az network application-gateway create --sku Basic` | `'Basic' is not a valid value for '--sku'` | CLI version does not expose Basic as an enumerated value despite ARM supporting it | Use `az rest` with ARM API directly at `api-version=2023-05-01` |
| 4 | `az network application-gateway create --sku Standard_Small` | `RequestDisallowedByPolicy: Only the Basic SKU is allowed` | Same policy as error 2 | Use `az rest` with `"name": "Basic", "tier": "Basic"` in the SKU body |
| 5 | `az network public-ip create --sku Basic` | `IPv4BasicSkuPublicIpCountLimitReached: Cannot create more than 0 Basic SKU IPs` | Subscription quota for Basic public IPs is zero in this region | Delete the existing Standard IP, recreate with `--sku Standard --allocation-method Static` |

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
| VM Admin User | `azureuser` | SSH key auth |
| Web Server | Nginx | Auto-installed via cloud-init |
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
* Do NOT attach an NSG to the AGW subnet unless you explicitly allow the AGW management port range (`65200-65535` for v2/Basic).
* Assign no public IP to backend VMs. All public access must route through the gateway to enforce centralized traffic inspection and NSG control.
* Use `/24` or smaller subnets for AGW. Azure Application Gateway consumes one IP per gateway instance plus one for the subnet gateway.

### Security

* Apply NSG rules with least-privilege principles: allow only the specific ports required (`TCP 80` in this case), not wildcard port ranges.
* Rotate SSH keys periodically and store private keys in Azure Key Vault, not on the client filesystem.
* In production, front the Application Gateway with Azure Web Application Firewall (WAF) to protect against OWASP Top 10 threats.
* Enable HTTPS on the Application Gateway listener and use TLS termination at the AGW layer with a certificate from Azure Key Vault.

### Infrastructure as Code

* Parameterize all resource names, IPs, and subscription IDs. Avoid hardcoding values across commands.
* Export environment variables at the start of every session (`RG`, `SUBSCRIPTION`, `VM_PRIVATE_IP`) and validate them with `echo` before running destructive or resource-creating commands.
* Use `az rest` as a last resort to bypass CLI validation gaps. Document the specific `api-version` used and test against the target ARM API version in your target environment.

### VM Configuration

* Always verify `--custom-data` scripts are idempotent. Cloud-init runs the script once at first boot; if the VM is reimaged, the script will run again.
* Use `systemctl enable nginx` alongside `systemctl start nginx` to ensure the service survives reboots.
* Use `cloud-config` YAML (`#cloud-config` header) for complex initialization over raw shell scripts for better error handling and logging visibility in `/var/log/cloud-init-output.log`.

### Observability

* Poll `provisioningState` with `watch` rather than querying once and assuming success. Application Gateway provisioning is asynchronous and can take up to 15 minutes.
* Use `az network application-gateway show-backend-health` immediately after provisioning to confirm end-to-end connectivity before marking the deployment complete.
* Set up Azure Monitor alerts on AGW backend health and unhealthy host count metrics for production workloads.

---

## Lessons Learned

**1. Azure Policy enforcement and CLI SKU enumeration can be out of sync.**
The subscription policy enforced `Basic` SKU only, but the installed Azure CLI version did not list `Basic` as an accepted `--sku` value. The resolution was to bypass the CLI entirely and use `az rest` with the raw ARM API body. Always check active Azure Policies before designing your deployment commands.

**2. Azure CLI parameter naming is version-sensitive.**
`--os-disk-sku` does not exist; the correct parameter is `--storage-sku`. Always consult `az vm create --help` or the current official CLI reference for the exact parameter set in your installed version. Do not copy commands from generic tutorials without validating against `--help` output.

**3. Basic SKU public IPs may be quota-blocked even in free tier subscriptions.**
The Azure Free Labs subscription had a quota of zero for Basic SKU public IPs in `eastus`. Standard SKU public IPs (Static) are the correct and future-proof choice. Azure has announced deprecation of Basic SKU public IPs; new deployments should always use Standard SKU.

**4. Application Gateway subnets must be dedicated.**
Attempting to attach an NSG to the AGW subnet without explicitly allowing the health probe and management port ranges will cause the gateway to fail health checks and prevent traffic from reaching backends. Dedicate the subnet and leave NSG management to the VM subnet.

**5. az rest is a powerful escape hatch for policy-compliant deployments.**
When the CLI wrapper and Azure Policy are misaligned (policy requires a SKU the CLI will not accept), directly calling the ARM REST API via `az rest --method PUT` with the full resource body resolves the conflict. This requires knowing the correct `api-version`, which can be found in the ARM API reference for the specific resource type.

**6. Dynamic IP assignment from DHCP is predictable in small subnets.**
The VM received `10.0.1.4` (the first assignable host after `.1` gateway, `.2` DNS, `.3` broadcast reservation) in the `10.0.1.0/24` subnet. For backend pool targeting by IP, use `az vm show --show-details --query privateIps` to capture the IP dynamically rather than assuming it.

**7. Backend health validation is a critical post-deployment gate.**
A `200` HTTP response from `curl` proves traffic is flowing, but `show-backend-health` showing `Healthy` proves the AGW health probe is passing. Both checks together confirm the full end-to-end path is operational. Never skip backend health validation.

---

## Screenshots

### Screenshot 1: Azure Account Verification
> `az account show` output confirming subscription name, state, and authenticated service principal.

```
[ SCREENSHOT PLACEHOLDER: az_account_show_output.png ]
```

### Screenshot 2: NSG Creation with Default Rules
> `az network nsg create` response showing the six auto-generated default security rules for `xfusion-nsg`.

```
[ SCREENSHOT PLACEHOLDER: nsg_creation_default_rules.png ]
```

### Screenshot 3: Custom HTTP Inbound Rule
> `az network nsg rule list -o table` confirming `Allow-HTTP` rule at priority 100 on TCP port 80.

```
[ SCREENSHOT PLACEHOLDER: nsg_rule_allow_http_table.png ]
```

### Screenshot 4: VNet and Subnet Creation
> `az network vnet subnet list -o table` showing both `xfusion-vm-subnet` and `xfusion-agw-subnet` with correct CIDR ranges and provisioning state `Succeeded`.

```
[ SCREENSHOT PLACEHOLDER: vnet_subnet_list_table.png ]
```

### Screenshot 5: SSH Key Generation
> Terminal output of `ssh-keygen` showing key fingerprint and randomart, followed by `cat ~/.ssh/id_rsa.pub` with the full public key.

```
[ SCREENSHOT PLACEHOLDER: ssh_keygen_output.png ]
```

### Screenshot 6: VM Creation Success
> `az vm create` JSON response confirming `"powerState": "VM running"`, `"privateIpAddress": "10.0.1.4"`, and `"publicIpAddress": ""`.

```
[ SCREENSHOT PLACEHOLDER: vm_create_success.png ]
```

### Screenshot 7: VM Create Error (--os-disk-sku)
> CLI error output: `unrecognized arguments: --os-disk-sku Standard_LRS` demonstrating the incorrect parameter before correction to `--storage-sku`.

```
[ SCREENSHOT PLACEHOLDER: vm_create_error_os_disk_sku.png ]
```

### Screenshot 8: Public IP Creation
> `az network public-ip create` response showing `"ipAddress": "20.25.49.151"` and `"sku": {"name": "Standard"}`.

```
[ SCREENSHOT PLACEHOLDER: public_ip_standard_created.png ]
```

### Screenshot 9: Basic SKU Public IP Quota Error
> CLI error: `IPv4BasicSkuPublicIpCountLimitReached: Cannot create more than 0 IPv4 Basic SKU public IP addresses` confirming the subscription-level restriction.

```
[ SCREENSHOT PLACEHOLDER: public_ip_basic_quota_error.png ]
```

### Screenshot 10: Application Gateway Policy Violation (Standard_v2)
> Full policy violation JSON error showing `RequestDisallowedByPolicy` and the policy definition requiring `Basic` SKU only.

```
[ SCREENSHOT PLACEHOLDER: agw_policy_violation_standard_v2.png ]
```

### Screenshot 11: Application Gateway CLI SKU Error (Basic)
> CLI validation error: `'Basic' is not a valid value for '--sku'. Allowed values: Standard_Small, Standard_Medium...` showing the CLI/policy mismatch.

```
[ SCREENSHOT PLACEHOLDER: agw_cli_sku_basic_invalid.png ]
```

### Screenshot 12: Application Gateway Policy Violation (Standard_Small)
> Policy enforcement JSON error confirming `Standard_Small` was also blocked, completing the SKU elimination trail before the `az rest` resolution.

```
[ SCREENSHOT PLACEHOLDER: agw_policy_violation_standard_small.png ]
```

### Screenshot 13: Application Gateway Deployed via az rest
> Successful `az rest --method PUT` response showing AGW provisioning state `Updating` with `"sku": {"name": "Basic", "tier": "Basic"}` confirming the workaround succeeded.

```
[ SCREENSHOT PLACEHOLDER: agw_rest_api_success_basic_sku.png ]
```

### Screenshot 14: Application Gateway Configuration Summary
> `az network application-gateway show -o table` output confirming SKU=Basic, backend pool, HTTP settings, listener, and routing rule names.

```
[ SCREENSHOT PLACEHOLDER: agw_config_summary_table.png ]
```

### Screenshot 15: HTTP 200 from AGW Public IP
> Terminal output showing `AGW Public IP: 20.25.49.151` followed by `curl` returning `200`, confirming end-to-end traffic flow through the gateway to the Nginx VM.

```
[ SCREENSHOT PLACEHOLDER: curl_agw_http_200.png ]
```

### Screenshot 16: Backend Health Status
> `az network application-gateway show-backend-health` output returning `Healthy`, confirming AGW health probe is passing against the VM backend.

```
[ SCREENSHOT PLACEHOLDER: agw_backend_health_healthy.png ]
```

---

## Author

**Deployment completed on:** Mon Mar 23 2026
**Region:** East US
**Subscription:** Azure Free Labs
**Lab window:** 03:34:45 UTC to 04:34:45 UTC

---

*This README was authored following FAANG-grade infrastructure documentation standards. All commands, error traces, and resolutions reflect the exact terminal session executed during deployment.*



<img width="1057" height="467" alt="image" src="https://github.com/user-attachments/assets/eda006d6-f1c7-4d99-9638-1fc75d7a5a54" />
<img width="1055" height="475" alt="image" src="https://github.com/user-attachments/assets/1428b368-8b77-4059-a322-6b488be7aac0" />
<img width="1062" height="872" alt="image" src="https://github.com/user-attachments/assets/9c5a13dd-af35-43ba-ab0a-70a397e9b387" />
<img width="1062" height="883" alt="image" src="https://github.com/user-attachments/assets/52ca01ad-1f4f-4b2e-8323-f10b84fa375c" />
<img width="1063" height="883" alt="image" src="https://github.com/user-attachments/assets/177930c2-d3c2-4979-b896-b8c46f5df905" />
<img width="1066" height="735" alt="image" src="https://github.com/user-attachments/assets/aea89e13-bf2d-4b1c-8e46-db13fa583fbc" />
<img width="1329" height="596" alt="image" src="https://github.com/user-attachments/assets/52e04450-4d06-408b-890c-9d3f8883d2ec" />
<img width="1330" height="626" alt="image" src="https://github.com/user-attachments/assets/c0d4856d-a6d3-4c98-bfd0-f588287f98ed" />
<img width="1298" height="602" alt="image" src="https://github.com/user-attachments/assets/b9513bb6-ff81-44f9-b581-2f7716edec6c" />
<img width="1298" height="860" alt="image" src="https://github.com/user-attachments/assets/8968e87f-23f6-4e20-b1d0-38998a4a6ce6" />
<img width="1294" height="863" alt="image" src="https://github.com/user-attachments/assets/9f4e2790-2853-4f7a-aa68-3c8dd91336b5" />
<img width="1298" height="701" alt="image" src="https://github.com/user-attachments/assets/ca06e14d-fa51-4b95-828a-f4a20ee5fca0" />
<img width="1297" height="682" alt="image" src="https://github.com/user-attachments/assets/ab9a01b8-ac18-452a-b229-23accf7c58b2" />
<img width="1294" height="617" alt="image" src="https://github.com/user-attachments/assets/c2a9b852-1f76-4c85-b751-1d0b7cde4abc" />
<img width="1297" height="866" alt="image" src="https://github.com/user-attachments/assets/478a28f1-db91-4878-b915-bf88a9d7e872" />
<img width="1292" height="722" alt="image" src="https://github.com/user-attachments/assets/801c967d-3b42-45df-a4fe-09643136ee56" />
<img width="1303" height="860" alt="image" src="https://github.com/user-attachments/assets/23e2b348-15b5-43fe-8567-b5fb1a829f55" />
<img width="1301" height="868" alt="image" src="https://github.com/user-attachments/assets/ecafc348-e6e3-4d51-948f-40997665628f" />
<img width="1297" height="823" alt="image" src="https://github.com/user-attachments/assets/bc1b417b-f53c-4f16-8c75-e2f97c985526" />
<img width="1299" height="866" alt="image" src="https://github.com/user-attachments/assets/82f14e2f-790c-4244-a8a0-d0bf12d71c35" />
<img width="1299" height="431" alt="image" src="https://github.com/user-attachments/assets/10e6d9ff-0452-40c1-8e47-cd7673991a78" />
<img width="1292" height="679" alt="image" src="https://github.com/user-attachments/assets/caa78f80-e534-48d4-ba67-84153fb0817f" />
<img width="1302" height="715" alt="image" src="https://github.com/user-attachments/assets/4f55a175-9192-4853-b19a-63826a585433" />
<img width="1299" height="866" alt="image" src="https://github.com/user-attachments/assets/781a4fef-f6fa-4152-9fc4-04fb93efd02a" />
<<img width="1295" height="865" alt="image" src="https://github.com/user-attachments/assets/9f655645-26b7-4609-983a-ee4f8dc60d0a" />
<img width="1297" height="851" alt="image" src="https://github.com/user-attachments/assets/f32a6541-eb7d-46f0-9de5-d3e0d4028c9b" />
<img width="1297" height="857" alt="image" src="https://github.com/user-attachments/assets/aa38dbb0-2564-45f4-bbd2-ed6d98c7e1aa" />
<img width="1298" height="868" alt="image" src="https://github.com/user-attachments/assets/0483809a-e361-4496-b26f-103bc1528180" />
<img width="1296" height="655" alt="image" src="https://github.com/user-attachments/assets/1e6cc023-c60a-41bf-b1a7-b875a5013af8" />
<img width="1303" height="362" alt="image" src="https://github.com/user-attachments/assets/24b3eb83-96eb-4e0e-8cfa-ddc3540fa0fd" />
<img width="1297" height="547" alt="image" src="https://github.com/user-attachments/assets/cfb6feee-a52a-4e19-8667-81efb0768cc4" />
<img width="1294" height="577" alt="image" src="https://github.com/user-attachments/assets/53155d51-066a-4249-89d1-d887c025c229" />

