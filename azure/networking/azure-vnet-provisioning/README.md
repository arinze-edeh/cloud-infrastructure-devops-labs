# Azure Infrastructure: Provisioning Virtual Networks via Azure CLI

## 📌 Lab Overview
- As part of a phased migration to Microsoft Azure, the Nautilus DevOps team
provisioned a Virtual Network (VNet) to serve as the foundational
networking layer for upcoming Azure services.

- This lab demonstrates how to create a Virtual Network using the Azure CLI.

---

## 🎯 Objectives
- Create a Virtual Network in Azure
- Define an IPv4 CIDR block
- Deploy the VNet in the correct region
- Validate successful creation

---

## 🧠 High-Level Logic (Pseudo-Code)

- AUTHENTICATE to Azure
- VERIFY subscription and region

- CREATE resource group
- CREATE virtual network with CIDR block

- VERIFY VNet exists

## 🛠️ Implementation Steps

## Step 1: Verify Azure CLI
- az version

📸 screenshot:

<img width="1033" height="647" alt="image" src="https://github.com/user-attachments/assets/7252cdbb-b2d0-426a-85cf-3db13d5684c6" />

## Step 2: Confirm Azure Login
- az account show

📸 screenshot:
<img width="1032" height="689" alt="image" src="https://github.com/user-attachments/assets/c1a716b1-39d4-41aa-b255-9c761d3f51ca" />

## Step 3: Select Resource Group
- az group create \
  -  --name nautilus-rg \
  -  --location eastus

📸 screenshot:
<img width="1031" height="737" alt="image" src="https://github.com/user-attachments/assets/38d93dde-6d8a-4b50-86f6-29626959ec50" />
<img width="1028" height="738" alt="image" src="https://github.com/user-attachments/assets/ae9f80a3-7081-426f-b8f4-42d3c20ee9b1" />

## Step 4: Create Virtual Network
- az network vnet create \
  -  --resource-group kml_rg_main-d61c7f18084c43a8 \
  -  --name nautilus-vnet \
  -  --address-prefix 192.168.0.0/24 \
  -  --location eastus

📸 screenshots:
<img width="1025" height="879" alt="image" src="https://github.com/user-attachments/assets/39a626f2-8104-42c3-b0eb-7f5527b44643" />
<img width="1036" height="868" alt="image" src="https://github.com/user-attachments/assets/44ec61f8-70c0-4189-993d-64bee992cbfb" />

## Step 5: Verify VNet
- az network vnet show \
  -  --resource-group kml_rg_main-d61c7f18084c43a8 \
  -  --name nautilus-vnet \
  -  --output table

📸 screenshot:
<img width="1399" height="803" alt="image" src="https://github.com/user-attachments/assets/04b5bfff-7976-4607-b385-9fdb0c49062f" />

## ✅ Final Outcome
- Virtual Network nautilus-vnet created successfully

- CIDR block 192.168.0.0/24 assigned

- VNet deployed in eastus region

## 🏷️ Tags
`azure` `vnet` `networking` `cloud-infrastructure` `devops`
