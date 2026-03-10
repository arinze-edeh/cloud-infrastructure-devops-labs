# Azure Cloud Engineering

<div align="center">

**Production-aligned Azure infrastructure implementations across compute, networking, storage, containers, databases, security, integration, and automation.**

[![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=flat-square&logo=microsoftazure&logoColor=white)](#)
[![ARM Templates](https://img.shields.io/badge/ARM_Templates-0089D6?style=flat-square&logo=microsoftazure&logoColor=white)](./automation)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](./containers)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](./containers)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](./compute)

</div>

---

## Overview

This directory contains **50+ production-grade Azure implementations** organised by service domain. Each implementation addresses a specific infrastructure, security, networking, or architecture challenge modelled on real-world Azure environments. The work spans foundational cloud administration through advanced multi-service architectures covering virtual machines, identity and access control, networking, storage, containers, databases, event-driven integration, and infrastructure automation.

Every implementation is fully documented with configuration details, architectural decisions, and validation outcomes.

---

## Repository Structure

```
azure/
├── automation/       # ARM Templates and CLI-driven infrastructure provisioning
├── compute/          # Virtual machine provisioning, sizing, and lifecycle management
├── containers/       # ACR, AKS, and containerised application deployments
├── databases/        # Azure SQL, MySQL provisioning and database operations
├── integration/      # Event Hub, Web Applications, and service integration pipelines
├── networking/       # VNet, subnets, peering, Application Gateway, and connectivity
├── security/         # Key Vault, NSG, managed identities, and access control
├── storage/          # Blob Storage, Table Storage, and lifecycle management
└── README.md
```

---

## Domain Implementations

### Compute

Provisioned and managed Azure Virtual Machines across a range of configurations using both the Azure Portal and Azure CLI for environment parity and automation readiness. Deployed VMs with SSH key pair authentication for secure remote access, performed live VM size changes for workload right-sizing without redeployment, and applied resource tags for cost allocation and governance across environments. Attached and expanded Managed Disks to running VMs for online storage growth, configured Network Interface Cards for multi-homed VM connectivity, and assigned both dynamic and static Public IP addresses for stable external endpoints. Automated VM initialisation using user data scripts for consistent, repeatable instance configuration at first boot, eliminating manual post-deployment steps and reducing environment drift. Provisioned VMs with scoped managed identities for credential-free, RBAC-governed authentication against Azure services.

**Key implementations:** VM provisioning via Portal and CLI, SSH key pair management, VM right-sizing, resource tag management, Managed Disk attachment and online expansion, NIC configuration, static and dynamic Public IP assignment, user data script automation, managed identity provisioning and scoping.

---

### Networking

Engineered Azure network architectures for both internet-facing and privately segmented workload deployments. Designed Virtual Networks with custom IPv4 CIDR addressing and subnet segmentation to enforce workload isolation between application tiers. Deployed VMs into public VNets with internet routing for externally accessible applications and into private VNets with restricted outbound paths for sensitive backend services. Configured VNet Peering for private, low-latency cross-VNet communication without traffic traversing the public internet, enabling secure inter-environment connectivity. Enabled controlled outbound internet access for private VM workloads and integrated Azure Application Gateway as a Layer 7 load balancer with SSL termination, path-based routing rules, and backend VM pool configuration for health-checked application delivery. Diagnosed and resolved a public VNet configuration failure where VM internet connectivity was blocked, tracing the fault through route tables, NSG rules, and Public IP association to identify and remediate the root cause.

**Key implementations:** VNet and subnet design with custom IPv4 CIDR, public and private VNet deployments, VNet Peering, Application Gateway with SSL termination and path-based routing, backend VM pool configuration, outbound internet access for private VMs, public VNet connectivity fault diagnosis and resolution.

---

### Security

Designed and enforced Azure security architectures aligned to the principle of least privilege across network controls, identity, and secrets management. Authored Network Security Group rule sets with granular port-level ingress and egress controls to enforce security boundaries at both the subnet and NIC levels, ensuring only explicitly permitted traffic flows were allowed. Centralised application secrets, connection strings, and cryptographic key management using Azure Key Vault with RBAC-enforced access policies, eliminating hardcoded credentials from application code and configuration files. Provisioned VMs with scoped managed identities to enable secure, credential-free service-to-service authentication against Azure Key Vault, storage accounts, and other Azure resources without storing access keys.

**Key implementations:** NSG inbound and outbound rule authoring, subnet-level and NIC-level NSG association, Azure Key Vault deployment and configuration, secret and connection string centralisation, RBAC access policy enforcement on Key Vault, managed identity provisioning for credential-free service authentication.

---

### Storage

Designed and operated Azure storage solutions spanning object storage, structured NoSQL storage, and policy-driven data lifecycle management. Provisioned private Blob Storage containers for internal application data with restricted access and public containers for externally accessible static assets. Executed bulk data uploads and cross-container migrations using Azure CLI scripting, and enforced access tier transitions from public to private to reduce attack surface as workload access patterns evolved. Configured storage lifecycle management policies to automate object tiering to cool and archive tiers based on last-modified timestamps, optimising storage costs at scale. Managed Azure Table Storage for structured, low-latency key-value NoSQL workloads requiring schema-flexible data access. Performed blob container backup operations and validated controlled deletion procedures with recovery confirmation.

**Key implementations:** Private and public Blob Storage container provisioning, CLI-based bulk data migration, public-to-private access tier conversion, storage lifecycle policy configuration, cool and archive tier automation, Azure Table Storage management, blob container backup and recovery validation, storage account integration with VM workloads.

---

### Databases

Provisioned and operated Azure managed and self-managed database services for relational workloads across varying configuration requirements. Deployed Azure SQL Database instances with firewall rule configuration and connection string management for secure, application-level backend access. Executed SQL database schema migration and environment setup procedures for application onboarding, validating data integrity, connectivity, and query performance post-migration. Deployed MySQL database servers on Azure VMs for workloads requiring custom engine configurations, fine-grained parameter tuning, and plugin management outside the constraints of fully managed database offerings.

**Key implementations:** Azure SQL Database provisioning, firewall rule and connection string configuration, SQL schema migration and post-migration validation, MySQL deployment on Azure VM, custom MySQL engine configuration and tuning, application connectivity verification.

---

### Containers

Provisioned and managed Azure container infrastructure across registry, orchestration, and application delivery layers. Deployed Azure Container Registry instances for private, access-controlled container image storage and versioning, integrated ACR with Azure VM workloads for streamlined, authenticated image pull operations, and synchronised container image sets using Azure CLI for cross-environment consistency. Deployed containerised application workloads on Azure VMs and hosted static websites using containerised Nginx servers on Azure infrastructure for lightweight, portable content delivery. Provisioned and administered Azure Kubernetes Service clusters including managed node pool configuration, Kubernetes Deployment and Service object management, workload scaling, and full cluster lifecycle operations covering upgrades and routine maintenance.

**Key implementations:** ACR provisioning and access control configuration, ACR and VM integration, container image synchronisation via CLI, containerised application deployment on Azure VMs, static website hosting with containers, AKS cluster provisioning, AKS managed node pool configuration, Kubernetes workload deployment and horizontal scaling on AKS.

---

### Integration

Designed and operated Azure service integration architectures for application delivery, event streaming, and data pipeline workloads. Deployed and managed Azure Web Applications with full lifecycle operations including runtime configuration, deployment management, and horizontal scaling to handle variable workload demand. Provisioned Azure Event Hub namespaces and event hubs integrated with VM-based producer workloads for high-throughput, low-latency event ingestion and real-time stream processing at scale. Architected EventHub-to-Blob Storage capture pipelines for durable, queryable event archival, ensuring zero data loss under peak ingestion volumes and enabling downstream analytics and compliance audit workflows.

**Key implementations:** Azure Web Application provisioning, runtime configuration and scaling policies, Event Hub namespace and hub provisioning, VM-to-Event Hub producer integration, high-throughput event stream processing, EventHub-to-Blob Storage capture pipeline configuration, durable event archival for analytics and audit.

---

### Automation

Provisioned Azure infrastructure declaratively using ARM Templates for consistent, repeatable, and auditable resource deployments across environments. Authored ARM Templates to define VM configurations, Virtual Network topologies, storage account specifications, NSG rules, and access control assignments as version-controlled infrastructure definitions. Applied CLI-driven user data automation for repeatable, zero-touch VM initialisation, eliminating manual post-deployment configuration steps and enforcing environment consistency. Integrated automation workflows with Azure CLI scripting for bulk operations including container synchronisation, data migration, and resource lifecycle management.

**Key implementations:** ARM Template authoring for VMs, VNets, storage accounts, and NSGs, ARM Template-based declarative stack deployment, CLI-driven user data configuration for VM initialisation, Azure CLI scripting for bulk resource operations, environment consistency enforcement through version-controlled IaC.

---

## Azure Services Coverage

```
Compute              Azure Virtual Machines, Managed Disks, Public IP,
                     NIC, VM Right-Sizing, User Data Automation, Managed Identities
Networking           Virtual Networks, Subnets, VNet Peering, NSG,
                     Application Gateway (SSL, Path-Based Routing),
                     Public and Private VNet Architectures
Security             Azure Key Vault, NSG Rule Authoring, RBAC,
                     Managed Identities, Credential-Free Service Authentication
Storage              Azure Blob Storage (Private and Public), Azure Table Storage,
                     Storage Lifecycle Policies, CLI Data Migration, Backup and Recovery
Databases            Azure SQL Database, MySQL on Azure VM,
                     Schema Migration, Firewall and Connection Management
Containers           Azure Container Registry, Azure Kubernetes Service,
                     Containerised VM Workloads, Static Website Containers
Integration          Azure Web Applications, Azure Event Hub,
                     EventHub-to-Blob Capture Pipelines, High-Throughput Event Streaming
Automation           ARM Templates, Azure CLI Scripting, User Data Automation,
                     Declarative Infrastructure Provisioning
```

---

## Related Repositories

| Repository | Description |
|---|---|
| [Cloud and DevOps Engineering](../README.md) | Full portfolio README covering DevOps, AWS, and Azure |
| [AWS Cloud Engineering](../aws/README.md) | 50+ AWS implementations across compute, networking, storage, containers, and more |
| [DevOps Engineering](../devops/README.md) | 100+ implementations across Linux, Docker, Kubernetes, Jenkins, Ansible, and Terraform |

---

<div align="center">

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/arinze-edeh)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/arinze-edeh)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white)](mailto:edeharinze389@gmail.com)

<sub>50+ production-grade Azure implementations across 8 service domains.</sub>

</div>

