# DevOps Engineering

<div align="center">

**Production-grade DevOps engineering across Linux administration, containerization, Kubernetes orchestration, CI/CD pipeline architecture, configuration management, infrastructure-as-code, networking, and scripting automation.**

[![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](./linux-administration)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](./docker)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](./kubernetes)
[![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)](./ci-cd)
[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat-square&logo=ansible&logoColor=white)](./configuration-management)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)](./infrastructure-as-code)
[![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white)](./git-version-control)
[![Bash](https://img.shields.io/badge/Bash-4EAA25?style=flat-square&logo=gnubash&logoColor=white)](./scripting-automation)

</div>

---

## Overview

This directory contains **100+ production-grade DevOps engineering implementations** organised across nine core domains. Each implementation addresses a specific infrastructure, automation, security, or operational challenge modelled on real-world production environments. The work spans the complete DevOps engineering lifecycle: from Linux system hardening and network security through container platform operations, Kubernetes cluster administration, multi-stage CI/CD pipeline architecture, Ansible configuration management, Terraform infrastructure provisioning, and Bash scripting automation.

Every implementation is fully documented with configuration details, architectural decisions, and validation outcomes.

---

## Repository Structure

```
devops/
├── ci-cd/                     # Jenkins pipeline architecture and CI/CD automation
├── configuration-management/  # Ansible playbooks, modules, and automation workflows
├── docker/                    # Dockerfile authoring, container networking, and orchestration
├── git-version-control/       # Git workflows, branching strategies, and repository operations
├── infrastructure-as-code/    # Terraform AWS infrastructure provisioning
├── kubernetes/                # Cluster operations, workload management, and incident resolution
├── linux-administration/      # Server hardening, services, storage, and system operations
├── networking-security/       # IPtables, SSL/TLS, Nginx, SSH hardening, and firewall rules
├── scripting-automation/      # Bash scripting for operational task automation
└── README.md
```

---

## Domain Implementations

### Linux Administration

Engineered and hardened production Linux server environments with a focus on security posture, user lifecycle governance, and operational reliability. Implemented non-interactive user provisioning with time-bounded expiry controls for temporary access use cases, enforced SSH key-based authentication with explicit root access lockdown, and scheduled automated operational tasks via cron for recurring maintenance workflows. Deployed and administered full-stack web infrastructure including LAMP stack configurations, Tomcat application servers, Nginx reverse proxy deployments, and PostgreSQL database servers. Resolved live process-level and network service incidents by identifying root causes through system monitoring utilities, applying targeted process management operations, and restoring service availability. Diagnosed and resolved a MariaDB service failure by analysing system service status output, inspecting error logs, isolating the fault to a configuration-level issue, and validating database connectivity post-recovery.

**Key implementations:** Non-interactive user provisioning with expiry, SSH key enforcement and root lockdown, cron-based task scheduling, LAMP stack deployment, Tomcat server configuration, PostgreSQL administration, live process and network service incident resolution, MariaDB fault diagnosis and recovery.

---

### Networking and Security

Designed and enforced network security controls and secure application delivery configurations across production Linux environments. Authored IPtables firewall ruleset configurations defining explicit inbound and outbound traffic policies, installed and configured Nginx as a Layer 7 load balancer for traffic distribution across backend application servers, and implemented SSL/TLS certificate provisioning and termination on Nginx for encrypted client connections. Configured PHP-FPM integration with Nginx via Unix socket for high-performance application processing, and enforced SSH access hardening including key-based authentication requirements, root login restrictions, and connection rate controls. Applied SELinux mandatory access control policies to enforce process-level confinement and reduce the attack surface on hardened server environments.

**Key implementations:** IPtables firewall rule authoring, inbound and outbound traffic policy enforcement, Nginx Layer 7 load balancer configuration, SSL/TLS certificate provisioning and Nginx termination, Nginx and PHP-FPM Unix socket integration, SSH hardening and access control, SELinux mandatory access control policy application.

---

### Git and Version Control

Designed and operated Git-based source control workflows aligned to trunk-based development and multi-branch release strategies used in enterprise engineering teams. Provisioned centralised Git repositories on dedicated storage servers, managed remote origin and upstream configurations, and implemented Git hooks to enforce pre-commit validation and pre-push policy checks across teams. Executed advanced version control operations including interactive rebase for commit history cleanup prior to merge, cherry-pick for targeted commit promotion across branches, hard reset for repository state restoration, and systematic merge conflict resolution in active parallel development workflows. Managed the full pull request lifecycle including branch protection enforcement, code review workflows, approval gates, and merge operations.

**Key implementations:** Centralised repository provisioning on storage servers, remote origin and upstream management, Git hook authoring for pre-commit and pre-push automation, interactive rebase, cherry-pick, hard reset, merge conflict resolution, pull request lifecycle management, branch protection enforcement, repository forking workflows.

---

### Docker

Designed and operated Docker container infrastructure across the full lifecycle from image authoring through multi-service production orchestration. Authored optimised Dockerfiles applying multi-stage build patterns and layer caching strategies to reduce image size and accelerate build pipelines. Built custom Docker bridge networks for inter-service isolation and controlled traffic flows between containers, deployed multi-container application stacks using Docker Compose with defined service dependencies, health checks, and volume mounts. Diagnosed and resolved Dockerfile build failures and runtime container instability by applying systematic layer-by-layer build output inspection to isolate failing instructions and correct base image incompatibilities. Performed live container introspection and runtime debugging via Docker EXEC, managed container-to-host file transfer operations, and provisioned containerised workloads for Python application services and Nginx web servers.

**Key implementations:** Optimised Dockerfile authoring with multi-stage builds and layer caching, custom Docker bridge network provisioning, Docker Compose multi-service orchestration, host-to-container port binding, container EXEC debugging, container-to-host file operations, Dockerfile defect diagnosis and resolution, Python and Nginx containerised application deployments.

---

### Kubernetes

Administered production-grade Kubernetes clusters prioritising workload reliability, resource governance, and operational security across self-managed, Amazon EKS, and Azure AKS environments. Deployed and managed Pods, Deployments, and multi-container workloads with explicit CPU and memory resource limits and requests to enforce scheduling boundaries and prevent resource contention. Executed zero-downtime rolling updates across Deployment replicas and performed version rollbacks to restore previously stable release states under incident conditions. Architected persistent storage solutions using PersistentVolumes and PersistentVolumeClaims with defined access modes and reclaim policies for stateful workloads. Implemented shared volume patterns for data exchange between co-located containers, deployed sidecar containers for log shipping and proxy workloads, and sequenced application startup dependencies using init containers. Diagnosed and resolved a live VolumeMounts misconfiguration where containers were failing to start due to incorrect mount path definitions and mismatched PersistentVolumeClaim bindings, restoring workload availability through targeted manifest correction. Delivered complete application platform deployments including Grafana observability dashboards, Redis caching layers, MySQL relational databases, and multi-tier web applications.

**Key implementations:** Pod and Deployment provisioning, CPU and memory resource limits and requests, rolling updates and version rollbacks, PersistentVolume and PVC configuration, shared volume patterns, sidecar and init container design, Kubernetes Secrets and environment variable injection, live VolumeMounts incident diagnosis and resolution, Grafana, Redis, MySQL, and multi-tier application deployments on Kubernetes.

---

### CI/CD

Architected and operated Jenkins-based CI/CD infrastructure from initial server provisioning through enterprise-grade pipeline delivery at distributed scale. Configured Jenkins with role-based access control (RBAC) to enforce least-privilege access across engineering teams, managed plugin lifecycle operations, and provisioned distributed build capacity using agent node architectures to parallelise pipeline execution across environments. Designed declarative multistage pipelines with conditional stage execution logic, parameterised build inputs for environment-specific deployment targeting, and scheduled triggers for recurring operational and maintenance jobs. Engineered automated database backup pipelines, application deployment pipelines with environment promotion gates, and chained build workflows to enforce strict deployment sequencing across development, staging, and production environments. Applied project-level security configurations to isolate pipeline credentials and restrict cross-project access.

**Key implementations:** Jenkins server provisioning and plugin management, RBAC configuration, distributed build agent node setup, declarative multistage pipeline authoring, conditional pipeline execution, parameterised builds, scheduled job configuration, database backup pipeline, application deployment pipeline, chained build workflows, environment promotion gates, project-level security isolation.

---

### Configuration Management

Built and maintained Ansible-based configuration management and application deployment automation across multi-server environments. Constructed structured inventory files for environment-based host segmentation and authored idempotent playbooks covering package installation, service start and restart lifecycle management, file and directory provisioning, and ACL rule enforcement. Applied the Blockinfile and Lineinfile modules for precise, targeted configuration file modifications without full file replacement, and implemented Jinja2 templating for environment-aware dynamic configuration rendering across server fleets. Introduced conditional task execution logic for platform-specific and state-conditional automation paths, validated infrastructure connectivity and configuration state using ad hoc commands and the Ping module, and troubleshot and corrected failing playbook configurations in live environments.

**Key implementations:** Structured Ansible inventory authoring, idempotent playbook development, package installation and service lifecycle management, file and directory provisioning, ACL enforcement, Blockinfile and Lineinfile module usage, Jinja2 template-based dynamic configuration rendering, conditional task execution, ad hoc command execution, Ping module connectivity validation, playbook troubleshooting and defect resolution.

---

### Infrastructure as Code

Authored and managed Terraform infrastructure-as-code configurations for AWS environments with a focus on security, repeatability, and operational auditability. Provisioned VPCs with public and private subnet architectures and associated route table configurations, security groups with least-privilege ingress and egress rules scoped to specific ports and CIDR ranges, and EC2 instances with IAM instance profile bindings for service-level permissions. Defined and attached IAM policies with fine-grained action and resource scopes for DynamoDB access, deployed EC2 workloads into private subnets with controlled NAT-based egress routing, and configured CloudWatch metric alarms with threshold definitions and notification actions for automated infrastructure health monitoring.

**Key implementations:** Terraform VPC and subnet provisioning, route table and internet gateway configuration, security group rule authoring, EC2 provisioning with IAM instance profile, DynamoDB-scoped IAM policy creation and attachment, private subnet EC2 deployment with NAT egress routing, CloudWatch alarm configuration with notification actions.

---

### Scripting and Automation

Authored Bash scripts to automate recurring operational tasks, reduce manual intervention in infrastructure workflows, and enforce consistency across server environments. Developed scripts for user provisioning workflows, system configuration tasks, service health validation, and operational data collection. Applied scripting patterns including conditional logic, loop constructs, error handling, and parameterised inputs to produce reliable, reusable automation suitable for cron-based scheduling and pipeline integration.

**Key implementations:** Bash scripting for user provisioning automation, service health validation scripts, system configuration automation, parameterised script design, error handling and exit code management, cron-compatible operational scripts, pipeline-integrated automation workflows.

---

## DevOps Engineering Coverage

```
Linux Administration       User lifecycle management, SSH hardening, cron automation,
                           LAMP and Nginx stack deployment, PostgreSQL, process and
                           service incident resolution, MariaDB fault diagnosis

Networking and Security    IPtables firewall rules, Nginx LBR, SSL/TLS termination,
                           PHP-FPM Unix socket, SSH access controls, SELinux MAC

Git and Version Control    Trunk-based workflows, Git hooks, interactive rebase,
                           cherry-pick, hard reset, conflict resolution, PR lifecycle

Docker                     Multi-stage Dockerfiles, Docker Compose orchestration,
                           custom bridge networks, runtime debugging, EXEC operations

Kubernetes                 Self-managed, EKS, AKS cluster operations, rolling updates,
                           rollbacks, PVs, PVCs, sidecars, init containers, Secrets,
                           live VolumeMounts incident resolution

CI/CD                      Jenkins multistage pipelines, RBAC, distributed agents,
                           conditional logic, parameterised builds, chained deployments,
                           environment promotion gates

Configuration Management   Ansible playbooks, Jinja2 templating, Blockinfile,
                           Lineinfile, ACL enforcement, conditional tasks, ad hoc ops

Infrastructure as Code     Terraform VPC, subnets, EC2, IAM, DynamoDB, CloudWatch,
                           private subnet deployments, NAT egress routing

Scripting and Automation   Bash scripting, parameterised inputs, error handling,
                           cron-compatible automation, pipeline-integrated scripts
```


---

## Related Repositories

| Repository | Description |
|---|---|
| [Cloud and DevOps Engineering](../README.md) | Full portfolio README covering DevOps, AWS, and Azure |
| [AWS Cloud Engineering](../aws/README.md) | 50+ AWS implementations across compute, networking, storage, containers, and more |
| [Azure Cloud Engineering](../azure/README.md) | 50+ Azure implementations across VMs, networking, storage, containers, and databases |

---

<div align="center">

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/arinze-edeh)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/arinze-edeh)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white)](mailto:edeharinze389@gmail.com)

<sub>100+ production-grade DevOps implementations across 9 engineering domains.</sub>

</div>


