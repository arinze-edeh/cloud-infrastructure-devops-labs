# Azure Virtual Network Creation (VNet)

## 📌 Lab Overview
- This lab demonstrates how to create an Azure Virtual Network (VNet)
using the Azure CLI as part of a phased cloud migration strategy.

- The VNet is created in the Central US region using an IPv4 CIDR block.

---

## 🎯 Objective
- Authenticate to Azure using Azure CLI
- Create a resource group (if required)
- Create a Virtual Network named `xfusion-vnet`
- Verify successful creation

---

## 🧠 High-Level Flow

- AUTHENTICATE to Azure
- VERIFY active subscription

- CHECK for existing resource group
- IF resource group does not exist
  -  CREATE resource group in centralus
- END IF

- CREATE Virtual Network with IPv4 CIDR
- VERIFY VNet exists and is active

## 🛠️ Implementation Steps
## Step 1: Retrieve Azure Credentials
- `showcreds`

📸 screenshot:
<img width="1022" height="742" alt="548048236-82eb4020-f4c5-493e-9842-c7954aa9701b" src="https://github.com/user-attachments/assets/f5fd2742-f96f-42cd-8fb0-397fdc26e930" />

## Step 2: Verify Azure Login
- `az account show`

📸 screenshot:
<img width="1030" height="615" alt="image" src="https://github.com/user-attachments/assets/8c21f6a8-2a63-41ff-9920-8677ad27b3f8" />

## Step 3: Check or Create Resource Group
- `az group list --output table
az group create \
  --name xfusion-rg \
  --location centralus`
  
📸 screenshot:
<img width="1027" height="619" alt="image" src="https://github.com/user-attachments/assets/5dc71056-a1e5-4a5f-b45d-082e0807d91f" />

## Step 4: Create Virtual Network
- `az network vnet create \
  --resource-group kml_rg_main-2de2f556a3f04cf0 \
  --name xfusion-vnet \
  --location centralus \
  --address-prefixes 10.0.0.0/16`

📸 screenshot:
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/918a2d0f-0529-46e2-92cf-22a7717aea28" />

## Step 5: Verify Virtual Network
- `az network vnet show \
  --resource-group kml_rg_main-2de2f556a3f04cf0 \
  --name xfusion-vnet \
  --output table`

📸 screenshot:
<img width="1392" height="857" alt="image" src="https://github.com/user-attachments/assets/bfe2c57c-16de-402e-b268-29f745fe5b1c" />

## ✅ Final Result

- Virtual Network xfusion-vnet successfully created

- IPv4 address space assigned

- Resource deployed in centralus region

- Task completed using Azure CLI only

## 🏷️ Tags
`azure` `networking` `vnet` `azure-cli` `cloud-migration`









