# Azure Public IP Allocation (devops-pip)

## 📌 Lab Overview
- The Nautilus DevOps team is migrating infrastructure to Azure using
incremental and controlled phases. As part of the networking foundation,
a Public IP address was allocated to support external connectivity
for future Azure resources.

- This lab focuses on allocating a Public IP using Azure CLI
while handling SKU and allocation constraints.

---

## 🎯 Objectives
- Retrieve Azure access credentials
- Identify the active resource group
- Create a Public IP named `devops-pip`
- Resolve allocation method constraints
- Verify successful Public IP provisioning

---

## 🧠 High-Level Logic (Pseudo-Code)

- CONNECT to azure-client host
- RETRIEVE Azure credentials using showcreds

- LOGIN to Azure subscription
- LIST existing resource groups
- SELECT active resource group

- ATTEMPT to create Public IP with Dynamic allocation
- IF allocation fails due to SKU restriction:
  -  RECREATE Public IP using Static allocation and Standard SKU

- VERIFY Public IP exists
- CONFIRM provisioning state = Succeeded

## 🛠️ Implementation Steps

## Step 1: Retrieve Azure Credentials
- Run credentials command on the azure-client host:

- showcreds

📸 screenshots/showcreds-output.png

## Step 2: Identify Resource Group
- List all resource groups in the subscription:

- az group list --output table

📸 screenshots/resource-group-list.png

## Step 3: Attempt Public IP Creation (Dynamic Allocation)
- Initial attempt using Dynamic allocation:

- az network public-ip create \
  -  --name devops-pip \
  -  --resource-group kml_rg_main-6b125c0390d64b55 \
  -  --allocation-method Dynamic
EXPECTED RESULT:

Command fails due to Standard SKU requiring Static allocation

📸 screenshots/public-ip-dynamic-failure.png

## Step 4: Create Public IP with Static Allocation (Corrected)
- Recreate Public IP using Static allocation and Standard SKU:

- az network public-ip create \
  -  --name devops-pip \
  -  --resource-group kml_rg_main-6b125c0390d64b55 \
  -  --allocation-method Static \
  -  --sku Standard
📸 screenshots/public-ip-static-create.png

## Step 5: Verify Public IP Configuration
- Confirm Public IP details and provisioning state:

- az network public-ip show \
  -  --name devops-pip \
  -  --resource-group kml_rg_main-6b125c0390d64b55
📸 screenshots/public-ip-verify.png

## ✅ Final Outcome
- Public IP devops-pip successfully created

- Allocation method set to Static

- SKU set to Standard

- IPv4 address assigned

- Provisioning state marked as Succeeded

- Resource ready for attachment to Azure services

🏷️ Tags
`azure` `networking` `public-ip` `cloud-migration` `devops` `azure-cli`

<img width="1031" height="319" alt="image" src="https://github.com/user-attachments/assets/88c04b3d-296a-4572-a54f-6e8f54ec5d66" />
<img width="1037" height="573" alt="image" src="https://github.com/user-attachments/assets/6bfa2a5b-4911-40b1-8495-14a84f29cf5e" />
<img width="1031" height="696" alt="image" src="https://github.com/user-attachments/assets/74256c3c-dbe8-4a92-941c-6e18efcdb916" />
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/d86ce8a9-9f96-4f51-9c17-feb916c3d10b" />




