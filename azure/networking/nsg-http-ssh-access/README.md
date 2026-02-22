# Azure Network Security Group (NSG) – HTTP & SSH Access

## Overview
- This project demonstrates the creation and configuration of an **Azure Network Security Group (NSG)** using the **Azure CLI** as part of the Nautilus infrastructure migration strategy.  
- The NSG enforces inbound access control by explicitly allowing **HTTP (port 80)** and **SSH (port 22)** traffic while relying on Azure’s default deny rules for all other inbound traffic.

- The implementation follows **cloud-native, least-privilege, and automation-first** principles expected in FAANG-level environments.

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

Azure CLI installed

Active Azure subscription

Authenticated Azure CLI session

## Step 1: Verify Azure Authentication

Confirm the active Azure account and subscription context.

az account show

Expected Outcome

Subscription state is Enabled

Correct tenant and subscription are active

📸 SCREENSHOT: Azure CLI authenticated account details

## Step 2: Identify Target Resource Group

Retrieve the resource group used for NSG deployment.

RG_NAME=$(az group list --query "[0].name" -o tsv)
echo "Your Resource Group is: $RG_NAME"

📸 SCREENSHOT: Resource group resolution output

## Step 3: Create the Network Security Group

Create the NSG within the identified resource group.

az network nsg create \
  --resource-group $RG_NAME \
  --name nautilus-nsg

Result

NSG is created with Azure default inbound and outbound rules

Provisioning state is Succeeded

📸 SCREENSHOT: NSG creation success output

## Step 4: Add Inbound Rule – Allow HTTP (Port 80)

Allow inbound HTTP traffic from any source.

az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-HTTP \
  --priority 100 \
  --destination-port-ranges 80 \
  --access Allow \
  --protocol Tcp

📸 SCREENSHOT: Allow-HTTP inbound rule configuration

## Step 5: Add Inbound Rule – Allow SSH (Port 22)

Allow inbound SSH access for secure remote administration.

az network nsg rule create \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --name Allow-SSH \
  --priority 110 \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

📸 SCREENSHOT: Allow-SSH inbound rule configuration

## Step 6: Validate NSG Rules

Verify that the inbound rules are correctly applied and prioritized.

az network nsg rule list \
  --resource-group $RG_NAME \
  --nsg-name nautilus-nsg \
  --output table

Expected Output

Allow-HTTP → Priority 100 → Port 80

Allow-SSH → Priority 110 → Port 22

📸 SCREENSHOT: Final NSG inbound rules table

## Automation Script

To ensure repeatability and consistency, the entire configuration was automated using a Bash script.

File: az-nsg-nautilus.sh

#!/bin/bash
# Azure Network Security Group Automation Script
# Project: Nautilus Infrastructure Migration

RG_NAME="kml_rg_main-f56fa9d90d7842e1"
NSG_NAME="nautilus-nsg"

echo "Creating NSG..."
az network nsg create -g $RG_NAME -n $NSG_NAME

echo "Adding Inbound Rules..."
az network nsg rule create -g $RG_NAME --nsg-name $NSG_NAME -n Allow-HTTP --priority 100 --destination-port-ranges 80 --access Allow --protocol Tcp
az network nsg rule create -g $RG_NAME --nsg-name $NSG_NAME -n Allow-SSH --priority 110 --destination-port-ranges 22 --access Allow --protocol Tcp

echo "Final Configuration:"
az network nsg rule list -g $RG_NAME --nsg-name $NSG_NAME --output table

📸 SCREENSHOT: Script execution and final rule verification

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


<img width="1034" height="556" alt="image" src="https://github.com/user-attachments/assets/6665d16e-1a21-4a8a-a45c-f257e0430949" />
<img width="1033" height="629" alt="image" src="https://github.com/user-attachments/assets/15612c68-a301-48ff-bbdf-7a7899224078" />
<img width="1034" height="870" alt="image" src="https://github.com/user-attachments/assets/1a4497d3-66a0-4072-b6c0-c4549f39f23f" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/8c18d50c-366a-4d35-87da-34ef00b3c2bc" />
<img width="1034" height="869" alt="image" src="https://github.com/user-attachments/assets/7c5520f2-0b7d-4d10-aace-8abf58acafc4" />
<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/1e462635-2045-4919-9727-2cc31023c396" />
<img width="1038" height="866" alt="image" src="https://github.com/user-attachments/assets/05009582-c13c-40de-bd1b-ec2d892ddbbc" />
<img width="1033" height="861" alt="image" src="https://github.com/user-attachments/assets/8b72b5b8-61e0-46d8-9f9a-8c58b39f7724" />
<img width="1191" height="627" alt="image" src="https://github.com/user-attachments/assets/acafbb42-bef3-4e13-ba85-83f4f0fbf464" />
<img width="1183" height="822" alt="image" src="https://github.com/user-attachments/assets/e62d9a05-72c5-4bce-a648-7945093ca168" />
<img width="1187" height="436" alt="image" src="https://github.com/user-attachments/assets/d2ad0bab-6fb5-4b7b-8fc3-c0feddb64280" />
<img width="1187" height="438" alt="image" src="https://github.com/user-attachments/assets/db576d75-3954-470a-bc18-1a66cd11ac68" />

