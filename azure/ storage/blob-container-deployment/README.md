# Azure Blob Storage Deployment – DevOps Project

## Project Overview

- As part of the ongoing data migration process, the Nautilus DevOps team consolidated storage into Azure by creating private Blob containers. This project documents the deployment of:

- Storage Account: devopsst1752

- Private Blob Container: `devops-blob-25744`

The deployment ensures secure, private storage of migration data within the Azure environment.

## Key Objectives:

- Provision a new Azure storage account

- Create a private Blob container for storing sensitive data

- Validate deployment using Azure CLI commands

## Prerequisites

- Azure CLI installed and configured on your azure-client host

- Access to Azure portal with credentials

## Deployment Steps

## Step 1: Verify Azure Account
- Show current Azure account details
- `az account show`

Screenshot:
<img width="1035" height="514" alt="image" src="https://github.com/user-attachments/assets/78a71991-3aa1-4dc1-bc39-d9f9daefd987" />

## Step 2: List Resource Groups

- Verify available resource groups
- `az group list --output table`

Screenshot:


## Step 3: Create Storage Account
- Create a new Storage Account named `devopsst1752`

- `az storage account create \`
 - `--name devopsst1752 \`
 - `--resource-group kml_rg_main-ef2d884406fd4cbf \`
 - `--location eastus \`
 - `--sku Standard_LRS \`
 - `--kind StorageV2`

Screenshot:

4. Create Private Blob Container

- Create a private Blob container
- `az storage container create \`
 - `--name devops-blob-25744 \`
 - `--account-name devopsst1752 \`
 - `--public-access off`

Screenshot:


## Step 5: Verify Blob Container

- Verify container creation

- `az storage container show \`
 - `--name devops-blob-25744 \`
 - `--account-name devopsst1752 \`
 - `--output table`

Screenshot:
<img width="1034" height="577" alt="image" src="https://github.com/user-attachments/assets/999bc0c9-9436-43bf-90aa-58e7886c92e5" />

## Result

- Storage Account devopsst1752 successfully created in resource group `kml_rg_main-ef2d884406fd4cbf`.

- Private Blob Container `devops-blob-25744` successfully provisioned.

- Deployment verified using Azure CLI.




<img width="1024" height="543" alt="image" src="https://github.com/user-attachments/assets/39880a81-5550-4350-a3ae-5f21c755ce2e" />
<img width="1036" height="864" alt="image" src="https://github.com/user-attachments/assets/1083ca38-86cf-4842-bbe0-43f9f0644a77" />
<img width="1019" height="862" alt="image" src="https://github.com/user-attachments/assets/da411152-ee9e-4c04-ae1b-b9283bd4dcc1" />
<img width="1028" height="423" alt="image" src="https://github.com/user-attachments/assets/f58d3962-5185-426c-8aa2-3a397bbae847" />


