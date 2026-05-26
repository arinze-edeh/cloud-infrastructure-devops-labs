# DevOps Engineering

<div align="center">

**Production-style DevOps engineering across Linux administration, containerization, Kubernetes orchestration, CI/CD pipeline architecture, configuration management, infrastructure-as-code, and networking.**

[![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](./linux-administration)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](./docker)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](./kubernetes)
[![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)](./ci-cd)
[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat-square&logo=ansible&logoColor=white)](./configuration-management)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)](./infrastructure-as-code)
[![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white)](./git-version-control)
[![Bash](https://img.shields.io/badge/Bash-4EAA25?style=flat-square&logo=gnubash&logoColor=white)](./linux-administration)

</div>

---

## Overview

This directory contains 100+ production-style DevOps engineering implementations across eight core domains, all executed in the **Stratos Datacenter** environment (xFusionCorp / Nautilus Labs). The work simulates real operational conditions: live multi-node infrastructure, bastion jump-host access models, non-standard configuration constraints, no-restart requirements, and audit-ready change management.

Each subdirectory is independently navigable with full implementation documentation, covering design decisions, encountered failures, and verified outcomes.

---

## Directory Structure

```
devops/
├── ci-cd/                     # Jenkins pipeline architecture and CI/CD automation
├── configuration-management/  # Ansible playbooks, roles, and fleet automation
├── docker/                    # Dockerfile authoring, container networking, and orchestration
├── git-version-control/       # Git workflows, branching strategies, and repository operations
├── infrastructure-as-code/    # Terraform AWS provisioning and multi-tier Linux deployments
├── kubernetes/                # Cluster operations, workload management, and incident resolution
├── linux-administration/      # Server hardening, service management, and system operations
├── networking-security/       # IPtables, SSL/TLS, Nginx, SSH hardening, and firewall enforcement
└── README.md
```

---

## Domain Summaries

### [CI/CD](./ci-cd)

Jenkins-based CI/CD infrastructure built from bare OS through distributed pipeline delivery. Covers server provisioning, plugin lifecycle management, distributed agent architecture, and RBAC enforcement.

**Key implementations:** Declarative multistage pipelines with parameterized inputs and conditional stage logic; chained upstream/downstream job dependencies; scheduled database backup and log collection pipelines; Matrix Authorization Strategy RBAC at global and job scopes; SSH key-based agent registration across three AlmaLinux 9 nodes; GPG signing issue resolution during initial install.

---

### [Configuration Management](./configuration-management)

Ansible automation for infrastructure provisioning, service deployment, and access control across a RHEL-based multi-node fleet. All tasks execute from a centralized jump host with zero manual intervention on target nodes.

**Key implementations:** Idempotent playbook development for package installation, service lifecycle, and file provisioning; POSIX ACL enforcement via the `acl` module; Jinja2 dynamic templating with `{{ inventory_hostname }}` substitution; `Blockinfile` and `Lineinfile` for targeted config file mutations; RSA key-based passwordless SSH setup and inventory migration; pre-execution validation pipelines (ping, syntax check, `--check`).

---

### [Docker](./docker)

Full Docker lifecycle from Dockerfile authoring through multi-service orchestration. Covers image engineering, container networking, volume management, and fault resolution.

**Key implementations:** Multi-stage Dockerfile authoring with layer caching and `sed`-based in-place config patching; Docker Compose LAMP stack with environment-variable secrets; macvlan network provisioning with custom IPAM; `docker cp` file transfer with MD5 integrity verification; Dockerfile fault diagnosis with `/proc/net/tcp` inode resolution when `ss`/`netstat` were absent; Docker Engine installation from scratch on CentOS Stream 9.

---

### [Git and Version Control](./git-version-control)

Git operations across self-hosted Gitea, bare repository infrastructure, and shared Linux storage servers. Covers advanced version control, access-constrained workflows, and server-side automation.

**Key implementations:** `post-update` hook for automated date-stamped release tagging; `git rebase` with `add/add` conflict resolution; `git reset --hard` with force push for history cleanup; `git cherry-pick` for targeted commit promotion; full PR lifecycle on Gitea including fork provisioning, reviewer assignment, and merge governance; CVE-2022-24765 `safe.directory` handling for root-owned repositories.

---

### [Infrastructure as Code](./infrastructure-as-code)

Two infrastructure workstreams: a horizontally scaled multi-tier WordPress deployment on bare-metal Linux, and a 40+ project Terraform portfolio targeting AWS via LocalStack emulation.

**Key implementations:** NFS-backed shared document root across a three-node Apache/PHP cluster; MariaDB wildcard host grants for cross-node connectivity; full `init > validate > plan > apply > verify` discipline on every Terraform project; implicit dependency resolution via attribute references; dual-layer verification through Terraform state and independent AWS CLI queries; duplicate provider conflict resolution and LocalStack tag drift remediation.

---

### [Kubernetes](./kubernetes)

Production-aligned Kubernetes operations on a live K3s v1.34.1 cluster. Covers workload deployment, incident diagnosis and repair, storage provisioning, and multi-container design patterns.

**Key implementations:** Zero-downtime rolling updates and controlled rollbacks with revision inspection; live deployment fault repair using `kubectl set image` and manifest patching without redeployment; static PV/PVC binding with `storageClassName: ""` to resolve k3s `local-path` interference; MySQL stateful deployment with Secrets-injected credentials; init container and sidecar patterns for pre-initialization and log shipping; Nginx/PHP-FPM sidecar repair via ConfigMap rewrite and `subPath` volume mounting.

---

### [Linux Administration](./linux-administration)

Production Linux server administration across CentOS Stream 9 and Ubuntu environments, with a focus on security posture, service reliability, and operational automation.

**Key implementations:** SSH hardening with root login restriction verified via `sshd -T`; SELinux configuration management; TLS deployment with HTTP/2 on Nginx; PHP 8.1 deployment via Remi repository with Unix socket PHP-FPM integration; MariaDB and PostgreSQL provisioning under no-restart constraints; compound Apache fault resolution across port conflicts, syntax errors, and `iptables` blocks; Nginx Layer 7 load balancer configuration for a three-node backend tier.

---

### [Networking and Security](./networking-security)

Host-based firewall hardening and traffic segmentation across a three-server application tier.

**Key implementations:** `iptables` INPUT chain enforcement with source-restricted ACCEPT for a trusted load balancer IP and catch-all DROP for all other inbound traffic on the application port; `iptables-services` for reboot-persistent rule state on CentOS Stream 9; SSH ACCEPT rule positioned first in all chains to prevent administrative lockout; DROP over REJECT to avoid leaking port-state information.

---

## Technologies and Tools

| Category | Tools |
|---|---|
| CI/CD | Jenkins 2.541.x LTS, Groovy (Declarative Pipeline), Gitea, GitHub |
| Configuration Management | Ansible Core 2.11.x / 2.14.x, Jinja2 |
| Containers | Docker Engine 26.x / 29.x, Docker Compose v2, containerd |
| Version Control | Git 2.x, Gitea 1.25.3 |
| IaC | Terraform v1.11.0, `hashicorp/aws` v5.91.0, LocalStack |
| Kubernetes | K3s v1.34.1, kubectl |
| Web Servers | Apache HTTPD, Apache2, Nginx, Tomcat 9 |
| Databases | MariaDB 10.5, MySQL 5.7, PostgreSQL |
| Languages / Runtimes | Bash, Groovy, PHP-FPM 8.1, Python / Flask, OpenJDK 11 / 17 |
| Networking / Security | iptables, iptables-services, OpenSSH RSA 4096, SELinux, SSL/TLS |
| OS Platforms | CentOS Stream 9, AlmaLinux 9, Ubuntu 24.04 LTS |
| Verification | AWS CLI, curl, JSONPath, sshd -T, nginx -t, httpd -t, systemctl |

---

## Key Skills Demonstrated

**Incident Diagnosis and Repair**
Live fault resolution across compound failures: port conflicts, config syntax errors, permission drift, broken Kubernetes deployments, and Dockerfile build failures. Repairs applied without service disruption or resource deletion where constraints required it.

**Production Security Posture**
SSH root lockdown, iptables ingress filtering, TLS termination, SELinux policy management, RBAC enforcement in Jenkins, and Kubernetes Secrets-based credential isolation.

**Distributed Infrastructure**
Multi-node architecture patterns across Kubernetes clusters, Jenkins agent pools, Ansible-managed server fleets, and NFS-backed application tiers. Consistent configuration state enforced across all nodes with post-change sweep validation.

**Pipeline and Automation Engineering**
Declarative CI/CD pipelines with parameterized inputs, conditional branching, scheduled triggers, and chained job dependencies. Ansible playbooks and Bash scripts designed for idempotency, cron compatibility, and pipeline integration. Bash scripting for automated backups, user provisioning, and service health validation is covered within Linux Administration.

**IaC Discipline**
Full Terraform lifecycle with `init > validate > plan > apply > verify` on every project. Convergence confirmed via `No changes` on post-apply plans. State and AWS CLI used independently for dual-layer verification.

**Operational Documentation**
Every implementation documented with configuration rationale, key decisions, errors encountered with root cause analysis, and verified end-state output.

---

## How to Navigate

Each subdirectory contains its own `README.md` with a full project index, per-task summaries, and links to individual implementation walkthroughs.

**Recommended entry points by focus area:**

| Focus | Start Here |
|---|---|
| Security hardening | `linux-administration/ssh-hardening` > `networking-security/iptables-firewall-hardening` > `linux-administration/secure-nginx-https-deployment` |
| Kubernetes operations | `kubernetes/deployment-fault-diagnosis-redis` > `kubernetes/mysql-stateful-deployment-secrets` > `kubernetes/guestbook-redis-multitier` |
| CI/CD pipelines | `ci-cd/jenkins/` index for sequenced pipeline complexity |
| Container engineering | `docker/dockerfile-fault-resolution` > `docker/docker-compose-lamp-stack-provisioning` |
| IaC | `infrastructure-as-code/multi-tier-wordpress-architecture` > `infrastructure-as-code/terraform/` index |
| Configuration management | `configuration-management/ansible-httpd-role-jinja2-template` > `configuration-management/ansible-ssh-passwordless-setup` |
| Git operations | `git-version-control/git-release-automation` > `git-version-control/distributed-scm-workflows` |

All projects share the same Stratos Datacenter infrastructure context (jump host, three application servers, dedicated DB and backup nodes), so environment details carry across documents.

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

<sub>100+ production-style DevOps implementations across 8 engineering domains.</sub>

</div>
