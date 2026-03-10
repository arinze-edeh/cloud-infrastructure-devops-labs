# Cloud and DevOps Engineering Portfolio

<div align="center">

**Production-grade infrastructure engineering across Linux, containers, orchestration, CI/CD, and multi-cloud platforms.**

[![AWS](https://img.shields.io/badge/Amazon_AWS-FF9900?style=flat-square&logo=amazonaws&logoColor=white)](./aws)
[![Azure](https://img.shields.io/badge/Microsoft_Azure-0089D6?style=flat-square&logo=microsoftazure&logoColor=white)](./azure)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](./devops)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)](./devops)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](./devops)
[![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)](./devops)
[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat-square&logo=ansible&logoColor=white)](./devops)

</div>

---

## Overview

This repository documents **200+ production-grade engineering implementations** across three domains: core DevOps engineering, AWS cloud architecture, and Azure cloud architecture. Every implementation reflects real-world operational patterns including infrastructure provisioning, security hardening, pipeline automation, container orchestration, and multi-cloud service integration.

The work spans the full engineering lifecycle from low-level Linux system administration through enterprise-scale Kubernetes cluster operations, multi-stage CI/CD pipelines, and cloud-native architecture design on both AWS and Azure.

---

## Repository Structure

```
.
├── aws/          # AWS cloud architecture and service implementations
├── azure/        # Azure cloud architecture and service implementations
├── devops/       # Core DevOps engineering: Linux, containers, CI/CD, IaC
└── README.md
```

---

## Technology Stack

| Domain | Technologies |
|---|---|
| **Operating Systems** | Linux (RHEL, Ubuntu), Bash scripting, SELinux, IPtables |
| **Containerization** | Docker, Docker Compose, Dockerfile authoring, container networking |
| **Orchestration** | Kubernetes (self-managed, EKS, AKS), persistent volumes, RBAC |
| **CI/CD** | Jenkins (declarative and scripted pipelines), Git, GitHub |
| **Infrastructure as Code** | Terraform, Ansible, ARM Templates, AWS CloudFormation |
| **Cloud Platforms** | AWS, Microsoft Azure |
| **Web and App Servers** | Nginx, Apache, Tomcat, PHP-FPM |
| **Databases** | PostgreSQL, MariaDB, MySQL, Amazon RDS, DynamoDB, Azure SQL |
| **Observability** | Grafana, AWS CloudWatch, container-level metrics |
| **Security** | IAM, KMS, Azure Key Vault, NSG, SSL/TLS, SSH hardening, SELinux |

---

## DevOps Engineering

### Linux Systems Engineering

Engineered and hardened production Linux environments with a focus on security, reliability, and operational efficiency. Implemented non-interactive user provisioning with expiry-based access controls, enforced SSH key-based authentication with root access restrictions, and applied SELinux policies for mandatory access control. Configured IPtables firewall rules, scheduled automated tasks via cron, and resolved live process and network service incidents. Deployed and configured full-stack web infrastructure including LAMP, Nginx with SSL/TLS termination, Nginx as a Layer 7 load balancer, PHP-FPM with Unix socket integration, and Tomcat application servers. Administered PostgreSQL database servers with a focus on secure access and performance.

### Git and Source Control Engineering

Established and maintained scalable Git workflows modeled on trunk-based development and feature branch strategies. Provisioned centralized Git repositories on storage servers, managed remote configurations, enforced repository access policies, and authored Git hooks for pre-commit and pre-push automation. Executed advanced version control operations including interactive rebasing, cherry-picking targeted commits across branches, hard resets to restore repository state, and conflict resolution in parallel development workflows. Managed the full pull request lifecycle including code review, approval gates, and merge operations.

### Container Platform Engineering

Designed and operated container infrastructure across the full lifecycle from image authoring to multi-service orchestration. Authored optimized Dockerfiles following multi-stage build and layer caching best practices, diagnosed and resolved build-time and runtime image defects, and built custom Docker bridge networks for service isolation. Deployed containerized applications including Nginx web servers, Python services, and multi-tier application stacks using Docker Compose. Performed container introspection and live debugging via Docker EXEC operations, mapped host-to-container port bindings, and managed container-to-host file transfer operations.

### Kubernetes Cluster Operations

Administered production-grade Kubernetes clusters with a focus on reliability, resource efficiency, and security. Deployed and managed Pods, Deployments, and StatefulSets with explicit CPU and memory resource boundaries. Executed zero-downtime rolling updates and performed version rollbacks to restore previous deployment states. Architected persistent storage solutions using PersistentVolumes and PersistentVolumeClaims, implemented shared volume patterns for multi-container Pods, and deployed sidecar containers for logging and proxy workloads. Managed init container workflows for dependency sequencing, injected runtime configuration via Kubernetes Secrets and environment variables, and resolved live VolumeMounts misconfigurations in production deployments. Delivered full application deployments including Grafana monitoring dashboards, Redis caching clusters, MySQL databases, and multi-tier web applications.

### Jenkins CI/CD Pipeline Engineering

Architected and operated Jenkins-based CI/CD infrastructure from initial server provisioning through enterprise-grade pipeline delivery. Configured Jenkins with role-based access control (RBAC), managed plugin ecosystems, and provisioned distributed build architectures using agent nodes. Designed and implemented declarative multistage pipelines with conditional execution logic, parameterized build configurations, and scheduled job triggers. Automated operational workflows including database backup pipelines and application deployment jobs. Built chained build pipelines to enforce deployment sequencing across environments and applied project-level security policies to enforce least-privilege pipeline execution.

### Ansible Configuration Management

Authored and maintained Ansible automation for configuration management and application deployment across multi-server environments. Built structured inventories for environment segmentation and developed idempotent playbooks for package installation, service lifecycle management, file and directory provisioning, and ACL enforcement. Leveraged Blockinfile and Lineinfile modules for precise configuration file management, implemented Jinja2 templates for environment-aware configuration rendering, and applied conditional task logic for platform-specific execution paths.

### Terraform Infrastructure Provisioning

Provisioned and managed AWS infrastructure using Terraform with a focus on security, modularity, and auditability. Authored Terraform configurations to deploy VPCs with public and private subnet architectures, security groups with least-privilege ingress and egress rules, and EC2 instances with associated IAM role bindings. Defined and attached IAM policies scoped to specific AWS services including DynamoDB, deployed EC2 instances into private subnets with appropriate routing, and configured CloudWatch alarms for automated infrastructure monitoring and alerting.

---

## AWS Cloud Engineering

### Identity, Access, and Security

Designed and implemented IAM architectures following the principle of least privilege. Provisioned IAM users, groups, and roles with scoped permission policies, authored read-only and service-specific IAM policies for EC2 console access and DynamoDB operations, and configured KMS encryption for data-at-rest security. Managed EC2 key pairs, security group rules, and SSH access hardening for secure instance access.

### Compute and Networking Architecture

Engineered AWS compute and network infrastructure for scalable, highly available application hosting. Designed VPCs with public and private subnet segmentation, internet gateway and route table configurations, and VPC Peering for cross-VPC private connectivity. Deployed and managed EC2 instances with Elastic IP addressing, Elastic Network Interface (ENI) attachment, EBS volume provisioning and resizing, GP3 volume configuration, and stop and termination protection. Configured NAT Instances and NAT Gateways to enable secure outbound internet access for private subnet resources. Deployed and configured Application Load Balancers (ALB) for HTTP/HTTPS traffic distribution across EC2 targets.

### Storage and Data Services

Implemented AWS storage solutions covering S3 bucket versioning, bucket policy management, and static website hosting. Executed cross-bucket data migration using AWS CLI, managed EBS volume snapshots for point-in-time backup and restoration, and provisioned private ECR repositories for container image management. Configured Amazon RDS instances in private subnets for secure application database access and performed RDS snapshot backup and restore operations. Designed and managed DynamoDB NoSQL tables with appropriate key schemas and access control policies.

### Serverless and Event-Driven Architecture

Developed and deployed AWS Lambda functions using both the AWS Console and CLI, implementing event-driven processing workflows triggered by S3 object events. Integrated Amazon SQS and SNS to build reliable asynchronous messaging pipelines with decoupled producers and consumers. Automated infrastructure provisioning using AWS CloudFormation for repeatable, version-controlled stack deployments.

### Container and Kubernetes Platforms

Deployed and managed containerized workloads using Amazon ECS for task and service orchestration. Provisioned and operated Amazon EKS clusters for production Kubernetes workloads, configured node groups and cluster autoscaling, and managed application deployments within EKS. Implemented Auto Scaling groups with scaling policies to maintain application availability and cost efficiency under variable load. Established centralized audit logging using VPC Flow Logs with VPC Peering for cross-account visibility.

---

## Azure Cloud Engineering

### Virtual Machine and Network Administration

Provisioned and managed Azure Virtual Machines using both the Azure Console and Azure CLI, including VM sizing, resizing, and tag management for cost allocation and governance. Designed Azure Virtual Networks (VNets) with custom IPv4 CIDR ranges, subnet segmentation, and NSG rule authoring for inbound and outbound traffic control. Configured Public IP addresses, managed Network Interface Cards (NICs), attached and expanded Managed Disks, and established secure SSH access for remote administration. Deployed infrastructure using ARM Templates for repeatable, declarative resource provisioning.

### Storage and Data Management

Implemented Azure Blob Storage solutions including private and public container provisioning, data upload and migration using the Azure CLI, and access tier transitions to enforce least-privilege data access. Managed Azure Table Storage for structured NoSQL data workloads, configured storage lifecycle management policies to automate data tiering and retention, and performed blob container backup and deletion operations. Integrated storage accounts with virtual machine workloads for application data persistence.

### Networking and Connectivity

Engineered Azure networking architectures for both public and private workload deployments. Deployed VMs in public and private VNet configurations, diagnosed and resolved public VNet connectivity issues, and configured VNet Peering for private cross-VNet communication. Enabled outbound internet connectivity for private VMs and integrated Azure Application Gateway for Layer 7 load balancing and SSL termination. Attached Application Gateway to VM backends and validated end-to-end request routing.

### Container and Application Platforms

Provisioned and managed Azure Container Registry (ACR) for private container image storage, integrated ACR with Azure VM workloads for streamlined image pull operations, and synchronized container images using the Azure CLI. Deployed containerized applications on Azure VMs and hosted static websites using containers on Azure infrastructure. Provisioned and managed Azure Kubernetes Service (AKS) clusters including node pool configuration, workload deployment, and cluster lifecycle management.

### Database and Application Services

Deployed and configured Azure SQL Database instances for application backend connectivity and performed SQL database migration and schema setup for application onboarding workloads. Provisioned MySQL on Azure VMs for custom database configurations and deployed Azure Web Applications with full lifecycle management including scaling and configuration. Integrated Azure Event Hub with VM-based consumers for high-throughput event streaming and configured EventHub-to-Blob Storage pipelines for durable event archival.

### Security and Secrets Management

Managed application secrets, connection strings, and cryptographic keys using Azure Key Vault with RBAC-enforced access policies. Configured NSGs with granular ingress and egress rules to enforce network-level security boundaries. Applied VM-level user data automation for consistent, repeatable instance initialization and provisioned VMs with scoped roles for secure service-to-service integration.

---

## Core Engineering Competencies

```
Multi-Cloud Architecture       AWS and Azure production infrastructure design and operations
Infrastructure as Code         Terraform, Ansible, ARM Templates, CloudFormation
Container Orchestration        Kubernetes (EKS, AKS, self-managed), Docker, ECS
CI/CD Pipeline Engineering     Jenkins declarative pipelines, multistage workflows, distributed builds
Linux Systems Engineering      Hardening, networking, storage, process and service management
Security Engineering           IAM, KMS, Key Vault, SELinux, IPtables, NSG, RBAC, SSL/TLS
Database Operations            PostgreSQL, MariaDB, MySQL, RDS, DynamoDB, Azure SQL
Observability                  Grafana, CloudWatch, alerting, metrics collection
Automation and Scripting       Bash, Ansible playbooks, Jinja2 templating, Git hooks
Networking                     VPC, VNet, subnets, peering, NAT, load balancing
```

---

## Connect

**Open to Senior DevOps Engineer, Cloud Infrastructure Engineer, and Site Reliability Engineer roles.**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/your-profile)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/your-username)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white)](mailto:your@email.com)

---

<div align="center">
<sub>Built with precision. Engineered for production.</sub>
</div>
