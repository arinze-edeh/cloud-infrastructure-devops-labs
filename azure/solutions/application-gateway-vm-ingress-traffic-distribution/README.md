# Azure Application Gateway with Load-Balanced Backend VMs

> **Enterprise-style deployment of an Azure Application Gateway distributing HTTP traffic across two Nginx-backed Ubuntu virtual machines using Azure CLI.**

---

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Phase 1 - Resource Group Discovery](#phase-1---resource-group-discovery)
  * [Phase 2 - Virtual Network and Subnet Provisioning](#phase-2---virtual-network-and-subnet-provisioning)
  * [Phase 3 - Network Security Group Configuration](#phase-3---network-security-group-configuration)
  * [Phase 4 - Virtual Machine Deployment](#phase-4---virtual-machine-deployment)
  * [Phase 5 - Nginx Installation and Content Configuration](#phase-5---nginx-installation-and-content-configuration)
  * [Phase 6 - Public IP Provisioning for Application Gateway](#phase-6---public-ip-provisioning-for-application-gateway)
  * [Phase 7 - Application Gateway Deployment](#phase-7---application-gateway-deployment)
  * [Phase 8 - Validation and Traffic Distribution Testing](#phase-8---validation-and-traffic-distribution-testing)
* [Resource Summary](#resource-summary)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Troubleshooting Reference](#troubleshooting-reference)

---

## Overview

The Nautilus DevOps team required an Azure Application Gateway to serve as a Layer 7 load balancer, distributing incoming HTTP traffic across a backend pool of two Ubuntu virtual machines. Each VM runs Nginx and serves a distinct version response, enabling clear verification of round-robin traffic distribution.

**Objectives:**

* Provision a dedicated VNet with isolated subnets for VMs and the Application Gateway
* Deploy two Ubuntu 22.04 VMs with Nginx installed and version-specific content
* Create an Azure Application Gateway (Basic SKU) with a backend pool, HTTP settings, listener, and routing rule
* Validate load balancing behaviour by confirming alternating responses from both VMs

**Region:** East US
**Authentication Method:** Password-based (lab environment)
**CLI Tool:** Azure CLI (`az`)

---

## Architecture

```
Internet
    |
    v
[ Public IP: xfusion-apgw-ip ]  (20.127.87.36 - Static, Standard SKU)
    |
    v
[ Application Gateway: xfusion-apgw ]  (Basic SKU, Capacity: 1)
    |  Subnet: xfusion-apgw-subnet (10.0.2.0/24)
    |
    +---[ Listener: xfusion-listener ]  (HTTP :80)
    |
    +---[ Routing Rule: xfusion-routing-rule ]  (Basic, Priority: 100)
    |
    +---[ Backend HTTP Settings: xfusion-http-settings ]  (HTTP :80, Timeout: 30s)
    |
    +---[ Backend Pool: xfusion-backend-pool ]
             |                    |
             v                    v
    [ xfusion-vm1 ]       [ xfusion-vm2 ]
    10.0.1.4               10.0.1.5
    Nginx: Version 1       Nginx: Version 2
    Subnet: xfusion-subnet (10.0.1.0/24)
    NSG: xfusion-nsg (Allow TCP :80, :22 Inbound)
```

**VNet Address Space:** `10.0.0.0/16`

| Subnet | CIDR | Purpose |
|---|---|---|
| `xfusion-subnet` | `10.0.1.0/24` | Backend virtual machines |
| `xfusion-apgw-subnet` | `10.0.2.0/24` | Application Gateway (dedicated, required by Azure) |

---

## Prerequisites

* Azure CLI installed and authenticated (`az login`)
* Contributor or Owner role on the target subscription
* An existing resource group (or permissions to create one)
* Bash shell environment

---

## Implementation Guide

### Phase 1 - Resource Group Discovery

Identify the target resource group dynamically to avoid hardcoding environment-specific values. This approach makes the implementation portable across lab resets and environment refreshes.

```bash
az group list --output table
```

*Screenshot: az group list output showing kml_rg_main-e632ffb9f5df49e1 in East US*

<img width="1034" height="435" alt="image" src="https://github.com/user-attachments/assets/65dc6d89-c1e9-4147-a015-ee39031ee044" />

Store the resource group name in an environment variable for reuse throughout the session:

```bash
RG=$(az group list --query "[0].name" --output tsv)
echo "Resource Group: $RG"
```

**Expected Output:**
```
Resource Group: kml_rg_main-e632ffb9f5df49e1
```

---

### Phase 2 - Virtual Network and Subnet Provisioning

#### 2.1 Create the Virtual Network

Create `xfusion-vnet` with a `/16` address space in the East US region. The `/16` block provides ample room for multiple subnets while keeping all resources logically grouped.

```bash
az network vnet create \
  --resource-group $RG \
  --name xfusion-vnet \
  --location eastus \
  --address-prefix 10.0.0.0/16
```

**Key output field to verify:** `"provisioningState": "Succeeded"`

*Screenshot: VNet creation JSON output confirming provisioningState Succeeded*

<img width="1031" height="758" alt="image" src="https://github.com/user-attachments/assets/e8fbd8a7-3b5f-4335-bea1-94ed4603236d" />

#### 2.2 Create the VM Subnet

Provision `xfusion-subnet` within the VNet for the backend virtual machines. Using a dedicated `/24` subnet for VMs ensures clean separation from gateway traffic.

```bash
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --name xfusion-subnet \
  --address-prefix 10.0.1.0/24
```

#### 2.3 Create the Application Gateway Subnet

Azure Application Gateway **requires** its own dedicated subnet. No other resources may reside in this subnet. Provision `xfusion-apgw-subnet` in the `10.0.2.0/24` range.

```bash
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --name xfusion-apgw-subnet \
  --address-prefix 10.0.2.0/24
```

#### 2.4 Verify Subnet Provisioning

```bash
az network vnet subnet list \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --query "[].{Name:name, Prefix:addressPrefix, State:provisioningState}" \
  --output table
```

**Expected Output:**
```
Name                 Prefix       State
-------------------  -----------  ---------
xfusion-subnet       10.0.1.0/24  Succeeded
xfusion-apgw-subnet  10.0.2.0/24  Succeeded
```

*Screenshots: Subnet list table confirming both subnets provisioned successfully*

---

### Phase 3 - Network Security Group Configuration

#### 3.1 Create the NSG

Create `xfusion-nsg` to enforce inbound traffic rules for the backend VMs. Azure creates a default set of security rules automatically on NSG creation.

```bash
az network nsg create \
  --resource-group $RG \
  --name xfusion-nsg \
  --location eastus
```

**Default rules created automatically:**

| Rule | Direction | Priority | Action |
|---|---|---|---|
| AllowVnetInBound | Inbound | 65000 | Allow |
| AllowAzureLoadBalancerInBound | Inbound | 65001 | Allow |
| DenyAllInBound | Inbound | 65500 | Deny |
| AllowVnetOutBound | Outbound | 65000 | Allow |
| AllowInternetOutBound | Outbound | 65001 | Allow |
| DenyAllOutBound | Outbound | 65500 | Deny |

#### 3.2 Allow HTTP Inbound (Port 80)

Add a custom rule to permit HTTP traffic from any source to port 80. The Application Gateway health probe and load-balanced traffic both require this rule.

```bash
az network nsg rule create \
  --resource-group $RG \
  --nsg-name xfusion-nsg \
  --name allow-http \
  --protocol Tcp \
  --direction Inbound \
  --priority 1000 \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-range 80 \
  --access Allow
```

#### 3.3 Allow SSH Inbound (Port 22)

Add a custom rule to permit SSH access for administrative connectivity. Priority `1010` places this rule after the HTTP rule.

```bash
az network nsg rule create \
  --resource-group $RG \
  --nsg-name xfusion-nsg \
  --name allow-ssh \
  --protocol Tcp \
  --direction Inbound \
  --priority 1010 \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --destination-address-prefix '*' \
  --destination-port-range 22 \
  --access Allow
```

*Screenshot Placeholder: NSG rule list confirming allow-http (1000) and allow-ssh (1010) rules*

---

### Phase 4 - Virtual Machine Deployment

Both VMs are deployed into `xfusion-subnet` with the `xfusion-nsg` attached directly at the NIC level. The `Standard_B1s` SKU is cost-efficient and appropriate for this workload profile.

#### 4.1 Deploy xfusion-vm1

```bash
az vm create \
  --resource-group $RG \
  --name xfusion-vm1 \
  --location eastus \
  --image Ubuntu2204 \
  --vnet-name xfusion-vnet \
  --subnet xfusion-subnet \
  --nsg xfusion-nsg \
  --admin-username azureuser \
  --admin-password "Azure@123456!" \
  --authentication-type password \
  --public-ip-sku Standard \
  --size Standard_B1s \
  --os-disk-size-gb 64 \
  --storage-sku Standard_LRS
```

**Provisioned with:**
* Private IP: `10.0.1.4`
* Public IP: `172.172.178.102`

#### 4.2 Deploy xfusion-vm2

```bash
az vm create \
  --resource-group $RG \
  --name xfusion-vm2 \
  --location eastus \
  --image Ubuntu2204 \
  --vnet-name xfusion-vnet \
  --subnet xfusion-subnet \
  --nsg xfusion-nsg \
  --admin-username azureuser \
  --admin-password "Azure@123456!" \
  --authentication-type password \
  --public-ip-sku Standard \
  --size Standard_B1s \
  --os-disk-size-gb 64 \
  --storage-sku Standard_LRS
```

**Provisioned with:**
* Private IP: `10.0.1.5`
* Public IP: `52.188.116.122`

#### 4.3 Verify VM Status

```bash
az vm list \
  --resource-group $RG \
  --show-details \
  --query "[].{Name:name, State:powerState, Provisioning:provisioningState, PrivateIP:privateIps, PublicIP:publicIps}" \
  --output table
```

**Expected Output:**
```
Name         State       Provisioning    PrivateIP    PublicIP
-----------  ----------  --------------  -----------  ---------------
xfusion-vm1  VM running  Succeeded       10.0.1.4     172.172.178.102
xfusion-vm2  VM running  Succeeded       10.0.1.5     52.188.116.122
```

*Screenshot Placeholder: VM list table confirming both VMs running with correct private IPs*

---

### Phase 5 - Nginx Installation and Content Configuration

The Azure Custom Script Extension is used to bootstrap Nginx on each VM without requiring direct SSH access. This approach is idiomatic for Azure VM initialization and aligns with infrastructure-as-code principles.

#### 5.1 Configure VM1 - Version 1 Content

Install Nginx on `xfusion-vm1` and write the version-specific response to `index.html`:

```bash
az vm extension set \
  --resource-group $RG \
  --vm-name xfusion-vm1 \
  --name customScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{"commandToExecute":"apt-get update -y && apt-get install -y nginx && echo \"Welcome to KKE Labs:Version 1\" > /var/www/html/index.html && systemctl enable nginx && systemctl restart nginx"}'
```

**Key output field to verify:** `"provisioningState": "Succeeded"`

#### 5.2 Configure VM2 - Version 2 Content

Install Nginx on `xfusion-vm2` and write the version-specific response to `index.html`:

```bash
az vm extension set \
  --resource-group $RG \
  --vm-name xfusion-vm2 \
  --name customScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{"commandToExecute":"apt-get update -y && apt-get install -y nginx && echo \"Welcome to KKE Labs:Version 2\" > /var/www/html/index.html && systemctl enable nginx && systemctl restart nginx"}'
```

#### 5.3 Verify Direct VM Responses

Confirm that each VM serves its correct content independently before placing them behind the Application Gateway:

```bash
curl -s http://172.172.178.102
# Expected: Welcome to KKE Labs:Version 1

curl -s http://52.188.116.122
# Expected: Welcome to KKE Labs:Version 2
```

*Screenshot Placeholder: curl output confirming Version 1 from vm1 and Version 2 from vm2 directly*

---

### Phase 6 - Public IP Provisioning for Application Gateway

Provision a Static Standard SKU public IP for the Application Gateway frontend. A static allocation ensures the gateway IP remains stable across restarts and reconfigurations.

```bash
az network public-ip create \
  --resource-group $RG \
  --name xfusion-apgw-ip \
  --location eastus \
  --sku Standard \
  --allocation-method Static
```

**Allocated IP:** `20.127.87.36`

Store the IP and required resource IDs for use in the Application Gateway deployment:

```bash
APGW_PUBLIC_IP=$(az network public-ip show \
  --resource-group $RG \
  --name xfusion-apgw-ip \
  --query "ipAddress" --output tsv)

SUBSCRIPTION_ID=$(az account show --query id --output tsv)

SUBNET_ID=$(az network vnet subnet show \
  --resource-group $RG \
  --vnet-name xfusion-vnet \
  --name xfusion-apgw-subnet \
  --query id --output tsv)

APGW_IP_ID=$(az network public-ip show \
  --resource-group $RG \
  --name xfusion-apgw-ip \
  --query id --output tsv)
```

**Verify resolved values:**
```
App Gateway IP : 20.127.87.36
VM1 IP         : 10.0.1.4
VM2 IP         : 10.0.1.5
```

*Screenshot Placeholder: Variable echo output confirming Subscription ID, Subnet ID, IP ID, and VM private IPs*

---

### Phase 7 - Application Gateway Deployment

The Application Gateway is deployed via `az rest` with a complete ARM JSON body. This approach provides full control over all resource sub-components in a single atomic API call.

The gateway configuration includes the following named components:

| Component | Name | Details |
|---|---|---|
| Gateway IP Config | `appGatewayIpConfig` | Bound to `xfusion-apgw-subnet` |
| Frontend IP | `xfusion-apgw-ip` | Bound to the static public IP |
| Frontend Port | `port_80` | TCP port 80 |
| Backend Pool | `xfusion-backend-pool` | Backend addresses: `10.0.1.4`, `10.0.1.5` |
| HTTP Settings | `xfusion-http-settings` | HTTP, port 80, 30s timeout |
| Listener | `xfusion-listener` | HTTP on `xfusion-apgw-ip:port_80` |
| Routing Rule | `xfusion-routing-rule` | Basic, priority 100 |

```bash
az rest \
  --method PUT \
  --url "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RG}/providers/Microsoft.Network/applicationGateways/xfusion-apgw?api-version=2023-05-01" \
  --body "{
    \"location\": \"eastus\",
    \"properties\": {
      \"sku\": {
        \"name\": \"Basic\",
        \"tier\": \"Basic\",
        \"capacity\": 1
      },
      \"gatewayIPConfigurations\": [
        {
          \"name\": \"appGatewayIpConfig\",
          \"properties\": {
            \"subnet\": { \"id\": \"${SUBNET_ID}\" }
          }
        }
      ],
      \"frontendIPConfigurations\": [
        {
          \"name\": \"xfusion-apgw-ip\",
          \"properties\": {
            \"publicIPAddress\": { \"id\": \"${APGW_IP_ID}\" }
          }
        }
      ],
      \"frontendPorts\": [
        {
          \"name\": \"port_80\",
          \"properties\": { \"port\": 80 }
        }
      ],
      \"backendAddressPools\": [
        {
          \"name\": \"xfusion-backend-pool\",
          \"properties\": {
            \"backendAddresses\": [
              { \"ipAddress\": \"10.0.1.4\" },
              { \"ipAddress\": \"10.0.1.5\" }
            ]
          }
        }
      ],
      \"backendHttpSettingsCollection\": [
        {
          \"name\": \"xfusion-http-settings\",
          \"properties\": {
            \"port\": 80,
            \"protocol\": \"Http\",
            \"cookieBasedAffinity\": \"Disabled\",
            \"requestTimeout\": 30
          }
        }
      ],
      \"httpListeners\": [
        {
          \"name\": \"xfusion-listener\",
          \"properties\": {
            \"frontendIPConfiguration\": {
              \"id\": \"/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RG}/providers/Microsoft.Network/applicationGateways/xfusion-apgw/frontendIPConfigurations/xfusion-apgw-ip\"
            },
            \"frontendPort\": {
              \"id\": \"/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RG}/providers/Microsoft.Network/applicationGateways/xfusion-apgw/frontendPorts/port_80\"
            },
            \"protocol\": \"Http\"
          }
        }
      ],
      \"requestRoutingRules\": [
        {
          \"name\": \"xfusion-routing-rule\",
          \"properties\": {
            \"ruleType\": \"Basic\",
            \"priority\": 100,
            \"httpListener\": {
              \"id\": \"/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RG}/providers/Microsoft.Network/applicationGateways/xfusion-apgw/httpListeners/xfusion-listener\"
            },
            \"backendAddressPool\": {
              \"id\": \"/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RG}/providers/Microsoft.Network/applicationGateways/xfusion-apgw/backendAddressPools/xfusion-backend-pool\"
            },
            \"backendHttpSettings\": {
              \"id\": \"/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RG}/providers/Microsoft.Network/applicationGateways/xfusion-apgw/backendHttpSettingsCollection/xfusion-http-settings\"
            }
          }
        }
      ]
    }
  }"
```

#### 7.1 Poll for Gateway Readiness

Application Gateway provisioning is asynchronous and typically takes 4 to 10 minutes. Poll until the `provisioningState` transitions to `Succeeded`:

```bash
echo "Polling every 30s..."
for i in $(seq 1 30); do
  STATE=$(az network application-gateway show \
    --resource-group $RG \
    --name xfusion-apgw \
    --query "provisioningState" --output tsv 2>/dev/null)
  echo "Attempt $i: $STATE"
  [ "$STATE" = "Succeeded" ] && echo "Gateway READY!" && break
  sleep 30
done
```

**Observed:** The gateway reached `Succeeded` on attempt 9 (approximately 4 minutes).

*Screenshot Placeholder: Polling loop output showing Updating states followed by Succeeded on attempt 9*

---

### Phase 8 - Validation and Traffic Distribution Testing

#### 8.1 Confirm Gateway Operational State

```bash
az network application-gateway show \
  --resource-group $RG \
  --name xfusion-apgw \
  --query "{Name:name, SKU:sku.name, Provisioning:provisioningState, Operational:operationalState}" \
  --output table
```

**Expected Output:**
```
Name          SKU    Provisioning    Operational
------------  -----  --------------  -------------
xfusion-apgw  Basic  Succeeded       Running
```

#### 8.2 Verify Frontend IP Configuration

```bash
az network application-gateway frontend-ip list \
  --resource-group $RG \
  --gateway-name xfusion-apgw \
  --query "[].{Name:name}" \
  --output table
```

**Expected Output:**
```
Name
---------------
xfusion-apgw-ip
```

#### 8.3 Confirm Backend Health

Both backend pool members must report as `Healthy` before traffic distribution can be verified:

```bash
az network application-gateway show-backend-health \
  --resource-group $RG \
  --name xfusion-apgw \
  --query "backendAddressPools[0].backendHttpSettingsCollection[0].servers[*].{Address:address, Health:health}" \
  --output table
```

**Expected Output:**
```
Address    Health
---------  --------
10.0.1.4   Healthy
10.0.1.5   Healthy
```

*Screenshot Placeholder: Backend health table showing 10.0.1.4 and 10.0.1.5 both as Healthy*

#### 8.4 Validate Round-Robin Load Balancing

Send 10 sequential HTTP requests to the Application Gateway public IP and confirm that responses alternate between both VM versions:

```bash
for i in $(seq 1 10); do
  echo -n "Request $i: "
  curl -s --max-time 10 http://$APGW_PUBLIC_IP
  echo ""
  sleep 1
done
```

**Observed Output:**
```
Request 1:  Welcome to KKE Labs:Version 2
Request 2:  Welcome to KKE Labs:Version 1
Request 3:  Welcome to KKE Labs:Version 2
Request 4:  Welcome to KKE Labs:Version 1
Request 5:  Welcome to KKE Labs:Version 1
Request 6:  Welcome to KKE Labs:Version 2
Request 7:  Welcome to KKE Labs:Version 2
Request 8:  Welcome to KKE Labs:Version 1
Request 9:  Welcome to KKE Labs:Version 2
Request 10: Welcome to KKE Labs:Version 1
```

Both VMs appear in the response set, confirming that the Application Gateway is distributing traffic across the backend pool.

*Screenshot Placeholder: curl loop output showing alternating Version 1 and Version 2 responses across 10 requests*

---

## Resource Summary

| Resource | Name | Type | Details |
|---|---|---|---|
| Virtual Network | `xfusion-vnet` | VNet | 10.0.0.0/16, East US |
| VM Subnet | `xfusion-subnet` | Subnet | 10.0.1.0/24 |
| Gateway Subnet | `xfusion-apgw-subnet` | Subnet | 10.0.2.0/24 |
| Network Security Group | `xfusion-nsg` | NSG | Allow TCP 80, 22 Inbound |
| Virtual Machine 1 | `xfusion-vm1` | VM | Ubuntu 22.04, 10.0.1.4, Standard_B1s |
| Virtual Machine 2 | `xfusion-vm2` | VM | Ubuntu 22.04, 10.0.1.5, Standard_B1s |
| Gateway Public IP | `xfusion-apgw-ip` | Public IP | 20.127.87.36, Static, Standard SKU |
| Application Gateway | `xfusion-apgw` | App Gateway | Basic SKU, Capacity 1, East US |

---

## Best Practices Applied

**Subnet Isolation:** The Application Gateway subnet (`xfusion-apgw-subnet`) is fully dedicated to the gateway, satisfying Azure's hard requirement that no other resource types coexist in this subnet. Using a separate `/24` block also ensures sufficient IP headroom for gateway scaling.

**Environment Variable Driven Execution:** The resource group name and all ARM resource IDs are resolved dynamically via CLI queries and stored in shell variables. This eliminates hardcoded values, reduces human error, and makes the script portable across environments.

**VM Extension for Bootstrap Configuration:** The Azure Custom Script Extension (`Microsoft.Azure.Extensions`) was used to install Nginx and configure content on each VM. This approach avoids the need for a bastion host or direct SSH during deployment and aligns with cloud-native VM initialization patterns.

**Static Public IP for Gateway Frontend:** A Static allocation was chosen for `xfusion-apgw-ip` rather than Dynamic. Static IPs are strongly preferred for production gateways since they remain stable across gateway stops, starts, and maintenance events, and can be pre-registered in DNS.

**Asynchronous Provisioning with Polling Loop:** Application Gateway creation is long-running and returns immediately with `provisioningState: Updating`. A controlled polling loop with 30-second intervals was implemented to detect readiness deterministically, avoiding arbitrary `sleep` calls or manual portal checks.

**Backend Health Verification Before Traffic Testing:** The `show-backend-health` command was used to confirm that both backend VMs reported `Healthy` before executing load balancing validation. This prevents false negatives in traffic tests caused by a backend still initializing.

**NSG Rules with Explicit Priorities:** Custom NSG rules were created with deliberate priority spacing (1000 for HTTP, 1010 for SSH), leaving room to insert additional rules in between without renumbering.

---

## Lessons Learned

**Application Gateway Subnet Must Be Dedicated**
Azure enforces that the subnet assigned to an Application Gateway contains no other resource types. Attempting to deploy the gateway into `xfusion-subnet` alongside the VMs would result in a `GatewaySubnetCannotBeUsedForOtherResources` error. Always pre-provision a dedicated subnet for the gateway before starting the deployment.

**ARM REST API Preferred Over az network application-gateway create for Complex Configurations**
The `az network application-gateway create` CLI command requires all sub-components (listener, routing rule, backend settings) to be defined atomically and can become unwieldy for full configurations. Using `az rest` with a structured ARM JSON body provides explicit control over every named component and avoids implicit naming from CLI shortcuts. This approach also matches what ARM templates and Bicep files produce internally, making it easier to transition to IaC.

**Provisioning Takes 4 to 10 Minutes - Plan Accordingly**
In this deployment, the gateway reached `Succeeded` on polling attempt 9, approximately 4.5 minutes after the API call returned. Any pipeline or automation that proceeds immediately after the creation command will fail. The polling loop pattern is essential and should be incorporated into any scripted or CI/CD deployment.

**Cookie-Based Affinity Disabled for True Round-Robin**
Setting `cookieBasedAffinity: Disabled` in the backend HTTP settings ensures that the gateway does not pin sessions to a single backend using sticky cookies. Without this setting, repeated requests from the same client would consistently land on the same VM, making load balancing validation appear broken.

**NSG Must Allow Port 80 on VM Subnet for Gateway Health Probes**
The Application Gateway sends TCP health probe traffic from its own subnet to the VMs on port 80. If the NSG on the VM subnet blocks this traffic, both backends will report `Unhealthy` and receive no traffic. The `allow-http` rule added to `xfusion-nsg` resolves this by permitting all inbound TCP traffic on port 80.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Backend shows `Unhealthy` | NSG blocking port 80 from gateway subnet | Verify `allow-http` rule exists on `xfusion-nsg` with priority less than `DenyAllInBound` |
| Gateway stuck in `Updating` | Normal provisioning latency | Continue polling; allow up to 10 minutes |
| All traffic returns same version | Cookie-based affinity enabled | Set `cookieBasedAffinity: Disabled` in backend HTTP settings |
| `GatewaySubnetCannotBeUsedForOtherResources` | VM placed in gateway subnet | Ensure VMs use `xfusion-subnet` and gateway uses `xfusion-apgw-subnet` exclusively |
| `curl` returns connection timeout | Gateway not yet in `Running` operational state | Wait for `operationalState: Running` before testing |
| VM extension `provisioningState: Failed` | Nginx install error inside VM | SSH to VM and check `/var/log/azure/custom-script/handler.log` |












<img width="1032" height="856" alt="image" src="https://github.com/user-attachments/assets/64cb05f7-9a9a-4631-9f1a-a273829c0a8b" />
<img width="1037" height="762" alt="image" src="https://github.com/user-attachments/assets/d3270981-5700-49b3-a086-1590faee15ec" />
<img width="1034" height="590" alt="image" src="https://github.com/user-attachments/assets/f9877439-de44-43b6-8515-719c7157c858" />
<img width="1030" height="862" alt="image" src="https://github.com/user-attachments/assets/09639fe9-f235-4842-9c20-933fa964452f" />
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/c23aaa52-361c-45a2-9521-83f38c05d56b" />
<img width="1033" height="857" alt="image" src="https://github.com/user-attachments/assets/b0c23406-4186-4bf7-bb09-6b76955b36df" />
<img width="1033" height="861" alt="image" src="https://github.com/user-attachments/assets/a5d9722d-753d-4bcd-9f53-b783c1b1ba05" />
<img width="1035" height="860" alt="image" src="https://github.com/user-attachments/assets/2c5d794e-d7da-4391-b447-aa252a77b3d1" />
<img width="1032" height="860" alt="image" src="https://github.com/user-attachments/assets/bd41a7dc-6fff-4dfd-971a-32d9199dfa51" />
<img width="1031" height="860" alt="image" src="https://github.com/user-attachments/assets/3ef58efe-3871-4dcc-935b-585ec360abe5" />
<img width="1035" height="864" alt="image" src="https://github.com/user-attachments/assets/11d9fc00-7670-4d22-bfed-6563c8950dcb" />
<img width="1031" height="768" alt="image" src="https://github.com/user-attachments/assets/1d9dc4af-6544-4592-92b5-54933e459135" />
<img width="1034" height="867" alt="image" src="https://github.com/user-attachments/assets/3bbbd73b-60b3-4f0d-82ba-f56a6db7512f" />
<img width="1035" height="852" alt="image" src="https://github.com/user-attachments/assets/2f075281-4bf8-408e-9d96-b92f7cbb0823" />
<img width="1036" height="807" alt="image" src="https://github.com/user-attachments/assets/6197d711-95c6-447a-828a-79e4f2567538" />
<img width="1030" height="818" alt="image" src="https://github.com/user-attachments/assets/64686ee8-2c89-41e3-8579-0c58d4a456e5" />
<img width="1033" height="842" alt="image" src="https://github.com/user-attachments/assets/7e2cabfb-09a5-41dc-b1c7-485eb556fd2c" />
<img width="1037" height="671" alt="image" src="https://github.com/user-attachments/assets/4e54b550-d3cf-4016-a9eb-8ae579ba3837" />
<img width="1102" height="842" alt="image" src="https://github.com/user-attachments/assets/eb9e623c-2613-4b9c-b39f-50e7feb3ed7c" />
<img width="1105" height="865" alt="image" src="https://github.com/user-attachments/assets/0e294a29-377e-4091-966b-6059d0dafe5e" />
<img width="1100" height="867" alt="image" src="https://github.com/user-attachments/assets/590929ca-2ebd-4ae2-830a-8189150d3419" />
<img width="1105" height="871" alt="image" src="https://github.com/user-attachments/assets/06ae9fb6-5f3e-4f8e-8e76-a3f51f981fe2" />
<img width="1106" height="862" alt="image" src="https://github.com/user-attachments/assets/f068ee15-79ce-432b-8bf4-342aeccd5868" />
<img width="1103" height="864" alt="image" src="https://github.com/user-attachments/assets/1e762199-8d27-4b25-9bc6-38b68bf4cb99" />
<img width="1106" height="861" alt="image" src="https://github.com/user-attachments/assets/bdfd78a7-c161-49c7-aaa6-7cb2df7dd442" />
<img width="1105" height="638" alt="image" src="https://github.com/user-attachments/assets/3091b60a-55ba-4c23-9b59-e837c8d7c2af" />
<img width="1102" height="802" alt="image" src="https://github.com/user-attachments/assets/9febfb11-fbd5-4de1-8827-2bd43367e2eb" />
<img width="1098" height="578" alt="image" src="https://github.com/user-attachments/assets/78daaa0b-113c-4bf1-a614-a8452afe1a87" />
<img width="1106" height="860" alt="image" src="https://github.com/user-attachments/assets/1b965452-27f8-43cb-9d65-da74ddcb97de" />
