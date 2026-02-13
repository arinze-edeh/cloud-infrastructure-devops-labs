# Azure Virtual Network & Subnet Creation (Azure CLI)

## 📌 Lab Overview

- The Nautilus DevOps team is migrating infrastructure to Azure using
incremental steps. As part of the foundational networking layer,
a Virtual Network (VNet) and subnet were provisioned using the
Azure CLI within an existing resource group.

## 🎯 Objectives

- Authenticate to Azure using provided lab credentials

- Identify existing resource group

- Create a Virtual Network (VNet)

- Create a subnet during VNet provisioning

- Configure IPv4 address space correctly

- Deploy resources in the correct Azure region

## 🧠 High-Level Logic
- CONNECT to Azure CLI host

- RETRIEVE Azure credentials using showcreds
- AUTHENTICATE to Azure session

- LIST available resource groups
- SELECT existing resource group

- CREATE Virtual Network
  -  SET VNet name = nautilus-vnet
  -  SET region = southcentralus
  -  SET address space = 10.0.0.0/16

- CREATE subnet during VNet creation
  -  SET subnet name = nautilus-subnet
  -  SET subnet CIDR = 10.0.1.0/24

## 🛠️ Implementation Steps

## Step 1: Retrieve Azure Credentials
- showcreds

📸 screenshot:
<img width="1031" height="551" alt="549549591-d3519e2b-60e8-48c1-9aff-7270941064d9" src="https://github.com/user-attachments/assets/a96a79f3-ca3f-4150-a57f-3d9386a97dc2" />

## Step 2: List Existing Resource Groups
- az group list --output table

📸 screenshot:
<img width="1056" height="484" alt="image" src="https://github.com/user-attachments/assets/c40da632-59aa-4ee6-80cf-85002879e540" />

## Step 3: Create Virtual Network and Subnet
- az network vnet create \
  -  --name nautilus-vnet \
  -  --resource-group kml_rg_main-0e6ddfb2b553417d \
  -  --location southcentralus \
  -  --address-prefix 10.0.0.0/16 \
  -  --subnet-name nautilus-subnet \
  -  --subnet-prefix 10.0.1.0/24

📸 screenshot:
<img width="1033" height="832" alt="image" src="https://github.com/user-attachments/assets/3e4dd377-2ab9-4ea0-a88e-dfffd2aac696" />
<img width="1036" height="867" alt="image" src="https://github.com/user-attachments/assets/b89647e9-8a49-4510-acd7-7b72cacde9c4" />

## ✅ Final Outcome

- Virtual Network nautilus-vnet successfully created

- IPv4 address space configured as 10.0.0.0/16

- Subnet nautilus-subnet created with 10.0.1.0/24

- Resources deployed in southcentralus

- Infrastructure ready for future Azure workloads

## 🏷️ Tags
`azure` `cli` `networking` `vnet` `subnet` `cloud` `infrastructure` `devops`
