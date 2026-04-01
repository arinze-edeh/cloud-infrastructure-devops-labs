# Azure Storage Engineering

![Azure](https://img.shields.io/badge/Azure-Storage-0078D4?style=flat-square&logo=microsoftazure&logoColor=white)
![CLI](https://img.shields.io/badge/Tool-Azure%20CLI-0078D4?style=flat-square&logo=microsoftazure)
![Projects](https://img.shields.io/badge/Projects-12-blueviolet?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

A collection of 12 hands-on Azure Storage labs covering blob lifecycle management, container security hardening, static website hosting, VM-to-storage integration, data migration pipelines, and Table Storage provisioning. Each project follows production-aligned patterns: CLI-driven execution, structured verification, security-aware configuration, and documented trade-off decisions.

---

## Directory Structure

```
azure/storage/
├── Blob-Transient-Asset-Egress-and-Decommission/
├── azure-static-website-hosting/
├── azure-vm-acr-blob-integrated-workload/
├── blob-container-access-hardening/
├── blob-container-deployment/
├── blob-container-migration/
├── blob-lifecycle-governance/
├── cli_blob_ingestion/
├── managed-disk-provisioning/
├── public-blob-storage-deployment/
├── table-storage-task-lifecycle/
├── vm-blob-storage-integration/
└── README.md
```

---

## Project Summaries

---

### [Blob Transient Asset Egress and Decommission](./Blob-Transient-Asset-Egress-and-Decommission/)

**Quick Summary:** Safely migrates all blobs from a private container to the local host, then permanently deletes the container with multi-layer verification to confirm clean decommission.

**Purpose:** Automate the end-of-life phase of a migration project, decommissioning a transient storage container after its contents are confirmed safe on the target host.

**Approach:** Resolves the resource group dynamically using `--query` filters to avoid hardcoding. Downloads blobs via `az storage blob download-batch`, then runs a three-step deletion audit: `container delete`, `container exists`, and `container list` to confirm complete removal.

**Outcome:** Container `xfusion-blob-11509` fully decommissioned. File integrity confirmed via byte count comparison (33 bytes, source to destination). Empty storage account state validated post-deletion.

---

### [Azure Static Website Hosting](./azure-static-website-hosting/)

**Quick Summary:** Deploys a publicly accessible static website on Azure Blob Storage, covering account creation, static website feature enablement, public access configuration, and HTTP reachability verification.

**Purpose:** Serve lightweight static content (documentation portals, status pages, landing pages) without provisioning a full web server or container infrastructure.

**Approach:** Creates a `StorageV2` account (required for static website support), enables the feature via `az storage blob service-properties update --static-website`, explicitly sets `--allow-blob-public-access true` (disabled by default post-2023), uploads `index.html` with `--content-type "text/html"`, and validates with `curl -I`.

**Outcome:** Site live at `https://devopswebst23591.z13.web.core.windows.net/` returning `HTTP/1.1 200 OK` with correct content type. Browser render confirmed.

**Key Trade-off:** `allowBlobPublicAccess` and static website hosting are independent settings; omitting either causes silent failure. Both require explicit configuration.

---

### [Azure VM, ACR, and Blob Integrated Workload](./azure-vm-acr-blob-integrated-workload/)

**Quick Summary:** End-to-end containerized deployment pipeline: Flask app built and pushed to ACR, deployed to a VM, and configured dynamically via a `config.json` downloaded from Blob Storage at runtime.

**Purpose:** Eliminate hardcoded config from container images by externalizing runtime configuration to Blob Storage, enabling environment-specific deployments without image rebuilds.

**Approach:** Provisions VM (`Standard_B1s`, Ubuntu 22.04 LTS), ACR (Basic SKU), and Blob Storage account via CLI. Builds and pushes a `python:3.9-slim`-based Docker image to ACR. Remotely installs Docker and Azure CLI on the VM via SSH, then pulls the image and launches the container. A runtime `HTTP 500` triggered by a missing `config.json` inside the container was resolved by downloading the file from Blob Storage and relaunching with a `-v` volume mount.

**Outcome:** Container serving `HTTP/1.1 200 OK` with clean Flask startup logs. Image digest `sha256:cef0472...` confirmed in ACR. Full pipeline validated from build through runtime.

**Key Trade-off:** Config externalization requires an active delivery mechanism at runtime; uploading to Blob Storage is not sufficient without a download step on the VM host.

---

### [Blob Container Access Hardening](./blob-container-access-hardening/)

**Quick Summary:** Converts a publicly accessible blob container to private access using Azure CLI, with pre- and post-change verification and no data loss.

**Purpose:** Remediate a misconfigured public container as part of a cloud security governance initiative, aligning storage configuration with internal-only access requirements.

**Approach:** Confirms current public access level with `az storage container show`, attempts `--auth-mode login` (unsupported for this operation), then falls back to key-based authentication via `az storage container set-permission --public-access off`. Verifies the change returns `null` for `publicAccess` and audits all containers to confirm no unintended changes to adjacent private containers.

**Outcome:** `nautilus-container-11818` converted from public to private. Both containers in the account confirmed private post-change. No container recreation or data loss.

---

### [Blob Container Deployment](./blob-container-deployment/)

**Quick Summary:** Provisions a private Azure Blob container and a public Blob container as separate tasks, covering both security postures with appropriate verification.

**Purpose:** Baseline provisioning pattern for Azure Blob Storage, supporting both internal-only data and publicly accessible blob workloads.

**Approach:** Creates `StorageV2` accounts with `Standard_LRS`. Uses `--public-access off` for the private container and `--public-access container` with `--allow-blob-public-access true` for the public container. Verifies each container's access level with `az storage container show`.

**Outcome:** Both containers provisioned and validated. Access levels confirmed matching requirements in each case.

---

### [Blob Container Migration](./blob-container-migration/)

**Quick Summary:** Migrates `nautilus.txt` between two blob containers within the same storage account, with MD5 checksum verification, binary diff, and a 13-gate verification checklist.

**Purpose:** Safely relocate blob data between containers as part of a structured storage reorganization, with deterministic integrity guarantees.

**Approach:** Loads the account key into a shell variable to avoid `+` character misinterpretation on raw paste. Creates the destination container with `--public-access off`. Copies the blob using `az storage blob copy start` and polls until `copy_status: success`. Validates integrity via MD5 checksum comparison, content length comparison, and a `diff` of downloaded local copies before cleanup.

**Outcome:** All 13 verification gates passed. `nautilus.txt` confirmed in both containers with matching checksums (`Lu7zilatbGguzSz2Ecn5IQ==`), identical size (33 bytes), and zero binary diff output.

**Key Trade-off:** Shell variable injection for account keys is mandatory when keys contain `+` characters; direct paste causes silent authentication failures.

---

### [Blob Lifecycle Governance](./blob-lifecycle-governance/)

**Quick Summary:** Implements an automated blob deletion policy on Azure Blob Storage that removes `blockBlob` objects older than 7 days, scoped to a specific container using `prefixMatch`.

**Purpose:** Eliminate manual cleanup operations and cap storage costs by automating data retention for time-bound blob workloads (logs, temp files, migration artifacts).

**Approach:** Provisions a `StorageV2` account (required for lifecycle management support), creates a scoped lifecycle policy JSON with `daysAfterModificationGreaterThan: 7`, and applies it via `az storage account management-policy create`. Validates with `--query "policy.rules[?name=='nautilus-del-rule']"` to confirm all six rule attributes.

**Outcome:** Policy `nautilus-del-rule` active and verified. Estimated 60 to 90% reduction in blob storage spend for typical log workloads. Execution is asynchronous (Azure evaluates once per 24-hour cycle).

**Key Trade-off:** `StorageV2` kind is non-negotiable for lifecycle management. The `BlobStorage` kind silently lacks support for management policies.

---

### [CLI Blob Ingestion](./cli_blob_ingestion/)

**Quick Summary:** Uploads a local file to an existing Azure Blob container using `--auth-mode login` (Azure AD authentication), demonstrating credential-free blob ingestion aligned with RBAC best practices.

**Purpose:** Establish a baseline blob upload pattern that avoids shared key authentication, using Azure AD-backed identity for auditability and least-privilege access.

**Approach:** Verifies account context, confirms file existence on the local host, then uploads using `az storage blob upload --auth-mode login`. Validates the upload with `az storage blob list --output table` confirming blob type, size, content type, and last-modified timestamp.

**Outcome:** `xfusion.txt` (33 bytes, `text/plain`) confirmed in `xfusion-blob-29364`. Server-side encryption confirmed via `request_server_encrypted: true`.

---

### [Managed Disk Provisioning](./managed-disk-provisioning/)

**Quick Summary:** Provisions a 2 GiB `Standard_LRS` Azure Managed Disk via CLI as part of an incremental VM storage migration, confirming disk state, SKU, and attachment readiness.

**Purpose:** Prepare a managed disk for VM attachment as a discrete, auditable infrastructure step in a phased cloud migration.

**Approach:** Resolves the resource group with `az group list --query`, then creates the disk via `az disk create`. Verifies with `az disk show --output table` confirming `provisioningState: Succeeded`, `diskState: Unattached`, `diskSizeGb: 2`, and `sku: Standard_LRS`.

**Outcome:** `nautilus-disk` provisioned and ready for compute integration. Disk is unattached, LRS-replicated, and CLI-managed with a clean audit trail.

---

### [Public Blob Storage Deployment](./public-blob-storage-deployment/)

**Quick Summary:** Provisions a public-access Blob container with anonymous read access enabled at the container level, suitable for hosting publicly accessible migration assets.

**Purpose:** Support data consolidation scenarios requiring publicly accessible blob content without a CDN or web server intermediary.

**Approach:** Creates the storage account with `--allow-blob-public-access true`, then creates the container with `--public-access container`. Verifies access level returns `"PublicAccess": "container"` via `az storage container show --query`.

**Outcome:** `devops-blob-23782` accessible with anonymous read access confirmed. Deployment validated via CLI query.

---

### [Table Storage Task Lifecycle](./table-storage-task-lifecycle/)

**Quick Summary:** Provisions an Azure Table Storage backend for a To-Do application, inserts two task entities with `description` and `status` fields, and validates persistence via individual reads and a full table query.

**Purpose:** Demonstrate Azure Table Storage as a cost-efficient, schema-less NoSQL backend for lightweight task management workloads without relational database overhead.

**Approach:** Provisions a `StorageV2` account with `Standard_LRS`. Creates a `tasks` table, inserts two entities (composite key: `PartitionKey=tasks`, `RowKey=1|2`), verifies each entity with `az storage entity show`, and confirms the full dataset with `az storage entity query --output table`.

**Outcome:** Both entities persisted correctly: Task 1 (`status: completed`), Task 2 (`status: in-progress`). Full table scan returned exactly 2 rows with correct field values.

**Key Trade-off:** Single-partition design (`PartitionKey=tasks`) is appropriate for lab scale. Production workloads with high entity counts require partitioning by `userId` or `projectId` to prevent hotspots.

---

### [VM to Blob Storage Integration](./vm-blob-storage-integration/)

**Quick Summary:** Provisions an Azure VM and a private Blob Storage account, then establishes an authenticated CLI-based upload path from inside the VM to the container, validated end-to-end.

**Purpose:** Establish a repeatable, auditable pattern for VM-generated data to be persisted directly to Blob Storage, foundational for log archival, backup, and data pipeline workloads.

**Approach:** Creates the storage account with `--allow-blob-public-access false`. Retrieves the VM public IP dynamically via `az vm list-ip-addresses --query`. SSHs into the VM, creates a test file, and uploads via `az storage blob upload` with key-based auth. Verifies the blob listing shows correct metadata: `BlockBlob`, `Hot` tier, expected content type and byte count.

**Outcome:** `testfile.txt` confirmed in `devops-container16287` with server-side encryption active. Full end-to-end path from VM filesystem to cloud storage validated. IP retrieval is fully scriptable, resilient to VM deallocation/reallocation cycles.

---

## Technologies and Tools

| Category | Tools and Services |
|---|---|
| Cloud Platform | Microsoft Azure |
| CLI Tooling | Azure CLI 2.67.0+ |
| Compute | Azure VM (Ubuntu 22.04 LTS, `Standard_B1s`) |
| Container Registry | Azure Container Registry (Basic SKU) |
| Storage Services | Azure Blob Storage, Azure Table Storage, Azure Managed Disks |
| Storage SKU | `Standard_LRS` (StorageV2) |
| Container Runtime | Docker (on-VM), `python:3.9-slim` base image |
| Application Runtime | Python 3.9 + Flask (Werkzeug) |
| Authentication | Service Principal, Storage Account Keys, Azure AD (`--auth-mode login`) |
| Networking | NSG rules, SSH key-based VM access |
| Scripting | Bash, shell variable injection, JMESPath `--query` filters |

---

## Key Outcomes and Skills Demonstrated

**Storage Operations**
- Blob upload, batch download, cross-container copy with integrity verification (MD5, binary diff, byte count)
- Container access control: private vs. public, hardening public-to-private conversion
- Static website hosting: end-to-end from account creation through browser-confirmed render

**Automation and CLI Patterns**
- Dynamic resource group resolution via `--query` to eliminate hardcoded values across all projects
- Shell variable injection for credentials to handle special character edge cases in storage keys
- Scripted IP retrieval and SSH automation for fully headless VM provisioning workflows

**Security Practices**
- Explicit `--allow-blob-public-access false` and `enableHttpsTrafficOnly` enforcement
- RBAC-aligned upload via `--auth-mode login` where supported
- `--auth-mode login` limitation identification and documented key-based fallback

**Lifecycle and Governance**
- Automated blob retention via management policies with `prefixMatch` scoping
- Multi-step deletion verification (action, boolean check, full audit) as a production runbook standard
- Resource group RBAC boundary identification and constraint-compliant provisioning

**Integration Architecture**
- Config externalization from Docker images with Blob Storage as the runtime delivery mechanism
- VM-to-ACR-to-container deployment pipeline with end-to-end HTTP validation
- Table Storage entity design with composite key strategy and OData query patterns

---

## How to Navigate

Each subdirectory contains a self-contained `README.md` with:

- A problem statement and architecture diagram
- Step-by-step CLI commands with expected output
- Screenshot placeholders at every verification point
- An error and resolution section (where applicable)
- Best practices and lessons learned sections

**Recommended reading order for new contributors:**

1. Start with `cli_blob_ingestion` and `blob-container-deployment` for baseline blob patterns
2. Progress to `blob-container-migration` and `Blob-Transient-Asset-Egress-and-Decommission` for lifecycle operations
3. Review `blob-container-access-hardening` and `blob-lifecycle-governance` for security and governance patterns
4. Explore `azure-vm-acr-blob-integrated-workload` for the full integration architecture

**Reusing commands:** All projects use parameterized, kebab-case consistent resource names. Commands referencing `--resource-group $(az group list --query "[0].name" -o tsv)` are portable across lab environments without modification.

---

