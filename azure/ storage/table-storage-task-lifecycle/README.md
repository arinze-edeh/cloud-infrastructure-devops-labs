# Azure Table Storage To-Do Task Manager

> **Enterprise-grade task management backend provisioned on Azure Table Storage using the Azure CLI**
> Developed for the Nautilus DevOps Platform | Region: `eastus` | Environment: `AzureCloud`

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Verify Azure CLI and Account Context](#step-1-verify-azure-cli-and-account-context)
  - [Step 2: Identify the Target Resource Group](#step-2-identify-the-target-resource-group)
  - [Step 3: Provision the Azure Storage Account](#step-3-provision-the-azure-storage-account)
  - [Step 4: Retrieve the Storage Account Key](#step-4-retrieve-the-storage-account-key)
  - [Step 5: Create the Table Storage Table](#step-5-create-the-table-storage-table)
  - [Step 6: Insert Task Entities](#step-6-insert-task-entities)
  - [Step 7: Verify Individual Entities](#step-7-verify-individual-entities)
  - [Step 8: Query All Entities in the Table](#step-8-query-all-entities-in-the-table)
- [Data Schema](#data-schema)
- [Verification and Validation](#verification-and-validation)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Problem Statement

The Nautilus DevOps team required a lightweight, scalable, and cost-efficient backend to store and manage task data for a To-Do application. The key requirements were:

- **Unique task identification** via a composite key strategy (`PartitionKey` + `RowKey`)
- **Descriptive task metadata** including a `description` field and a `status` field (e.g., `completed`, `in-progress`)
- **Minimal operational overhead** with no need for a relational database engine
- **CLI-driven provisioning** to enable repeatable, scriptable, and audit-friendly infrastructure creation

**Root Cause of Complexity:** Traditional relational databases (SQL Server, PostgreSQL) introduce schema overhead, connection management complexity, and higher cost for simple key-value or tabular workloads. Azure Table Storage solves this with a NoSQL, schema-less, serverless-friendly model billed purely on storage consumption and transaction count.

---

## Solution Architecture

```
+-----------------------------+
|      Azure CLI (az)         |
|  (Provisioning Plane)       |
+-------------+---------------+
              |
              v
+-----------------------------+
|  Azure Storage Account      |
|  Name: devopstablest197     |
|  SKU:  Standard_LRS         |
|  Kind: StorageV2            |
|  Region: eastus             |
+-------------+---------------+
              |
              v
+-----------------------------+
|  Azure Table Storage        |
|  Table Name: tasks          |
+-------------+---------------+
              |
     +--------+--------+
     |                 |
+----+----+       +----+----+
| RowKey=1 |     | RowKey=2 |
| status:  |     | status:  |
| completed|     |in-progress|
+----------+     +----------+
```

**Storage Account Properties:**

| Property | Value |
|---|---|
| Account Name | `devopstablest197` |
| Resource Group | `kml_rg_main-4132cedd91b749aa` |
| Location | `eastus` |
| SKU | `Standard_LRS` |
| Kind | `StorageV2` |
| Access Tier | `Hot` |
| HTTPS Only | `true` |
| Public Blob Access | `false` |

---

## Prerequisites

Ensure the following are in place before executing this guide:

- [ ] Azure CLI version `2.67.0` or later installed
- [ ] Active Azure subscription with sufficient permissions (`Contributor` or `Owner` role)
- [ ] An existing Resource Group in the `eastus` region
- [ ] Authenticated CLI session via Service Principal or interactive login
- [ ] Bash shell environment (Linux, macOS, or WSL on Windows)

**Verify CLI installation:**

```bash
az --version
```

*Expected output: CLI version, core version, Python runtime details, and dependency versions.*

---

## Project Structure

```
azure-table-storage-todo/
|
|-- README.md                   # This documentation
|-- scripts/
|   |-- provision.sh            # Full end-to-end provisioning script
|   |-- insert-entities.sh      # Entity insertion script
|   |-- query-entities.sh       # Entity query and verification script
|-- .env.example                # Environment variable template (never commit secrets)
```

---

## Step-by-Step Implementation

### Step 1: Verify Azure CLI and Account Context

Before provisioning any resource, confirm the CLI version and validate that the correct subscription and tenant are active. This prevents accidental resource creation in the wrong environment.

```bash
az --version
```

**Expected Output:**

```
azure-cli                         2.67.0
core                              2.67.0
telemetry                          1.1.0
```

Next, confirm your active account context:

```bash
az account show
```

**Expected Output (key fields):**

```json
{
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "name": "Azure Free Labs",
  "state": "Enabled",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552"
}
```

> **Why this matters:** Running `az account show` before any deployment is a non-negotiable first step in multi-subscription environments. Deploying into the wrong subscription is a common and costly mistake.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-01-az-version-and-account-show.png]`
> *Caption: Terminal output showing Azure CLI version 2.67.0 and active account context for the Azure Free Labs subscription.*

---

### Step 2: Identify the Target Resource Group

List all resource groups in the active subscription to confirm the target group exists and is healthy before attaching new resources to it.

```bash
az group list --output table
```

**Expected Output:**

```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-4132cedd91b749aa  eastus      Succeeded
```

> **Why this matters:** Confirming the resource group status is `Succeeded` ensures no underlying Azure Resource Manager issues exist that could cause downstream provisioning failures.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-02-resource-group-list.png]`
> *Caption: Azure CLI listing the resource group `kml_rg_main-4132cedd91b749aa` in `eastus` with status `Succeeded`.*

---

### Step 3: Provision the Azure Storage Account

Create the storage account that will host Table Storage. Use `Standard_LRS` (Locally Redundant Storage) for cost efficiency in a development/lab environment. `StorageV2` is the current recommended kind as it supports all storage services including Tables, Blobs, Queues, and Files.

```bash
az storage account create \
  --name devopstablest197 \
  --resource-group kml_rg_main-4132cedd91b749aa \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

**Key flags:**

| Flag | Value | Purpose |
|---|---|---|
| `--name` | `devopstablest197` | Globally unique storage account name |
| `--resource-group` | `kml_rg_main-4132cedd91b749aa` | Target resource group |
| `--location` | `eastus` | Azure region for data residency |
| `--sku` | `Standard_LRS` | Locally redundant, cost-optimized replication |
| `--kind` | `StorageV2` | Modern general-purpose v2 account |

**Expected Output (key fields):**

```json
{
  "name": "devopstablest197",
  "location": "eastus",
  "kind": "StorageV2",
  "provisioningState": "Succeeded",
  "enableHttpsTrafficOnly": true,
  "allowBlobPublicAccess": false,
  "primaryEndpoints": {
    "table": "https://devopstablest197.table.core.windows.net/"
  }
}
```

> **Security Note:** Observe that `allowBlobPublicAccess` is set to `false` and `enableHttpsTrafficOnly` is `true` by default. These are secure defaults and must never be overridden without a documented business justification.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-03-storage-account-create.png]`
> *Caption: Full JSON output from `az storage account create` confirming `provisioningState: Succeeded` for `devopstablest197`.*

---

### Step 4: Retrieve the Storage Account Key

Extract the primary access key and store it in a shell variable for use in subsequent CLI operations. This avoids repeated API calls and keeps commands clean.

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name devopstablest197 \
  --resource-group kml_rg_main-4132cedd91b749aa \
  --query "[0].value" \
  --output tsv)
```

Confirm the key was captured:

```bash
echo $STORAGE_KEY
```

**Expected Output:**

```
2xsJECAl5SA6FXUdadeYL1JWySgnhYd8aUB1+pEoEMxvkhZvBAj1EWf7HCnSYN0Stk465N8dgpes+AStreGC3Q==
```

> **Security Warning:** Storage account keys grant full, unrestricted access to all data in the account. In production, use **Azure Managed Identities** or **Azure Active Directory (Entra ID)** authentication with RBAC instead of Shared Key authentication. Never commit keys to version control. Use `.env` files listed in `.gitignore` or Azure Key Vault for secret management.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-04-storage-key-retrieval.png]`
> *Caption: Terminal showing the storage account key successfully extracted and echoed as a shell variable.*

---

### Step 5: Create the Table Storage Table

Create a table named `tasks` inside the storage account. Azure Table Storage tables are schema-less; no column definitions are required at creation time.

```bash
az storage table create \
  --name tasks \
  --account-name devopstablest197 \
  --account-key $STORAGE_KEY
```

**Expected Output:**

```json
{
  "created": true
}
```

> **Key concept:** A `true` response in the `created` field confirms the table did not previously exist and was created fresh. If the table already existed, the response would return `false` without error, making this command idempotent and safe for use in automation pipelines.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-05-table-create.png]`
> *Caption: Azure CLI confirming table `tasks` was successfully created with `"created": true`.*

---

### Step 6: Insert Task Entities

Insert two task entities into the `tasks` table. Each entity is uniquely identified by the combination of `PartitionKey` and `RowKey`.

**Insert Task 1: Learn Table Storage (status: completed)**

```bash
az storage entity insert \
  --account-name devopstablest197 \
  --account-key $STORAGE_KEY \
  --table-name tasks \
  --entity \
    PartitionKey=tasks \
    RowKey=1 \
    description="Learn Table Storage" \
    status=completed
```

**Expected Output:**

```json
{
  "date": "2026-03-21T05:29:46+00:00",
  "etag": "W/\"datetime'2026-03-21T05%3A29%3A46.3190199Z'\"",
  "version": "2019-02-02"
}
```

**Insert Task 2: Build To-Do App (status: in-progress)**

```bash
az storage entity insert \
  --account-name devopstablest197 \
  --account-key $STORAGE_KEY \
  --table-name tasks \
  --entity \
    PartitionKey=tasks \
    RowKey=2 \
    description="Build To-Do App" \
    status=in-progress
```

**Expected Output:**

```json
{
  "date": "2026-03-21T05:30:05+00:00",
  "etag": "W/\"datetime'2026-03-21T05%3A30%3A06.2364897Z'\"",
  "version": "2019-02-02"
}
```

> **Design note:** The `PartitionKey=tasks` groups all tasks under a single logical partition. For large-scale applications, partition by user ID, project ID, or date range to maximize throughput and avoid partition hotspots.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-06-entity-insert-task1-task2.png]`
> *Caption: Both entity insert operations returning HTTP-equivalent success responses with `etag` and `date` fields.*

---

### Step 7: Verify Individual Entities

Retrieve each entity individually to confirm the data was persisted correctly with the expected field values.

**Verify Task 1:**

```bash
az storage entity show \
  --account-name devopstablest197 \
  --account-key $STORAGE_KEY \
  --table-name tasks \
  --partition-key tasks \
  --row-key 1
```

**Expected Output:**

```json
{
  "PartitionKey": "tasks",
  "RowKey": "1",
  "Timestamp": "2026-03-21T05:29:46.319019+00:00",
  "description": "Learn Table Storage",
  "status": "completed"
}
```

**Verify Task 2:**

```bash
az storage entity show \
  --account-name devopstablest197 \
  --account-key $STORAGE_KEY \
  --table-name tasks \
  --partition-key tasks \
  --row-key 2
```

**Expected Output:**

```json
{
  "PartitionKey": "tasks",
  "RowKey": "2",
  "Timestamp": "2026-03-21T05:30:06.236489+00:00",
  "description": "Build To-Do App",
  "status": "in-progress"
}
```

> **Validation pattern:** Always verify individual entity reads after bulk inserts. This is the equivalent of a read-after-write consistency check and is critical in eventual-consistency distributed storage systems.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-07-entity-show-task1-task2.png]`
> *Caption: Individual entity reads confirming `status: completed` for Task 1 and `status: in-progress` for Task 2.*

---

### Step 8: Query All Entities in the Table

Run a full table scan to confirm the complete data set is visible and correctly formatted as a unified result.

```bash
az storage entity query \
  --account-name devopstablest197 \
  --account-key $STORAGE_KEY \
  --table-name tasks \
  --output table
```

**Expected Output:**

```
PartitionKey    RowKey    Description          Status
--------------  --------  -------------------  -----------
tasks           1         Learn Table Storage  completed
tasks           2         Build To-Do App      in-progress
```

> **Performance note:** `az storage entity query` without a filter performs a full table scan. In production, always use `--filter` with OData expressions (e.g., `--filter "status eq 'in-progress'"`) to avoid unnecessary RU consumption and latency at scale.

**SCREENSHOT PLACEHOLDER**
> `[screenshot-08-entity-query-all.png]`
> *Caption: Full table query output displaying both task entities in a formatted table confirming correct data persistence.*

---

## Data Schema

Azure Table Storage entities in the `tasks` table follow this logical schema:

| Field | Type | Required | Description |
|---|---|---|---|
| `PartitionKey` | String | Yes | Logical grouping key. Value: `tasks` |
| `RowKey` | String | Yes | Unique entity identifier within partition |
| `Timestamp` | DateTime | Auto | System-managed last-modified timestamp |
| `description` | String | Yes | Human-readable task description |
| `status` | String | Yes | Task lifecycle state: `completed`, `in-progress`, `pending` |

---

## Verification and Validation

Use the following checklist to confirm a successful deployment:

- [ ] `az account show` returns the correct subscription ID and tenant
- [ ] Resource group `kml_rg_main-4132cedd91b749aa` has status `Succeeded`
- [ ] Storage account `devopstablest197` has `provisioningState: Succeeded`
- [ ] `STORAGE_KEY` variable is non-empty after extraction
- [ ] Table `tasks` was created with `"created": true`
- [ ] Task 1 (`RowKey=1`) shows `status: completed`
- [ ] Task 2 (`RowKey=2`) shows `status: in-progress`
- [ ] `az storage entity query` returns exactly 2 rows

---

## Best Practices

### Security

- **Never use Shared Key authentication in production.** Use Azure Managed Identities or Entra ID RBAC with the `Storage Table Data Contributor` role.
- **Enable soft delete** and resource locks on storage accounts holding critical data.
- **Rotate storage keys** on a defined schedule (recommended: every 90 days) and use Azure Key Vault for automated rotation.
- **Enable Azure Defender for Storage** to detect anomalous access patterns, malware uploads, and suspicious activity.
- Set `--min-tls-version TLS1_2` explicitly during storage account creation. The default `TLS1_0` observed in this lab is not acceptable in production.

### Naming Conventions

- Storage account names must be globally unique, 3 to 24 characters, lowercase alphanumeric only. Use a consistent naming pattern: `{project}{environment}{service}{random}` e.g., `nautilusprodtable4732`.
- Table names should be descriptive and domain-aligned: `tasks`, `users`, `auditlogs`.

### Operational Excellence

- **Tag all resources** with `Environment`, `Owner`, `CostCenter`, and `Project` tags at creation time using `--tags`.
- **Use infrastructure-as-code** (Bicep, Terraform, or ARM templates) for repeatable, version-controlled provisioning rather than imperative CLI commands for production workloads.
- **Store scripts in version control.** Every CLI command in this guide should be wrapped in a Bash script committed to a Git repository with change history.
- **Use `--output json` and pipe to `jq`** for programmatic validation in CI/CD pipelines rather than relying on `--output table` for human-readable output.

### Performance and Scalability

- Design `PartitionKey` values to distribute load evenly. Avoid single-partition designs (`PartitionKey=tasks` for all tasks) at scale. Partition by `userId`, `projectId`, or `dateRange`.
- Use batch operations (`az storage entity batch-insert`) for inserting large volumes of entities to reduce round-trip latency.
- Enable **Azure Monitor metrics** on the storage account to track transaction count, latency, and availability.

---

## Lessons Learned

### 1. Storage Account Names Must Be Globally Unique

Azure enforces global uniqueness on storage account names across all Azure tenants worldwide. A name collision returns an error immediately. Incorporate a random suffix (e.g., last 6 digits of subscription ID or a timestamp fragment) into naming conventions to avoid conflicts in automated pipelines.

### 2. Default TLS Version Is TLS 1.0

The provisioned storage account defaulted to `minimumTlsVersion: TLS1_0`. This is a known Azure default that represents a security risk. Always explicitly pass `--min-tls-version TLS1_2` during account creation. Security benchmarks (CIS Azure, Microsoft Defender for Cloud) will flag TLS 1.0 as a high-severity finding.

### 3. Shared Key Access Is Enabled by Default

Production environments must disable Shared Key access (`--allow-shared-key-access false`) and enforce Entra ID-only authentication. Shared keys, if leaked, provide full, unrestricted access to all data with no audit trail scoped to individual users.

### 4. `az storage table create` Is Idempotent

Running the create command against an existing table returns `"created": false` without throwing an error. This behavior makes it safe to include in idempotent deployment scripts and CI/CD pipelines without conditional logic guards.

### 5. Full Table Scans Are Costly at Scale

The `az storage entity query` command without an OData filter performs a full table scan. At hundreds of thousands of entities, this results in high latency and unnecessary transaction cost. Establishing a filter discipline from day one prevents expensive rearchitecting later.

### 6. Environment Variable Management Is Critical

Storing `STORAGE_KEY` as a plain shell variable works for lab exercises but is not acceptable in team or production environments. Treat all keys as secrets from the first line of code and enforce secret management through Azure Key Vault or CI/CD secret stores (GitHub Actions Secrets, Azure DevOps Variable Groups).

### 7. Resource Tagging Should Be Non-Negotiable

The storage account was created without resource tags. In enterprise environments, untagged resources cause cost attribution failures, compliance violations, and operational blind spots. Enforce tagging via Azure Policy with a `deny` effect for resources missing mandatory tags.

---

## Troubleshooting

| Issue | Likely Cause | Resolution |
|---|---|---|
| `StorageAccountAlreadyTaken` | Storage account name already globally registered | Append a unique numeric suffix to the account name |
| `AuthorizationFailure` on entity operations | `STORAGE_KEY` variable is empty or expired | Re-run key retrieval: `az storage account keys list` |
| `ResourceGroupNotFound` | Wrong resource group name or region context | Run `az group list` to confirm the exact group name |
| `TableAlreadyExists` on table create | Table was previously created | This is safe; `"created": false` is the expected response |
| `EntityAlreadyExists` on entity insert | Entity with same `PartitionKey` and `RowKey` exists | Use `az storage entity replace` or `az storage entity merge` to update |
| TLS handshake errors from client apps | Storage account defaulting to TLS 1.0 | Update account: `az storage account update --min-tls-version TLS1_2` |

---

## References

- [Azure Table Storage Documentation](https://learn.microsoft.com/en-us/azure/storage/tables/table-storage-overview)
- [az storage table CLI Reference](https://learn.microsoft.com/en-us/cli/azure/storage/table)
- [az storage entity CLI Reference](https://learn.microsoft.com/en-us/cli/azure/storage/entity)
- [Azure Storage Security Guide](https://learn.microsoft.com/en-us/azure/storage/common/storage-security-guide)
- [Azure Managed Identity for Storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-auth-aad-msi)
- [CIS Microsoft Azure Foundations Benchmark](https://www.cisecurity.org/benchmark/azure)
- [Azure Table Storage Design Patterns](https://learn.microsoft.com/en-us/azure/storage/tables/table-storage-design-patterns)

---

## Author

**Nautilus DevOps Engineering Team**
Platform: Azure Cloud | Environment: `AzureCloud` | Region: `eastus`
Provisioned: `2026-03-21` | CLI Version: `azure-cli 2.67.0`

---

*This documentation follows the Nautilus DevOps documentation standard. All infrastructure changes must be peer-reviewed, version-controlled, and linked to an approved change record before production deployment.*




<img width="1031" height="814" alt="image" src="https://github.com/user-attachments/assets/9dac55a1-d987-405d-89a9-06b574a5be40" />
<img width="1033" height="839" alt="image" src="https://github.com/user-attachments/assets/3de3ecac-3970-4fe1-ad92-fe2fba20c4db" />
<img width="1031" height="860" alt="image" src="https://github.com/user-attachments/assets/60c05979-8ba3-40b1-a224-04ce04096b48" />
<img width="1031" height="858" alt="image" src="https://github.com/user-attachments/assets/f1e0c5fb-73e9-4142-869b-cad0fcc69050" />
<img width="1032" height="856" alt="image" src="https://github.com/user-attachments/assets/345205f0-f930-457d-a488-0bcd256edf02" />
<img width="1029" height="360" alt="image" src="https://github.com/user-attachments/assets/410d4190-53f0-42a7-8fdd-13a31ac94066" />
<img width="1033" height="820" alt="image" src="https://github.com/user-attachments/assets/f417e947-9e7a-4157-889c-ee1ec3d2fd3f" />
<img width="1033" height="603" alt="image" src="https://github.com/user-attachments/assets/e09bed54-7eb0-4487-97db-6f5db2468121" />
<img width="1029" height="611" alt="image" src="https://github.com/user-attachments/assets/e69021f3-dd5c-4f59-aa29-5bcf5d601b40" />
<img width="1034" height="800" alt="image" src="https://github.com/user-attachments/assets/6673ad0c-853c-48a7-bdde-18a5ab4adc9d" />

