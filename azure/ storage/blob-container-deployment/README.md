# Azure Blob Storage Provisioning: Private Container Deployment for Secure Data Migration

> **Domain:** Cloud Storage | Azure CLI | Infrastructure Provisioning
> **Environment:** Azure Free Labs | East US Region
> **Scope:** Storage account creation, private blob container provisioning, and CLI-driven validation

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Deployment Steps](#deployment-steps)
  - [Step 1: Verify Azure Account](#step-1-verify-azure-account)
  - [Step 2: List Resource Groups](#step-2-list-resource-groups)
  - [Step 3: Create Storage Account](#step-3-create-storage-account)
  - [Step 4: Create Private Blob Container](#step-4-create-private-blob-container)
  - [Step 5: Verify Blob Container](#step-5-verify-blob-container)
- [Result](#result)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Lessons Learned](#lessons-learned)

---

## Overview

As part of an ongoing data migration initiative, the Nautilus DevOps team required a secure, centralized storage target within Azure to consolidate operational data. This document covers the end-to-end provisioning of an Azure Storage Account and a private Blob container using the Azure CLI, from environment validation through deployment verification.

**Resources Provisioned:**

- **Storage Account:** `devopsst1752`
- **Blob Container:** `devops-blob-25744`
- **Resource Group:** `kml_rg_main-ef2d884406fd4cbf`
- **Region:** East US

---

## Problem Statement

Data migration operations require a dedicated, isolated, and private storage target to prevent unintended public exposure of sensitive operational data. Provisioning storage through the Azure CLI ensures repeatability, auditability, and alignment with infrastructure-as-code principles, as opposed to manual portal-based configurations that introduce human error and lack traceability.

---

## Architecture Summary

```
Azure Subscription (Azure Free Labs)
└── Resource Group: kml_rg_main-ef2d884406fd4cbf (East US)
    └── Storage Account: devopsst1752
        ├── SKU: Standard_LRS
        ├── Kind: StorageV2
        └── Blob Container: devops-blob-25744
            └── Public Access: OFF (Private)
```

---

## Prerequisites

- Azure CLI installed and authenticated on the target host
- Service principal with sufficient RBAC permissions to create storage resources
- Target resource group pre-existing in the subscription (resource group creation not permitted in this environment)
- Confirmed subscription in `Enabled` state prior to execution

---

## Deployment Steps

### Step 1: Verify Azure Account

Before provisioning any resources, confirm the active subscription context and authentication state. This prevents resource creation in the wrong tenant or subscription.

```bash
az account show
```

**Expected output** confirms the subscription name, tenant ID, subscription ID, and the authenticated user type. In this deployment the user type is `servicePrincipal`, consistent with CI/CD and automated provisioning workflows.

> **Operational Note:** Always verify account context before provisioning. Accidental deployments to non-target subscriptions are a common cause of billing anomalies and resource sprawl.

![Step 1: az account show output confirming active subscription, tenant ID, and service principal authentication](https://github.com/user-attachments/assets/78a71991-3aa1-4dc1-bc39-d9f9daefd987)

*The output confirms subscription `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` under the Azure Free Labs environment, authenticated as a service principal.*

---

### Step 2: List Resource Groups

Before creating any storage resource, identify the pre-existing resource group to use as the deployment target. In constrained lab and enterprise environments, resource groups are often pre-provisioned and managed by platform teams.

```bash
az group list --output table
```

**Expected output** returns the resource group name, location, and provisioning state in tabular format for quick human-readable validation.

![Step 2: az group list output showing available resource groups in the subscription](https://github.com/user-attachments/assets/39880a81-5550-4350-a3ae-5f21c755ce2e)

*The resource group `kml_rg_main-ef2d884406fd4cbf` is confirmed as `Succeeded` in `eastus`, making it a valid target for storage account provisioning.*

---

### Step 3: Create Storage Account

Provision the Azure Storage Account with a Standard LRS SKU and StorageV2 kind. StorageV2 is the recommended account type for most production workloads due to its support for the latest features including blob lifecycle management, static website hosting, and hierarchical namespace (when needed).

```bash
az storage account create \
  --name devopsst1752 \
  --resource-group kml_rg_main-ef2d884406fd4cbf \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

**Parameter Rationale:**

| Parameter | Value | Reason |
|---|---|---|
| `--name` | `devopsst1752` | Globally unique, lowercase, alphanumeric |
| `--resource-group` | `kml_rg_main-ef2d884406fd4cbf` | Pre-provisioned target group |
| `--location` | `eastus` | Co-located with resource group |
| `--sku` | `Standard_LRS` | Cost-effective, single-region redundancy |
| `--kind` | `StorageV2` | Recommended for all new storage workloads |

**Expected output** is a full JSON object describing the newly created storage account, including primary endpoints, encryption settings, SKU, provisioning state, and network rule set.

![Step 3a: Storage account creation command output (part 1) showing account configuration and encryption settings](https://github.com/user-attachments/assets/1083ca38-86cf-4842-bbe0-43f9f0644a77)

*The creation response confirms HTTPS-only traffic (`enableHttpsTrafficOnly: true`), service-managed encryption for both blob and file services, and a `Hot` default access tier.*

![Step 3b: Storage account creation command output (part 2) showing endpoints, SKU, and provisioning state](https://github.com/user-attachments/assets/da411152-ee9e-4c04-ae1b-b9283bd4dcc1)

*Primary endpoints are assigned across blob, DFS, file, queue, table, and web services. `provisioningState: Succeeded` confirms successful deployment. The network rule set defaults to `Allow` with `AzureServices` bypass enabled.*

---

### Step 4: Create Private Blob Container

With the storage account provisioned, create a private Blob container within it. Setting `--public-access off` ensures that no anonymous or unauthenticated access is permitted at the container level, a critical security requirement for migration data.

```bash
az storage container create \
  --name devops-blob-25744 \
  --account-name devopsst1752 \
  --public-access off
```

> **Security Note:** The CLI advisory recommends supplying `--auth-mode login`, `--account-key`, or `--connection-string` for authenticated access. In this deployment, the environment falls back to account key resolution automatically. In production, enforce RBAC-based access using `--auth-mode login` and assign the `Storage Blob Data Contributor` role to the executing identity.

**Expected output:**

```json
{
  "created": true
}
```

![Step 4: Blob container creation output showing created: true, with credential advisory visible](https://github.com/user-attachments/assets/f58d3962-5185-426c-8aa2-3a397bbae847)

*The `"created": true` response confirms successful container provisioning. The advisory message regarding credential methods is informational and does not indicate an error.*

---

### Step 5: Verify Blob Container

Validate the deployed container by querying its metadata directly. This step confirms the container name, lease status, and creation timestamp, providing an auditable verification record.

```bash
az storage container show \
  --name devops-blob-25744 \
  --account-name devopsst1752 \
  --output table
```

**Expected output** returns a tabular view of the container showing its name, lease status (`unlocked`), and last modified timestamp.

![Step 5: az storage container show output confirming container name, lease status, and last modified timestamp](https://github.com/user-attachments/assets/999bc0c9-9436-43bf-90aa-58e7886c92e5)

*Container `devops-blob-25744` is confirmed as `unlocked` with a last modified timestamp of `2026-02-23T04:51:35+00:00`, indicating it is active and ready for data ingestion.*

---

## Result

| Resource | Value | Status |
|---|---|---|
| Storage Account | `devopsst1752` | Provisioned |
| Resource Group | `kml_rg_main-ef2d884406fd4cbf` | Confirmed |
| Region | `eastus` | Confirmed |
| SKU | `Standard_LRS` | Active |
| Blob Container | `devops-blob-25744` | Created |
| Public Access | `OFF` | Enforced |
| Provisioning State | `Succeeded` | Verified |

All resources were successfully deployed and verified using Azure CLI. The storage account and private blob container are operational and ready for use in the data migration workflow.

---

## Key Decisions

- **Standard_LRS over GRS:** Locally Redundant Storage was selected to minimize cost in a non-production migration context. For production workloads containing critical data, Geo-Redundant Storage (GRS) or Zone-Redundant Storage (ZRS) should be evaluated.
- **StorageV2 over BlobStorage kind:** StorageV2 supports all blob types (block, append, page) and offers access tier management at the account level, making it suitable for both current and future storage workload patterns.
- **Public access disabled at container creation:** Rather than relying on post-creation policy updates, access was locked at provisioning time to ensure no window of unintended exposure existed.
- **CLI provisioning over portal:** All resources were provisioned via Azure CLI to ensure reproducibility, logging, and compatibility with scripted or pipeline-based workflows.

---

## Best Practices and Operational Considerations

- **Use `--auth-mode login` in production.** Relying on account key fallback is acceptable in constrained environments but bypasses RBAC enforcement. Prefer Azure AD-based authorization with the `Storage Blob Data Contributor` role assigned to the executing identity.
- **Enable soft delete on blob containers.** For migration workloads, accidental deletions can have significant impact. Enable soft delete via `az storage blob service-properties delete-policy update` to allow recovery within a configurable retention window.
- **Apply resource tags at creation time.** Tags such as `environment`, `project`, `owner`, and `cost-center` should be applied using `--tags` during account creation for cost allocation and governance visibility.
- **Enforce HTTPS-only traffic.** The storage account defaults to `enableHttpsTrafficOnly: true`, which must not be disabled in any environment handling sensitive data.
- **Lock the storage account with a resource lock** in long-running environments to prevent accidental deletion: `az lock create --lock-type CanNotDelete`.
- **Review network rules.** The default `networkRuleSet.defaultAction: Allow` permits all network traffic. In production, restrict access to specific VNets or IP ranges using `az storage account update --default-action Deny`.

---

## Risks, Edge Cases, and Troubleshooting

| Scenario | Risk | Mitigation |
|---|---|---|
| Storage account name collision | Names must be globally unique across all Azure subscriptions | Use a naming convention incorporating environment, team, and a random suffix |
| Missing RBAC roles | `az storage container create` may fail with 403 if the service principal lacks `Storage Blob Data Contributor` | Assign the role at the storage account scope before execution |
| Account key fallback in CI/CD | Credential advisory appears when no explicit auth method is passed | Always pass `--auth-mode login` or inject account keys via environment variables in pipelines |
| Container left in unlocked state with no data | Unlocked containers with no immutability policies are at risk of premature deletion | Apply lifecycle management policies or immutability policies where required |
| Region mismatch | Creating the storage account in a different region than the resource group can introduce latency and cross-region egress charges | Always confirm the target location matches the resource group region |

---

## Lessons Learned

- **Validate account context first.** Confirming the active subscription and service principal identity before any provisioning step eliminates an entire class of misdirected resource creation errors.
- **The credential advisory is not an error.** When no explicit auth method is provided to storage container commands, Azure CLI emits an advisory and falls back to account key resolution. This is expected behavior and does not indicate command failure.
- **`"created": true` is the correct success indicator** for `az storage container create`. Unlike resource creation commands that return full JSON objects, container creation returns a minimal confirmation object.
- **StorageV2 is the correct default** for all new storage accounts. The legacy `BlobStorage` kind is limited and should not be used for new workloads.
- **Verification is not optional.** Running `az storage container show` after creation provides an auditable record of the deployed state and confirms that the container is active and accessible before handing off to the next pipeline stage.




























# Azure Blob Storage Deployment - DevOps Project

## Project Overview

- As part of the ongoing data migration process, the Nautilus DevOps team consolidated storage into Azure by creating private Blob containers. This project documents the deployment of:

- Storage Account: `devopsst1752`

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
<img width="1024" height="543" alt="image" src="https://github.com/user-attachments/assets/39880a81-5550-4350-a3ae-5f21c755ce2e" />

## Step 3: Create Storage Account
- Create a new Storage Account named `devopsst1752`

- `az storage account create \`
 - `--name devopsst1752 \`
 - `--resource-group kml_rg_main-ef2d884406fd4cbf \`
 - `--location eastus \`
 - `--sku Standard_LRS \`
 - `--kind StorageV2`

Screenshots:
<img width="1036" height="864" alt="image" src="https://github.com/user-attachments/assets/1083ca38-86cf-4842-bbe0-43f9f0644a77" />
<img width="1019" height="862" alt="image" src="https://github.com/user-attachments/assets/da411152-ee9e-4c04-ae1b-b9283bd4dcc1" />

## Step 4. Create Private Blob Container

- Create a private Blob container
- `az storage container create \`
 - `--name devops-blob-25744 \`
 - `--account-name devopsst1752 \`
 - `--public-access off`

Screenshot:
<img width="1028" height="423" alt="image" src="https://github.com/user-attachments/assets/f58d3962-5185-426c-8aa2-3a397bbae847" />

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
