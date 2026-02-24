# Azure Blob Storage Deployment – Nautilus DevOps

- This repository documents the creation of a public Azure Blob container for the Nautilus DevOps data migration project. The goal is to centralize storage within Azure and provide public access to selected data.

## Project Overview

- Objective: `Create a storage account and public blob container in Azure.`

- Storage Account Name: `devopsst26155`

- Blob Container Name: `devops-blob-23782`

- Access Level: `Public (anonymous read access for blobs and containers)`

- Azure Region: `East US`

- Resource Group: `kml_rg_main-a721d14a88d344a0`

## Prerequisites

- Azure CLI installed and configured.

- Access to Azure with Contributor or Storage Account Contributor role.

- Knowledge of the target resource group and naming conventions.

## Deployment Steps

## Step 1: Verify Azure Account
- `az account show`

Expected Outcome:
- `Displays the active subscription and tenant info to confirm CLI readiness.`

Screenshot:
<img width="1031" height="536" alt="image" src="https://github.com/user-attachments/assets/d53f8334-1084-4c01-bc15-395cb875227e" />

## Step 2: Verify Resource Group
- `az group list --query "[].name" -o tsv`

Expected Outcome:
- Lists available resource groups to confirm the target group exists  `(kml_rg_main-a721d14a88d344a0)`.

Screenshot:
<img width="1029" height="656" alt="image" src="https://github.com/user-attachments/assets/6bf54a6e-d0a7-42f6-93f6-5aa878632cda" />

## Step 3: Create Storage Account
- `az storage account create \`
 - `--name devopsst26155 \`
 - `--resource-group kml_rg_main-a721d14a88d344a0 \`
 - `--location eastus \`
 - `--sku Standard_LRS \`
 - `--kind StorageV2 \`
 - `--allow-blob-public-access true`

Expected Outcome:
- Storage account is created with public blob access enabled.

Screenshots:
<img width="1032" height="858" alt="image" src="https://github.com/user-attachments/assets/c5ee1461-0399-440a-bfe0-7a684efbe722" />
<img width="1031" height="864" alt="image" src="https://github.com/user-attachments/assets/68bfe97e-f8ad-4916-a2b3-a8d4252e2902" />
<img width="1039" height="629" alt="image" src="https://github.com/user-attachments/assets/9909116d-edaf-4152-a63b-7948eba7dc8c" />

## Step 4: Create Public Blob Container
az storage container create \
  --name devops-blob-23782 \
  --account-name devopsst26155 \
  --public-access container

Expected Outcome:
Blob container created with anonymous read access.

Screenshot:
<img width="1036" height="551" alt="image" src="https://github.com/user-attachments/assets/1ec934ba-cf56-4c8a-a915-6a9e78645af5" />

## Step 5: Verify Blob Container Access
az storage container show \
  --name devops-blob-23782 \
  --account-name devopsst26155 \
  --query "{Name:name, PublicAccess:properties.publicAccess}"

Expected Output:

{
  "Name": "devops-blob-23782",
  "PublicAccess": "container"
}

Screenshot:

<img width="1030" height="422" alt="image" src="https://github.com/user-attachments/assets/fb01237c-21a7-4569-b781-18b7cbbfc07c" />

## Validation Checklist

- Azure account verified and active subscription confirmed.

- Target resource group exists.

- Storage account `devopsst26155` created successfully.

- Public blob container `devops-blob-23782` created successfully.

- Anonymous read access for blobs and containers verified.

## Key Services Used

- Azure Storage Account (StorageV2) - scalable blob storage.

- Azure Blob Storage - public container for data hosting.

- Azure CLI - command-line tool for resource creation and management.

## Outcome

- Result: Storage account and public blob container deployed successfully.

