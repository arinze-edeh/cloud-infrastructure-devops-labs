# Cloud and DevOps Engineering Portfolio

<div align="center">

**Production-grade infrastructure engineering across Linux, containers, orchestration, CI/CD, IaC, and multi-cloud platforms.**

[![AWS](https://img.shields.io/badge/Amazon_AWS-FF9900?style=flat-square&logo=amazonaws&logoColor=white)](./aws)
[![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=flat-square&logo=microsoftazure&logoColor=white)](./azure)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](./devops)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)](./devops)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](./devops)
[![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)](./devops)
[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat-square&logo=ansible&logoColor=white)](./devops)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](./devops)

</div>

---

## Portfolio Highlights

> A snapshot of the engineering depth across this repository. Every item below represents a fully implemented, production-aligned solution.

| Highlight | Detail |
|---|---|
| **Multi-Cloud Coverage** | Production implementations across both AWS and Azure covering compute, networking, IAM, storage, containers, databases, and serverless |
| **End-to-End CI/CD Architecture** | Jenkins multistage declarative pipelines with RBAC, distributed agent nodes, conditional logic, parameterized builds, chained deployments, and automated backup jobs |
| **Kubernetes at Depth** | Full cluster operations across self-managed, EKS, and AKS including rolling updates, rollbacks, persistent volumes, sidecar patterns, init containers, Secrets management, and live incident resolution |
| **Infrastructure as Code** | Terraform-provisioned AWS infrastructure covering VPC, subnets, EC2, IAM, DynamoDB policies, and CloudWatch alarms; Ansible automation across 10+ modules for multi-server configuration management |
| **Container Platform Engineering** | Docker image authoring, multi-service Docker Compose orchestration, custom bridge networks, runtime debugging, and full application containerization pipelines |
| **Security Engineering Depth** | IAM least-privilege policy design, KMS encryption, Azure Key Vault, SELinux mandatory access control, IPtables firewall rules, NSG authoring, SSL/TLS termination, and SSH hardening |
| **Networking Architecture** | VPC and VNet design with public and private subnet segmentation, VPC Peering, VNet Peering, NAT Instances, NAT Gateways, Application Load Balancers, and Application Gateways |
| **Troubleshooting and Incident Response** | Diagnosed and resolved live incidents across Kubernetes VolumeMounts, Dockerfile build failures, MariaDB failures, process and network service outages, and public VNet connectivity faults |
| **Full Linux Engineering Stack** | Server hardening, SSH authentication enforcement, user lifecycle management, SELinux, IPtables, cron automation, LAMP and Nginx stack deployment, PHP-FPM Unix socket configuration, and database administration |
| **Dual Cloud Kubernetes** | Kubernetes cluster operations on both Amazon EKS and Azure AKS including node pool management, workload deployment, autoscaling, and cluster lifecycle administration |

---

## Overview

This repository contains **200+ production-grade engineering implementations** organised across three domains: core DevOps engineering, AWS cloud architecture, and Azure cloud architecture. Each implementation addresses a specific infrastructure, automation, security, or architectural challenge modelled on real-world production environments.

The portfolio spans the complete engineering lifecycle: from Linux system hardening and Git workflow design, through container platform operations and Kubernetes cluster administration, to multi-stage CI/CD pipeline architecture, infrastructure-as-code provisioning, and cloud-native service integration across AWS and Azure.

**Target Roles:** DevOps Engineer, Cloud Infrastructure Engineer, Site Reliability Engineer, Platform Engineer, Kubernetes Engineer, CI/CD Engineer, Automation Engineer, Cloud Security Engineer, Build and Release Engineer, Infrastructure Engineer.

---

## Repository Structure

```
.
├── aws/          # 50+ AWS cloud architecture and service implementations
├── azure/        # 50+ Azure cloud architecture and service implementations
├── devops/       # 100+ DevOps engineering implementations
└── README.md
```

---

## Technology Stack

| Domain | Technologies |
|---|---|
| **Operating Systems** | Linux (RHEL, Ubuntu), Bash scripting, SELinux, IPtables, cron |
| **Containerization** | Docker, Docker Compose, Dockerfile authoring, container networking, image registries |
| **Orchestration** | Kubernetes (self-managed, EKS, AKS), PersistentVolumes, RBAC, Secrets, ConfigMaps |
| **CI/CD** | Jenkins (declarative and scripted pipelines, distributed builds), Git, GitHub |
| **Infrastructure as Code** | Terraform, Ansible (10+ modules), ARM Templates, AWS CloudFormation |
| **Cloud Platforms** | AWS (20+ services), Microsoft Azure (20+ services) |
| **Web and App Servers** | Nginx (reverse proxy, LBR, SSL), Apache, Tomcat, PHP-FPM |
| **Databases** | PostgreSQL, MariaDB, MySQL, Amazon RDS, DynamoDB, Azure SQL |
| **Observability** | Grafana, AWS CloudWatch (alarms, metrics, logs), container metrics |
| **Security** | IAM, KMS, Azure Key Vault, NSG, SELinux, IPtables, SSL/TLS, SSH hardening, RBAC |
| **Networking** | VPC, VNet, subnets, VPC Peering, VNet Peering, NAT, ALB, Application Gateway |
| **Messaging and Events** | Amazon SQS, Amazon SNS, Azure Event Hub |
| **Serverless** | AWS Lambda, S3 event triggers, CloudFormation stacks |

---

## DevOps Engineering

### Linux Systems Engineering

Engineered and hardened production Linux server environments with a focus on security posture, access control, and operational reliability. Implemented non-interactive user provisioning with time-bounded expiry controls for temporary access use cases, enforced SSH key-based authentication with explicit root access lockdown, and applied SELinux mandatory access control policies to enforce process-level confinement. Authored IPtables firewall ruleset configurations for inbound and outbound traffic enforcement, scheduled automated operational tasks via cron, and deployed full-stack web infrastructure including LAMP, Nginx with SSL/TLS termination, Nginx as a Layer 7 load balancer, PHP-FPM via Unix socket, and Tomcat application servers. Administered PostgreSQL database servers and resolved live network service and process-level incidents in running environments.

### Git and Source Control Engineering

Designed and operated Git-based source control workflows aligned to trunk-based development and multi-branch release strategies. Provisioned centralised Git repositories on dedicated storage servers, managed remote origin and upstream configurations, enforced repository-level access policies, and implemented Git hooks to automate pre-commit validation and pre-push enforcement. Executed advanced version control operations including interactive rebase for commit history cleanup, cherry-pick for targeted commit promotion across branches, hard reset for repository state restoration, and merge conflict resolution in active parallel development workflows. Owned the full pull request lifecycle including branch protection, code review workflows, approval gates, and merge operations.

### Container Platform Engineering

Designed and operated container infrastructure from image authoring through multi-service production orchestration. Authored optimised Dockerfiles applying multi-stage build patterns and layer caching strategies to minimise image size and build time. Built custom Docker bridge networks for inter-service isolation, deployed multi-container application stacks using Docker Compose with defined service dependencies and volume mounts, and mapped host-to-container port bindings for controlled external access. Performed live container introspection and runtime debugging via Docker EXEC, managed container-to-host file operations, and provisioned containerised workloads for Python application services and Nginx web servers.

### Kubernetes Cluster Operations

Administered production-grade Kubernetes clusters prioritising workload reliability, resource governance, and operational security. Deployed and managed Pods, Deployments, and multi-container workloads with explicit CPU and memory resource limits and requests to enforce scheduling boundaries and prevent noisy-neighbour resource contention. Executed zero-downtime rolling updates across Deployment replicas and performed version rollbacks to restore previously stable release states. Architected persistent storage solutions using PersistentVolumes and PersistentVolumeClaims with defined access modes and reclaim policies. Implemented shared volume patterns for data exchange between co-located containers, deployed sidecar containers for log shipping and proxy workloads, and sequenced application startup dependencies using init containers. Managed runtime configuration injection via Kubernetes Secrets and environment variables, and delivered complete application platform deployments including Grafana observability dashboards, Redis caching layers, MySQL relational databases, and multi-tier web applications.

### Jenkins CI/CD Pipeline Engineering

Architected and operated Jenkins CI/CD infrastructure from initial server provisioning through enterprise-grade pipeline delivery at scale. Configured Jenkins with role-based access control (RBAC) to enforce least-privilege access across teams, managed plugin lifecycle, and provisioned distributed build capacity using agent node architectures to parallelise pipeline execution. Designed declarative multistage pipelines with conditional stage execution, parameterised build inputs for environment-specific deployments, and scheduled triggers for recurring operational jobs. Engineered automated database backup pipelines, application deployment pipelines with environment promotion gates, and chained build workflows to enforce strict deployment sequencing across environments. Applied project-level security configurations to isolate pipeline credentials and restrict cross-project access.

### Ansible Configuration Management

Built and maintained Ansible-based configuration management and application deployment automation across multi-server environments. Constructed structured inventory files for environment-based host segmentation and authored idempotent playbooks covering package installation, service start and restart lifecycle management, file and directory provisioning, and ACL rule enforcement. Applied Blockinfile and Lineinfile modules for precise, targeted configuration file modifications without full file replacement. Implemented Jinja2 templating for environment-aware dynamic configuration rendering and introduced conditional task execution logic for platform-specific and state-specific automation paths. Validated infrastructure state and connectivity using ad hoc Ansible commands and the Ping module across managed node fleets.

### Terraform Infrastructure Provisioning

Authored and managed Terraform infrastructure-as-code configurations for AWS environments with a focus on security, repeatability, and auditability. Provisioned VPCs with public and private subnet architectures and associated routing table configurations, security groups with least-privilege ingress and egress rules scoped to specific ports and CIDR ranges, and EC2 instances with IAM instance profile bindings for service-level permissions. Defined and attached IAM policies with fine-grained action and resource scopes for DynamoDB access, deployed EC2 workloads into private subnets with controlled egress routing, and configured CloudWatch metric alarms for automated infrastructure health monitoring and alerting.

---

## Troubleshooting and Incident Response

This section highlights diagnostic and resolution work across infrastructure layers, demonstrating the systematic fault-isolation and remediation capability expected of DevOps, SRE, and Platform Engineering roles.

### Kubernetes Incident Resolution

Diagnosed and resolved a live VolumeMounts misconfiguration in a running Kubernetes Deployment where containers were failing to start due to incorrect mount path definitions and mismatched PersistentVolumeClaim bindings. Isolated the fault by inspecting Pod describe output and container logs, identified the root cause in the Deployment manifest, applied a corrected configuration, and validated container readiness post-remediation. Separately diagnosed and resolved a Python application deployment failure on a running Kubernetes cluster by tracing runtime errors through container logs and correcting application environment variable injection.

### Dockerfile and Container Build Failures

Identified and resolved Dockerfile defects causing image build failures and runtime container instability. Applied systematic layer-by-layer inspection of build output to isolate failing instructions, corrected base image incompatibilities, and validated fixed images through full build and run cycles.

### MariaDB Database Troubleshooting

Diagnosed a MariaDB service failure in a production Linux environment by analysing system service status, inspecting error logs, and isolating the fault to a configuration-level issue. Restored service availability through targeted configuration correction and service restart, then validated database connectivity post-recovery.

### Linux Process and Network Service Incidents

Resolved live Linux process and network service incidents by identifying runaway or failed processes using system monitoring utilities, applying targeted process management operations, and restoring service availability. Diagnosed network service binding and connectivity faults through socket inspection and service configuration review.

### AWS EC2 Internet Accessibility

Diagnosed and resolved an internet accessibility failure for an EC2-hosted application by systematically inspecting the VPC route table, internet gateway attachment, security group ingress rules, and instance public IP association to isolate and correct the misconfigured component blocking inbound traffic.

### Azure Public VNet Connectivity

Troubleshot a public Azure Virtual Network configuration where VM internet connectivity was failing. Traced the fault through VNet configuration, subnet route tables, NSG inbound and outbound rules, and public IP association to identify and resolve the blocking misconfiguration.

---

## AWS Cloud Engineering

### Identity, Access, and Security

Designed and implemented AWS IAM architectures following the principle of least privilege across users, groups, roles, and policies. Authored scoped IAM policies for EC2 read-only console access and DynamoDB service operations, attached policies to IAM users and EC2 instance profiles for service-to-service authentication, and configured AWS KMS for data-at-rest encryption. Managed EC2 key pairs, security group ingress and egress rules, and SSH hardening configurations for secure remote access.

### Compute and Networking Architecture

Engineered AWS compute and networking infrastructure for scalable, highly available, and securely segmented application hosting. Designed VPCs with public and private subnet architectures, internet gateway attachment, route table configurations, and VPC Peering for private cross-VPC communication without traffic traversing the public internet. Deployed and managed EC2 instances with Elastic IP addressing for stable public endpoints, Elastic Network Interface (ENI) attachment for secondary network connectivity, EBS volume provisioning and online resizing, GP3 volume configuration for cost-optimised storage performance, and instance stop and termination protection for critical workloads. Configured NAT Instances and NAT Gateways to provide secure outbound internet access for private subnet resources while blocking unsolicited inbound connections. Deployed Application Load Balancers for HTTP and HTTPS traffic distribution across EC2 target groups with health check-based failover.

### Storage and Data Services

Implemented AWS storage architectures covering S3 bucket versioning for object-level change history and recovery, bucket policy management for access control, and S3 static website hosting for frontend application delivery. Executed cross-bucket data migration using AWS CLI scripting, managed EBS volume snapshots for point-in-time recovery, and provisioned private ECR repositories for secure container image storage and versioning. Configured Amazon RDS instances within private subnets for application database connectivity with restricted public access, performed RDS snapshot backup and restore operations for data recovery validation, and designed DynamoDB NoSQL tables with partition key schemas and IAM-scoped access policies.

### Serverless and Event-Driven Architecture

Developed and deployed AWS Lambda functions using both the AWS Console and CLI, implementing event-driven compute workflows triggered by S3 object creation events for automated data processing pipelines. Integrated Amazon SQS queues and SNS topics to build decoupled, resilient asynchronous messaging architectures with guaranteed delivery and fan-out notification patterns. Automated repeatable infrastructure provisioning using AWS CloudFormation templates for version-controlled, auditable stack deployments.

### Container and Kubernetes Platforms

Deployed and managed containerised workloads on Amazon ECS including task definition authoring, service configuration, and container lifecycle management. Provisioned and operated Amazon EKS clusters for production Kubernetes workloads, configured managed node groups, and managed application Deployment and Service objects within the cluster. Implemented EC2 Auto Scaling groups with target tracking and step scaling policies to maintain application availability and optimise compute cost under variable traffic load. Established centralised audit and flow logging with cross-VPC visibility using VPC Peering to support security and compliance requirements.

---

## Azure Cloud Engineering

### Virtual Machine and Network Administration

Provisioned and managed Azure Virtual Machines using both the Azure Console and Azure CLI, covering initial deployment, VM size changes for right-sizing, and resource tag management for cost allocation and governance. Designed Azure Virtual Networks with custom IPv4 CIDR addressing, subnet segmentation for workload isolation, and Network Security Group rules for granular inbound and outbound traffic control. Configured dynamic and static Public IP addresses, attached and detached Network Interface Cards for multi-homed VM configurations, provisioned and expanded Managed Disks for storage growth, and secured remote access via SSH key pair management. Deployed and managed Azure infrastructure declaratively using ARM Templates for consistent, repeatable provisioning.

### Storage and Data Management

Designed and operated Azure Blob Storage solutions including private container provisioning for internal application data and public container configuration for externally accessible assets. Executed bulk data migration into Blob Storage containers using Azure CLI scripting, enforced access tier transitions from public to private to reduce attack surface, and configured storage lifecycle management policies to automate data tiering to cool and archive tiers based on access patterns. Managed Azure Table Storage for structured NoSQL workloads requiring low-latency key-value access, performed blob container backup and controlled deletion operations, and integrated storage accounts with virtual machine compute workloads for persistent application data.

### Networking and Connectivity

Engineered Azure networking topologies for both internet-facing and air-gapped private workload deployments. Deployed VMs into public VNets with internet gateway routing for externally accessible applications and into private VNets with restricted outbound paths for sensitive workloads. Configured VNet Peering for private, low-latency cross-VNet communication without traffic traversing the public internet. Enabled controlled outbound internet access for private VM workloads and integrated Azure Application Gateway as a Layer 7 load balancer with SSL termination, path-based routing, and backend VM pool configuration.

### Container and Application Platforms

Provisioned Azure Container Registry instances for private container image storage with access-controlled pull operations, integrated ACR with Azure VM workloads for seamless image retrieval, and synchronised container image sets across registries using Azure CLI. Deployed containerised application workloads on Azure VMs and delivered static website hosting using containerised web servers on Azure infrastructure. Provisioned and administered Azure Kubernetes Service clusters including managed node pool configuration, application workload deployment and scaling, and full cluster lifecycle management covering upgrades and maintenance operations.

### Database and Application Services

Provisioned Azure SQL Database instances with configured firewall rules and connection string management for secure application backend access. Executed SQL database schema migration and environment setup for application onboarding workflows. Deployed MySQL database servers on Azure VMs for workloads requiring custom engine configurations outside of managed service constraints. Managed Azure Web Application deployments including runtime configuration, scaling rules, and application lifecycle operations. Designed and configured Azure Event Hub namespaces and event hubs for high-throughput ingestion from VM-based producers, and built EventHub-to-Blob Storage capture pipelines for durable, queryable event archival.

### Security and Secrets Management

Centralised application secret, connection string, and cryptographic key management using Azure Key Vault with RBAC-enforced access policies to eliminate hardcoded credentials from application configurations. Authored NSG rule sets with granular port-level ingress and egress controls to enforce network security boundaries at the subnet and NIC levels. Automated VM initialisation using user data scripts for consistent, repeatable instance configuration at launch, and provisioned VMs with scoped managed identities for credential-free service-to-service authentication against Azure resources.

---

## Core Engineering Competencies

```
Multi-Cloud Architecture        AWS and Azure production infrastructure design and operations
Infrastructure as Code          Terraform, Ansible, ARM Templates, AWS CloudFormation
Container Orchestration         Kubernetes (EKS, AKS, self-managed), Docker, Docker Compose, ECS
CI/CD Pipeline Engineering      Jenkins multistage pipelines, distributed builds, RBAC, chained deployments
Linux Systems Engineering       Hardening, networking, storage, process management, service administration
Security Engineering            IAM, KMS, Key Vault, SELinux, IPtables, NSG, RBAC, SSL/TLS, SSH
Incident Response               Kubernetes, container, database, network, and cloud connectivity diagnosis
Database Operations             PostgreSQL, MariaDB, MySQL, Amazon RDS, DynamoDB, Azure SQL
Observability and Alerting      Grafana, CloudWatch alarms, metrics collection, log inspection
Networking                      VPC, VNet, subnets, peering, NAT, load balancing, routing tables
Automation and Scripting        Bash, Ansible playbooks, Jinja2 templating, Git hooks, AWS CLI, Azure CLI
Serverless and Events           AWS Lambda, S3 triggers, SQS, SNS, Azure Event Hub
```

---

## Target Roles

| Role | Primary Skills Demonstrated |
|---|---|
| **DevOps Engineer** | Linux, Git, Docker, Kubernetes, Jenkins, Ansible, Terraform, full SDLC automation |
| **Cloud Infrastructure Engineer** | AWS and Azure architecture, VPC/VNet design, IAM, compute, storage, networking |
| **Site Reliability Engineer** | Kubernetes operations, rolling updates, rollbacks, incident response, observability |
| **Platform Engineer** | Kubernetes cluster admin, CI/CD pipeline design, IaC, internal tooling |
| **Kubernetes Engineer** | EKS, AKS, self-managed clusters, persistent volumes, RBAC, sidecar, init containers |
| **CI/CD Engineer** | Jenkins multistage pipelines, distributed agents, parameterised builds, RBAC |
| **Cloud Security Engineer** | IAM, KMS, Key Vault, SELinux, NSG, IPtables, SSL/TLS, RBAC, SSH hardening |
| **Infrastructure Engineer** | Multi-cloud networking, IaC, compute provisioning, storage architecture |
| **Automation Engineer** | Ansible, Terraform, Bash, Jinja2, Git hooks, CloudFormation, ARM Templates |
| **Build and Release Engineer** | Jenkins pipelines, chained builds, deployment sequencing, environment promotion |

---

## Connect

**Actively seeking DevOps, Cloud Infrastructure, SRE, and Platform Engineering roles.**
Available for full-time positions, contract engagements, and technical interviews.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/arinze-edeh)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/arinze-edeh)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white)](mailto:edeharinze389@gmail.com)

---

<div align="center">
<sub>200+ production-style implementations across DevOps, AWS, and Azure engineering.</sub>
</div>


