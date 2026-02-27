# Azure ARM Virtual Network Deployment – Datacenter Standard

## Project Overview
This project documents the modification and deployment of an Azure Virtual Network using an ARM template. The goal was to standardize naming, update address space configuration, apply additional environment tagging, and deploy the infrastructure using Azure CLI in a controlled, repeatable manner.

---

## Scope of Work
- Modify existing ARM template for Virtual Network
- Update VNet name and displayName tag
- Change address space to datacenter CIDR range
- Add environment-specific tagging
- Deploy ARM template using Azure CLI
- Validate successful provisioning

---

## Prerequisites
- Azure CLI installed
- Authenticated Azure session
- Access to ARM template files
- Existing Azure Resource Group

---

## Step 1: Verify Azure Account Context
```
az account show
```

Screenshot:
<img width="1031" height="544" alt="image" src="https://github.com/user-attachments/assets/910fba49-f5ae-48a1-a16d-c8dcd7f39965" />

## Step 2: Navigate to ARM Templates Directory
```
cd /root/arm-templates
```
Screenshot:

<img width="1031" height="546" alt="image" src="https://github.com/user-attachments/assets/4698b010-f4d0-4bb5-9df2-589054894d96" />

## Step 3: Confirm ARM Template File Exists
```
ls -l vnet-deployment-template.json
```
Screenshot:
<img width="1033" height="612" alt="image" src="https://github.com/user-attachments/assets/d0779c4b-224e-41c7-8051-1b54b7e0e494" />

## Step 4: Identify Target Resource Group
```
az group list --query '[].name' --output table | grep 'kml'
```
Screenshot:

**

## Step 5: Update Virtual Network Name and displayName Tag
```
sed -i 's/"name": ".*"/"name": "arm-vnet-datacenter"/g' vnet-deployment-template.json

sed -i 's/"displayName": ".*"/"displayName": "arm-vnet-datacenter"/g' vnet-deployment-template.json
```
Screenshot:


## Step 6: Update Address Space
```
sed -i 's/"addressPrefixes": \[ ".*" \]/"addressPrefixes": [ "192.168.0.0\/16" ]/g' vnet-deployment-template.json
```
Screenshot:

**

## Step 7: Add Environment Tag
`sed -i '/"displayName": "arm-vnet-datacenter"/a \                "Environment": "KKE-datacenter"' vnet-deployment-template.json`

Screenshot:

## Step 8: Validate and Fix JSON Formatting (If Required)
```
vi vnet-deployment-template.json
```
Ensure proper comma placement and valid JSON structure before deployment.

Screenshot:

**

## Step 9: Deploy ARM Template
```
az deployment group create \
  --resource-group kml_rg_main-7dd447df1f3c4b74 \
  --template-file vnet-deployment-template.json
```
Screenshot:

## Step 10: Verify Virtual Network Deployment
```
az network vnet show \
  --resource-group kml_rg_main-7dd447df1f3c4b74 \
  --name arm-vnet-datacenter \
  --query '{Name:name, Address:addressSpace.addressPrefixes, Tags:tags}'
```

Screenshot:

**

## Validation Results
```
Virtual Network Name: arm-vnet-datacenter

Address Space: 192.168.0.0/16
```
#### Tags Applied:
```
displayName: arm-vnet-datacenter

Environment: KKE-datacenter

Deployment State: Succeeded
```
## Outcome

The ARM template was successfully modified and deployed. The Virtual Network now conforms to datacenter naming standards, uses the correct CIDR range, and includes environment-level tagging for governance and visibility.




<img width="1031" height="696" alt="image" src="https://github.com/user-attachments/assets/53dd6bad-82ce-42f7-810e-8ea459223d08" />
<img width="1033" height="640" alt="image" src="https://github.com/user-attachments/assets/607b23a4-a98b-4da0-b05f-815202a04276" />
<img width="1033" height="474" alt="image" src="https://github.com/user-attachments/assets/7a807a9f-9b2f-4117-acda-8ec340dc6844" />
<img width="1034" height="754" alt="image" src="https://github.com/user-attachments/assets/6621f67f-185a-491a-ba08-f7c691c98a94" />
<img width="1037" height="851" alt="image" src="https://github.com/user-attachments/assets/e93b790b-3a39-4644-87b0-37ade4cf5cf5" />
<img width="1034" height="522" alt="image" src="https://github.com/user-attachments/assets/409e06a8-19c1-44bd-a655-6c14b57ebf32" />
<img width="1031" height="863" alt="image" src="https://github.com/user-attachments/assets/dca35f19-9bfa-4d10-adfe-87af52b733b5" />
<img width="1035" height="851" alt="image" src="https://github.com/user-attachments/assets/1726c5c9-e10f-464e-8bd4-3dca19a10e7a" />
<img width="1031" height="858" alt="image" src="https://github.com/user-attachments/assets/54ef6b46-e35b-4075-b0cc-0163ebdec926" />
<img width="1030" height="861" alt="image" src="https://github.com/user-attachments/assets/2877d760-6c6a-4c45-9667-ce93da9ce880" />




