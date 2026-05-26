# Azure Cloud Engineering

<div align="center">

**Production-aligned Azure infrastructure implementations across compute, networking, storage, containers, databases, security, integration, and automation.**

[![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=flat-square&logo=microsoftazure&logoColor=white)](#)
[![ARM Templates](https://img.shields.io/badge/ARM_Templates-0089D6?style=flat-square&logo=microsoftazure&logoColor=white)](./automation)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](./containers)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](./containers)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](./compute)
[![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)](#)

</div>

---

## Overview

This directory contains **50+ production-style Azure implementations** organized by service domain. Each project was executed against live Azure Free Labs subscriptions under organizational policy enforcement, mirroring real-world enterprise constraints: SKU restrictions, policy-blocked resource types, pre-existing resource groups, and multi-region deployment requirements.

The work spans foundational cloud administration through advanced multi-service architectures, and is documented to a standard suitable for audit trails, onboarding, and portfolio review. All implementations follow CLI-first patterns with explicit post-deployment verification before sign-off.

---

## Repository Structure

```
azure/
├── compute/          # VM provisioning, sizing, disk management, and lifecycle operations
├── containers/       # ACR registry pipelines and AKS cluster provisioning
├── databases/        # Azure SQL provisioning, BACPAC export, and backup pipelines
├── integration/      # Event Hub streaming, cross-VM connectivity, and log aggregation
├── networking/       # VNet design, NSG authoring, load balancing, and gateway deployment
├── security/         # Key Vault cryptography and SSH hardening
├── solutions/        # Multi-service integration architectures
├── storage/          # Blob lifecycle, access control, static hosting, and data migration
└── README.md
```

---

## Domain Summaries

---

### [Compute](./compute/)

12 implementations covering the full Azure VM lifecycle: provisioning via Portal and CLI, SSH key management, cloud-init bootstrapping, VM right-sizing, disk attachment and online expansion, NIC configuration, static IP assignment, tagging, and App Service deployment. All labs executed under policy-restricted subscriptions enforcing `Standard_LRS` disk SKU and 128 GB OS disk limits.

**Representative projects:**

| Project | Quick Summary |
|---|---|
| [azure-vm-nginx-bootstrap](./compute/azure-vm-nginx-bootstrap/) | Zero-touch Nginx deployment via `--custom-data` cloud-init; validated with `curl -I` returning `HTTP 200` |
| [secure-linux-vm-provisioning](./compute/secure-linux-vm-provisioning/) | RSA 4096-bit SSH VM deployment with three documented Azure Policy failure recoveries |
| [vm-storage-expansion](./compute/vm-storage-expansion/) | OS disk expanded from 32 to 64 GiB; new data disk partitioned, formatted ext4, and UUID-mounted with `nofail` |
| [appservice-python-deployment](./compute/appservice-python-deployment/) | Python 3.11 App Service deployed CLI-only with governance tags applied at provisioning time |

---

### [Containers](./containers/)

Two production-pattern container infrastructure projects covering registry management, image lifecycle, and managed Kubernetes provisioning under policy constraints.

| Project | Quick Summary |
|---|---|
| [acr-setup](./containers/acr-setup/) | ACR provisioned; full Docker build-tag-push pipeline executed with digest-level verification (`sha256:395c45bb...`) |
| [aks-private-cluster-provisioning](./containers/aks-private-cluster-provisioning/) | Private AKS cluster (Kubernetes 1.33.0) with cluster autoscaler; silent Managed Prometheus activation caught and disabled via separate API surface audit |

**Key decisions:** Kubernetes version pinned after region-specific availability query. JMESPath silent empty return traced and resolved before downstream provisioning to prevent cascading failures.

---

### [Databases](./databases/)

Two implementations covering Azure SQL deployment and a full BACPAC backup and recovery pipeline, documenting production credential hygiene and export integrity verification.

| Project | Quick Summary |
|---|---|
| [sql-database-deployment](./databases/sql-database-deployment/) | Azure SQL Server and Basic-tier database provisioned via CLI; `PasswordNotComplex` error diagnosed and resolved |
| [azure-sql-data-protection-and-archival](./databases/azure-sql-data-protection-and-archival/) | BACPAC export to Blob Storage via SAS token; artifact verified locally at byte level (2771 bytes, source to destination) |

**Key decisions:** SAS token scoped to a 3-hour window. `errorMessage: null` confirmed in API response before proceeding to download. Firewall scoped to Azure services only (`0.0.0.0/0.0.0.0`).

---

### [Integration](./integration/)

Three multi-service pipeline implementations covering event streaming, cross-VM application connectivity, and dual-target log aggregation.

| Project | Quick Summary |
|---|---|
| [centralized-telemetry-ingestion-pipeline](./integration/centralized-telemetry-ingestion-pipeline/) | Python `EventHubProducerClient` streaming VM logs to Event Hubs; 120 incoming frames and 8,569 bytes confirmed via Azure Monitor metrics |
| [cross-vm-application-database-integration](./integration/cross-vm-application-database-integration/) | PHP app on East US VM connected to MySQL on Central US VM; `Connected successfully` confirmed in browser and via `curl` |
| [event-driven-log-aggregation-system](./integration/event-driven-log-aggregation-system/) | Single Python script publishing logs to both Event Hubs and Blob Storage; `AppendBlob` type confirmed with 18-byte payload |

**Key decisions:** AMQP framing overhead (120 frames vs. 60 application events) documented as expected behavior. `|` delimiter used in `sed` to avoid forward-slash conflicts in connection string URLs.

---

### [Networking](./networking/)

15 implementations covering VNet design, NSG authoring, load balancing, gateway deployment, VNet peering, and incident diagnosis and remediation.

| Project | Quick Summary |
|---|---|
| [application-gateway-vm-backend-orchestration](./networking/application-gateway-vm-backend-orchestration/) | Application Gateway deployed via `az rest` ARM PUT after CLI validator rejected all available SKUs; backend health confirmed `Healthy` |
| [vm-egress-diagnostics](./networking/vm-egress-diagnostics/) | Complete VM outbound failure traced to `Deny *` NSG rule at priority 200; resolved with `az network nsg rule delete` and verified atomically |
| [vnet-peering-connectivity](./networking/vnet-peering-connectivity/) | Bidirectional VNet peering between `10.2.0.0/16` and `10.1.0.0/16`; ICMP confirmed 0% packet loss at 1.417 ms average |
| [ingress-traffic-orchestration](./networking/ingress-traffic-orchestration/) | Standard Load Balancer with HTTP health probe fronting Nginx VM; `HTTP 200` confirmed via `curl` |
| [private-vnet-compute-isolation](./networking/private-vnet-compute-isolation/) | VM deployed with no public IP; NSG bound at both subnet and NIC layers for defense in depth |

---

### [Security](./security/)

Two foundational security implementations: cloud-managed cryptography via Azure Key Vault and hardened root SSH access on a live Ubuntu VM.

| Project | Quick Summary |
|---|---|
| [key-vault-cryptography](./security/key-vault-cryptography/) | RSA-OAEP 4096-bit encrypt/decrypt round-trip on a sensitive file; `diff` and `md5sum` confirmed byte-for-byte integrity; all operations server-side |
| [ssh-key-authentication](./security/ssh-key-authentication/) | Public key injected to `/root/.ssh/authorized_keys` via compound SSH command; `PermitRootLogin prohibit-password` enforced; root login verified without password prompt |

---

### [Solutions](./solutions/)

Two multi-service integration architectures combining networking, compute, and storage into validated end-to-end deployments.

| Project | Quick Summary |
|---|---|
| [application-gateway-vm-ingress-traffic-distribution](./solutions/application-gateway-vm-ingress-traffic-distribution/) | Application Gateway distributing HTTP traffic across two Nginx VMs via round-robin; 10 sequential `curl` requests confirmed alternating Version 1 and Version 2 responses |
| [private-blob-nginx-static-content-delivery](./solutions/private-blob-nginx-static-content-delivery/) | Private Blob container as a static content source; VM fetches `index.html` at deploy time and serves it via Nginx; 233-byte response confirmed externally |

---

### [Storage](./storage/)

12 implementations covering blob lifecycle management, container access control, static website hosting, VM-to-storage integration, data migration, and Table Storage provisioning.

| Project | Quick Summary |
|---|---|
| [blob-container-migration](./storage/blob-container-migration/) | `nautilus.txt` migrated between containers with MD5 checksum, binary diff, and a 13-gate verification checklist; all gates passed |
| [blob-lifecycle-governance](./storage/blob-lifecycle-governance/) | Automated deletion policy for `blockBlob` objects older than 7 days; scoped with `prefixMatch`; estimated 60-90% storage spend reduction for log workloads |
| [azure-static-website-hosting](./storage/azure-static-website-hosting/) | StorageV2 static website enabled; `index.html` served at `HTTP 200` with correct content type confirmed via `curl -I` |
| [azure-vm-acr-blob-integrated-workload](./storage/azure-vm-acr-blob-integrated-workload/) | Flask app containerized, pushed to ACR, deployed to VM; runtime config externalized to Blob Storage and volume-mounted to resolve `HTTP 500` |
| [table-storage-task-lifecycle](./storage/table-storage-task-lifecycle/) | Azure Table Storage backend for a To-Do app; two entities inserted and verified individually and via full OData table scan |
| [Blob-Transient-Asset-Egress-and-Decommission](./storage/Blob-Transient-Asset-Egress-and-Decommission/) | Container decommissioned after batch download; three-step deletion audit confirmed clean removal (33-byte integrity, `container exists: false`) |

---

## Technologies and Tools

| Category | Tools and Services |
|---|---|
| Cloud Platform | Microsoft Azure |
| CLI and API | Azure CLI 2.67.0+, ARM REST API (`az rest`), JMESPath `--query` filters |
| Compute | Azure Virtual Machines (Ubuntu 22.04 LTS), Azure App Service, `Standard_B1s/B2s/D2s_v3` |
| Containers | Azure Container Registry (Basic SKU), Azure Kubernetes Service, Docker Engine 29.2.1 |
| Orchestration | Kubernetes 1.33.0, AKS Cluster Autoscaler |
| Storage | Azure Blob Storage, Azure Table Storage, Azure Managed Disks (`Standard_LRS`) |
| Databases | Azure SQL Database (Basic tier, DTU model), MySQL 5.7 |
| Networking | VNet, Subnet, NSG, VNet Peering, Azure Load Balancer (Standard), Application Gateway (Basic) |
| Security | Azure Key Vault (RSA-OAEP 4096-bit), SSH key pairs, RBAC, `sshd_config` hardening |
| Integration | Azure Event Hubs (Standard SKU), Azure Monitor Metrics |
| SDKs | `azure-eventhub 5.15.1`, `azure-storage-blob 12.28.0`, Python 3.10, PHP 8.x |
| IaC and Scripting | ARM Templates, Bash, `sed`, `scp`, cloud-init (`--custom-data`) |
| Runtimes | Flask (Werkzeug), Nginx 1.18.0, Apache2 |
| Auth | Storage Account Keys, SAS tokens, Azure AD (`--auth-mode login`), Managed Identity |

---

## Key Outcomes and Skills Demonstrated

**Infrastructure and Provisioning**
CLI-first resource deployment across all service domains with no portal dependency for verification. Dynamic resource group resolution via `az group list --query` makes all command patterns portable across ephemeral lab environments. ARM REST API used as a controlled escape hatch when CLI validators conflicted with subscription policy.

**Security Posture**
Password authentication disabled at provisioning time across all applicable VMs. Public blob access explicitly set rather than assumed. NSG rules applied at both subnet and NIC layers for defense-in-depth enforcement. SAS tokens scoped by time window and permission surface. Key Vault operations executed entirely server-side with zero private key exposure.

**Operational Troubleshooting**
Real failures documented with root cause analysis and verified resolutions across all domains: policy-blocked SKUs, silent JMESPath empty returns, AMQP frame counting discrepancies, orphaned ghost disks, `PasswordNotComplex` rejections, stuck VM states, and Azure CLI breaking changes. Each resolution includes a prevention note.

**Pipeline and Integration Design**
Config externalized from Docker images at runtime via Blob Storage volume mount. Dual-target log pipeline confirmed with `AppendBlob` type and Azure Monitor metric verification. BACPAC export pipeline validated with byte-level integrity check before local artifact confirmation.

**Verification Discipline**
All deployments include a structured post-deployment gate: `az ... show --query` for resource state, `curl` for HTTP reachability, `md5sum`/`diff` for data integrity, and Azure Monitor metrics for event delivery. No implementation is signed off on expected behavior alone.

---

## How to Navigate

Each subdirectory contains a domain-level `README.md` indexing the projects within it. Each project folder contains a self-contained `README.md` with:

- Full CLI command sequences with expected output
- Inline screenshots at every major verification step
- An errors section with root cause, failed command, and verified resolution
- A best practices and lessons learned section scoped to production applicability

**To reproduce any project:**

```bash
# Authenticate
az login

# Set target subscription
az account set --subscription <subscription-id>

# Resolve resource group dynamically (used across all projects)
RG=$(az group list --query "[0].name" --output tsv)
echo "Resource Group: $RG"
```

Then follow the phase sequence in the target project README. All commands are written for Bash with Azure CLI authenticated via `az login` or a pre-configured service principal.

**Recommended reading order for new contributors:**

1. Start with [`compute/azure-vm-nginx-bootstrap`](./compute/azure-vm-nginx-bootstrap/) for a clean, policy-aware VM deployment baseline
2. Progress to [`networking/vm-egress-diagnostics`](./networking/vm-egress-diagnostics/) and [`networking/vnet-peering-connectivity`](./networking/vnet-peering-connectivity/) for network troubleshooting patterns
3. Review [`storage/blob-container-migration`](./storage/blob-container-migration/) and [`storage/blob-lifecycle-governance`](./storage/blob-lifecycle-governance/) for storage operations
4. Explore [`solutions/application-gateway-vm-ingress-traffic-distribution`](./solutions/application-gateway-vm-ingress-traffic-distribution/) for the most complete multi-service architecture

---

## Related Repositories

| Repository | Description |
|---|---|
| [Cloud and DevOps Engineering](../README.md) | Full portfolio README covering DevOps, AWS, and Azure |
| [AWS Cloud Engineering](../aws/README.md) | 50+ AWS implementations across compute, networking, storage, and containers |
| [DevOps Engineering](../devops/README.md) | 100+ implementations across Linux, Docker, Kubernetes, Jenkins, Ansible, and Terraform |

---

<div align="center">

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/arinze-edeh)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/arinze-edeh)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white)](mailto:edeharinze389@gmail.com)

<sub>50+ production-grade Azure implementations across 8 service domains.</sub>

</div>
