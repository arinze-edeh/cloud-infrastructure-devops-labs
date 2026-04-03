# Azure Blob Storage Provisioning: Public Container Deployment via CLI

> Deploying a public-access Azure Blob Storage account and container using the Azure CLI as part of an infrastructure migration initiative for the Nautilus DevOps team.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Deployment Steps](#deployment-steps)
  - [Step 1: Verify Azure Account](#step-1-verify-azure-account)
  - [Step 2: Verify Resource Group](#step-2-verify-resource-group)
  - [Step 3: Create Storage Account](#step-3-create-storage-account)
  - [Step 4: Create Public Blob Container](#step-4-create-public-blob-container)
  - [Step 5: Verify Blob Container Access](#step-5-verify-blob-container-access)
- [Validation Checklist](#validation-checklist)
- [Key Services Used](#key-services-used)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks and Troubleshooting](#risks-and-troubleshooting)
- [Outcome](#outcome)

---

## Overview

As part of an ongoing infrastructure migration, the Nautilus DevOps team requires a centralized, publicly accessible Azure Blob Storage solution. This document details the end-to-end CLI-driven process for provisioning a new Azure Storage account (`devopsst26155`) and a publicly accessible Blob container (`devops-blob-23782`) with anonymous read access enabled for both containers and blobs.

This implementation consolidates storage operations within the Azure environment and prepares the foundation for data hosting, migration staging, and public asset delivery.

| Attribute | Value |
|---|---|
| **Storage Account** | `devopsst26155` |
| **Blob Container** | `devops-blob-23782` |
| **Access Level** | Public (anonymous read for blobs and containers) |
| **Azure Region** | East US |
| **Resource Group** | `kml_rg_main-a721d14a88d344a0` |
| **SKU** | Standard_LRS |
| **Kind** | StorageV2 |

---

## Problem Statement

The Nautilus DevOps team requires a scalable, publicly accessible object storage solution in Azure to support data migration and infrastructure consolidation. The specific requirements are:

- A new Azure Storage Account provisioned in the East US region with locally redundant storage.
- A Blob container configured with **anonymous read access** at the container level, enabling public listing and blob retrieval without authentication.
- All resources provisioned via the Azure CLI for repeatability and auditability, avoiding portal-based manual configurations.

---

## Architecture Summary

```
Azure Subscription (Azure Free Labs)
  └── Resource Group: kml_rg_main-a721d14a88d344a0
        └── Storage Account: devopsst26155 (StorageV2, Standard_LRS, East US)
              └── Blob Container: devops-blob-23782
                    └── Public Access: container (anonymous read for blobs and container)
```

---

## Prerequisites

- **Azure CLI** installed and authenticated (`az login` or service principal configured).
- **Contributor** or **Storage Account Contributor** RBAC role on the target subscription or resource group.
- The target resource group (`kml_rg_main-a721d14a88d344a0`) must already exist. Do not attempt to create it; it is pre-provisioned by the environment.
- Familiarity with Azure Storage concepts: accounts, containers, access tiers, and public access levels.

---

## Deployment Steps

### Step 1: Verify Azure Account

Before provisioning any resources, confirm that the Azure CLI session is authenticated and targeting the correct subscription.

```bash
az account show
```

**Why this matters:** Running resource commands against the wrong subscription is a common and costly mistake. This step provides explicit confirmation of the active tenant, subscription ID, subscription name, and authentication type before any changes are made.

**Expected output:** The active subscription details are returned, including `environmentName`, `id`, `name`, `state`, and the authenticated user or service principal.

> Screenshot: Active subscription confirmed as "Azure Free Labs" authenticated via service principal.

<img width="1031" height="536" alt="az account show output confirming active subscription" src="https://github.com/user-attachments/assets/d53f8334-1084-4c01-bc15-395cb875227e" />

---

### Step 2: Verify Resource Group

Confirm that the target resource group exists in the subscription before attempting to deploy resources into it.

```bash
az group list --query "[].name" -o tsv
```

**Why this matters:** Attempting to create a storage account in a non-existent resource group will result in a `ResourceGroupNotFound` error. This step validates the environment state before proceeding.

**Expected output:** The resource group `kml_rg_main-a721d14a88d344a0` appears in the listed results.

> Screenshot: Resource group `kml_rg_main-a721d14a88d344a0` confirmed present in the subscription.

<img width="1029" height="656" alt="az group list output confirming target resource group exists" src="https://github.com/user-attachments/assets/6bf54a6e-d0a7-42f6-93f6-5aa878632cda" />

---

### Step 3: Create Storage Account

Provision the Azure Storage Account with the required configuration: StorageV2 kind, Standard LRS redundancy, East US region, and public blob access enabled.

```bash
az storage account create \
  --name devopsst26155 \
  --resource-group kml_rg_main-a721d14a88d344a0 \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access true
```

**Parameter rationale:**

- `--sku Standard_LRS`: Locally Redundant Storage keeps three copies within a single region. Chosen for cost efficiency in this migration context; upgrade to GRS or ZRS for production workloads requiring higher durability.
- `--kind StorageV2`: General-purpose v2 is the recommended and most current storage account type, supporting all storage services (Blob, File, Queue, Table) and the latest features including tiered access.
- `--allow-blob-public-access true`: A prerequisite for setting container-level public access in Step 4. Without this flag, container-level anonymous access cannot be enabled.
- `--location eastus`: Placement in East US reduces latency for the target workloads and aligns with the team's regional policy.

**Expected output:** The full storage account JSON object is returned with `provisioningState: Succeeded`, confirming successful creation. Key fields to verify: `name`, `location`, `kind`, `sku.name`, `allowBlobPublicAccess`, and `primaryEndpoints`.

> Screenshot 1: Storage account creation command and initial JSON response.

<img width="1032" height="858" alt="Storage account creation output showing initial properties" src="https://github.com/user-attachments/assets/c5ee1461-0399-440a-bfe0-7a684efbe722" />

> Screenshot 2: Continuation of storage account JSON output showing resource ID, SKU, and key creation timestamps.

<img width="1031" height="864" alt="Storage account output continuation showing resource ID and SKU details" src="https://github.com/user-attachments/assets/68bfe97e-f8ad-4916-a2b3-a8d4252e2902" />

> Screenshot 3: Final portion of storage account output confirming primary endpoints, provisioning state, and resource group assignment.

<img width="1039" height="629" alt="Storage account output showing primary endpoints and provisioningState Succeeded" src="https://github.com/user-attachments/assets/9909116d-edaf-4152-a63b-7948eba7dc8c" />

---

### Step 4: Create Public Blob Container

Create the Blob container within the storage account with anonymous read access set at the container level.

```bash
az storage container create \
  --name devops-blob-23782 \
  --account-name devopsst26155 \
  --public-access container
```

**Public access levels explained:**

| Level | Behavior |
|---|---|
| `off` | No anonymous access. Authentication required for all operations. |
| `blob` | Anonymous read access for individual blobs only; container listing is restricted. |
| `container` | Anonymous read access for both container listing and individual blobs. **This is the configured level.** |

`--public-access container` is selected because the migration requirement specifies anonymous read access for both the container and its blobs, enabling unauthenticated clients to list and retrieve objects.

**Note on credential warnings:** The CLI outputs an advisory recommending the use of `--connection-string`, `--account-key`, or `--sas-token`, or `--auth-mode login`. In this environment, the CLI falls back to querying the account key automatically. In production, always prefer `--auth-mode login` with RBAC-assigned roles to avoid key-based authentication.

**Expected output:** `{ "created": true }` confirms the container was created successfully.

> Screenshot: Blob container created with `--public-access container`. The `{ "created": true }` response confirms success.

<img width="1036" height="551" alt="Blob container creation output showing created true" src="https://github.com/user-attachments/assets/1ec934ba-cf56-4c8a-a915-6a9e78645af5" />

---

### Step 5: Verify Blob Container Access

Validate that the container was created with the correct name and the intended public access level before concluding the deployment.

```bash
az storage container show \
  --name devops-blob-23782 \
  --account-name devopsst26155 \
  --query "{Name:name, PublicAccess:properties.publicAccess}"
```

**Why this matters:** Verifying the access configuration independently of the creation command catches any misconfiguration before the storage is put into use. It also serves as an auditable confirmation artifact for handoff documentation.

**Expected output:**

```json
{
  "Name": "devops-blob-23782",
  "PublicAccess": "container"
}
```

> Screenshot: Container verification query confirming `Name: devops-blob-23782` and `PublicAccess: container` as configured.

<img width="1030" height="422" alt="Container show output confirming name and public access level" src="https://github.com/user-attachments/assets/fb01237c-21a7-4569-b781-18b7cbbfc07c" />

---

## Validation Checklist

- Azure CLI session verified against the correct subscription (`Azure Free Labs`).
- Target resource group `kml_rg_main-a721d14a88d344a0` confirmed present.
- Storage account `devopsst26155` created with `provisioningState: Succeeded`.
- Storage account configured with `Standard_LRS`, `StorageV2`, and `allowBlobPublicAccess: true`.
- Blob container `devops-blob-23782` created with `{ "created": true }`.
- Container public access level verified as `container` via `az storage container show`.

---

## Key Services Used

| Service | Purpose |
|---|---|
| **Azure Storage Account (StorageV2)** | Scalable, general-purpose object storage backbone for all blob operations. |
| **Azure Blob Storage** | Object storage layer hosting the public container for anonymous data access. |
| **Azure CLI** | Imperative tooling used for repeatable, auditable resource provisioning without portal dependencies. |

---

## Key Decisions

**StorageV2 over BlobStorage kind:** StorageV2 is the current recommended account type and supports all storage tiers (Hot, Cool, Archive) as well as all storage services. BlobStorage kind is a legacy option restricted to Blob operations only.

**Standard_LRS over higher redundancy tiers:** LRS was selected based on cost constraints in this environment. For workloads requiring cross-zone or cross-region durability, ZRS or GRS should be evaluated.

**Container-level public access over blob-level:** The task requirements explicitly specify anonymous read access for both container listing and individual blobs. Blob-level access would satisfy blob retrieval but would not allow clients to enumerate container contents.

**`--allow-blob-public-access true` at account level:** Azure requires this account-level flag to be explicitly set before any container can be configured for public access. Omitting this flag causes container-level public access settings to be silently overridden.

---

## Best Practices and Operational Considerations

- **Prefer `--auth-mode login` in production:** Rather than relying on account key fallback, assign the `Storage Blob Data Contributor` RBAC role to the authenticated identity and use `--auth-mode login` in all CLI commands. This eliminates key-based access and enforces least-privilege.
- **Enable soft delete for blobs:** For production containers, enable blob soft delete (`az storage account blob-service-properties update --enable-delete-retention true`) to protect against accidental deletion during migration operations.
- **Use lifecycle management policies:** For long-running storage accounts, configure tiering policies to automatically move infrequently accessed blobs to Cool or Archive tiers to reduce storage costs.
- **Audit public access decisions:** Public containers expose all blobs to the internet without authentication. Confirm this is intentional and time-bound. Restrict access as soon as the migration phase requiring public access is complete.
- **Tag resources from creation:** Add resource tags (`--tags project=migration env=staging`) at creation time to support cost attribution and governance. Retrofitting tags is operationally expensive at scale.

---

## Risks and Troubleshooting

**Risk: Deploying into the wrong subscription**
Verify with `az account show` before every provisioning session, especially in environments with multiple subscriptions. Set the target explicitly with `az account set --subscription <id>` if needed.

**Risk: Public blob containers exposing sensitive data**
Public access at the `container` level means any blob URL is globally accessible. Ensure no confidential data is uploaded to this container while public access is active. Rotate to `blob` level or `off` immediately after the migration window.

**Issue: `ResourceGroupNotFound` error**
The resource group must be pre-existing. Do not attempt to create it; it is environment-managed. Rerun `az group list` to confirm the exact name, including any trailing identifiers.

**Issue: `AllowBlobPublicAccess` property blocking container creation**
If the storage account was created without `--allow-blob-public-access true`, the container create command with `--public-access container` will succeed but the access level will be silently downgraded to `off`. Always verify with `az storage container show` after creation.

**Issue: Credential warning on container commands**
The CLI advisory about missing credentials (`--account-key`, `--connection-string`, `--sas-token`) is informational and does not block execution in environments where the CLI can resolve account keys via RBAC. Use `--auth-mode login` to suppress this warning and enforce RBAC-based authentication.

---

## Outcome

The Azure Storage Account `devopsst26155` and public Blob container `devops-blob-23782` were provisioned successfully via the Azure CLI with all configuration verified against expected values. The container is accessible with anonymous read access at the container level, satisfying the Nautilus DevOps team's data migration requirements. All steps were executed without portal intervention, producing a fully auditable, CLI-driven deployment record.
