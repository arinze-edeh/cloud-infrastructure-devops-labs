# Azure Databases

![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Azure SQL](https://img.shields.io/badge/Azure_SQL-CC2927?style=for-the-badge&logo=microsoft-sql-server&logoColor=white)
![Azure CLI](https://img.shields.io/badge/Azure_CLI-0078D4?style=for-the-badge&logo=windows-terminal&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=for-the-badge)

> Hands-on Azure SQL Database projects covering the full lifecycle: provisioning, public access configuration, BACPAC export, blob-based archival, and verified local recovery. Work is scoped to production-relevant patterns used in cloud migration, backup management, and disaster recovery readiness.

---

## Directory Structure

```
azure/databases/
├── azure-sql-data-protection-and-archival/   # BACPAC export pipeline with blob storage and local recovery verification
├── sql-database-deployment/                  # Azure SQL Server and Database provisioning via Azure CLI
└── README.md
```

---

## Project Summaries

### [azure-sql-data-protection-and-archival](./azure-sql-data-protection-and-archival/)

**Quick Summary:** Provisions an Azure SQL Database, exports a full BACPAC backup to Blob Storage using a scoped SAS token, and verifies the backup artifact locally on the client host.

| | |
|---|---|
| **Purpose** | Simulate a production backup and recovery pipeline for a database undergoing cloud migration. Covers the full backup lifecycle from database provisioning to local artifact verification. |
| **Approach** | Provisioned an Azure SQL Server and Basic-tier database in `westus`. Created a `Standard_LRS` storage account and blob container. Generated a time-scoped SAS token and used `az sql db export` to write a BACPAC to the container. Downloaded and verified the artifact on the `azure-client` host using size consistency as the primary integrity indicator. |
| **Key Decisions** | SAS token scoped to a 3-hour window to limit credential exposure. Storage key masked in all terminal output. Firewall rule set to `0.0.0.0/0.0.0.0` (Azure services only pattern) rather than open internet access. Export status validated via `errorMessage: null` in the API response before proceeding to download. |
| **Outcome** | BACPAC exported and confirmed at 2771 bytes both in blob storage and locally at `/opt/xfusion-db-backup.bacpac`. All five acceptance criteria passed. Export completed in approximately 4 minutes via the Azure Import/Export service agent. |

---

### [sql-database-deployment](./sql-database-deployment/)

**Quick Summary:** Deploys a publicly accessible Azure SQL Server and Basic-tier database via Azure CLI, including firewall configuration, password policy compliance, and full deployment verification.

| | |
|---|---|
| **Purpose** | Establish a repeatable, auditable Azure SQL Database deployment pattern suitable for CI/CD integration and infrastructure migration workflows. |
| **Approach** | Authenticated via Azure CLI, provisioned an Azure SQL Server in `centralus`, and configured a firewall rule allowing public access per task requirements. Created a Basic-tier database with locally-redundant backup storage. Encountered and resolved a `PasswordNotComplex` error caused by a password containing a substring of the admin username. |
| **Key Decisions** | Used `--backup-storage-redundancy Local` to satisfy task constraints while documenting production alternatives (Geo, Zone). Firewall range set to `0.0.0.0 - 255.255.255.255` intentionally per requirements, with hardening recommendations documented. Password corrected to eliminate any character overlap with the admin login, resolving the Azure complexity policy rejection. |
| **Outcome** | SQL Server provisioned in `Ready` state with `publicNetworkAccess: Enabled`. Database confirmed `Online` with all eight verification criteria passed. Deployment is parameterized and ready for pipeline integration. |

---

## Technologies and Tools

| Tool / Service | Role |
|---|---|
| Azure CLI (`az`) | Primary provisioning and management interface across all tasks |
| Azure SQL Database | Managed relational database service (Basic tier, DTU model) |
| Azure Blob Storage | Backup artifact storage (Standard LRS, StorageV2) |
| SAS Token (Shared Access Signature) | Scoped, time-bound credential for export authorization |
| BACPAC Format | Portable Azure SQL backup format combining schema and data |
| Bash | Variable management, command chaining, and local verification |

---

## Key Outcomes and Skills Demonstrated

- Provisioned Azure SQL infrastructure (server, database, firewall) end-to-end via CLI with no portal dependency
- Constructed a working BACPAC export pipeline using SAS-authenticated blob storage as the backup target
- Applied credential hygiene practices: masked storage keys, time-scoped SAS tokens, no hardcoded secrets in commands
- Diagnosed and resolved a `PasswordNotComplex` policy rejection with root cause analysis and a documented fix
- Validated backup integrity via byte-level consistency between blob metadata and local file size
- Documented production hardening gaps (TLS version, storage redundancy, SAS permission scope) identified during execution

---

## How to Navigate

Each subdirectory contains a self-contained `README.md` runbook with:

- Full CLI command sequences with expected output
- Inline screenshots at every major step
- An errors section covering root cause, failed command, and verified resolution
- A verification checklist confirming all acceptance criteria
- Best practices and lessons learned sections scoped to production readiness

Start with [`sql-database-deployment`](./sql-database-deployment/) for foundational provisioning patterns, then proceed to [`azure-sql-data-protection-and-archival`](./azure-sql-data-protection-and-archival/) for the backup and recovery pipeline built on top of that foundation.

---

> Part of the [`cloud-infrastructure-devops-labs`](../../) portfolio. See the root README for full repository structure and navigation.
