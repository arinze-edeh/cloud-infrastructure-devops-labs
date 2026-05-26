# CI/CD Engineering Portfolio

**Domain:** Continuous Integration and Continuous Delivery | **Environment:** Stratos Datacenter (xFusionCorp / Nautilus Labs)

---

## Overview

This directory covers CI/CD infrastructure built across a simulated enterprise datacenter. The work reflects real operational scenarios: automating software delivery, enforcing access controls, managing distributed build infrastructure, and maintaining live application deployments through structured pipelines.

Currently documented: Jenkins. Coverage spans the full Jenkins lifecycle, from server installation and plugin management through multi-node agent configuration, declarative pipeline authoring, RBAC enforcement, and production-pattern deployment automation.

---

## Directory Structure

```
ci-cd/
└── jenkins/    # Jenkins CI/CD implementations across 14 operational scenarios
```

---

## Subfolders

### [Jenkins](./jenkins/)

**Quick Summary:** 14 Jenkins implementations covering the complete CI/CD engineering stack. Ranges from bare-OS server setup through distributed agent architecture, parameterized pipelines, webhook-triggered deployments, scheduled automation, and least-privilege access control.

**Scope:**
- Server installation and initial configuration on Ubuntu 24.04, including GPG signing issue resolution
- Multi-node SSH agent registration across three AlmaLinux 9 app servers with per-user credential isolation
- Declarative and Freestyle pipelines with parameterized inputs, conditional branching, and live validation stages
- Webhook-triggered and cron-scheduled automation for deployments, database backups, and log collection
- Matrix Authorization Strategy RBAC at both global and job-level scopes with anonymous access fully removed
- Plugin ecosystem management including dependency chain resolution before installation

**Production Patterns Applied:** Key-based SSH authentication, idempotent deployment scripts with exit-code guards, pre-flight connectivity validation, out-of-band verification on target servers, and staged restarts to protect in-flight builds.

---

## Technologies and Tools

| Category | Technologies |
|---|---|
| CI/CD Platform | Jenkins 2.541.x (LTS) |
| Source Control | Gitea, Git 2.52, GitHub |
| Languages | Groovy (Declarative Pipeline), Bash |
| Authentication | OpenSSH RSA 4096, ssh-copy-id, ssh-keygen, JSch |
| Plugins | SSH Build Agents, Publish Over SSH, Git, GitLab, Pipeline, Credentials, Matrix Authorization Strategy, Bouncy Castle API |
| Web Server | Apache HTTP Server (httpd) |
| Package Management | APT (Ubuntu), YUM/DNF (AlmaLinux 9 / CentOS Stream 9) |
| Runtimes | OpenJDK 17 (agent and controller) |
| OS Platforms | Ubuntu 24.04 LTS (controller), AlmaLinux 9 / CentOS Stream 9 (agents) |
| Data Tools | mysqldump, rsync, sshpass, scp |
| Infrastructure | Multi-node Stratos Datacenter, load balancer, storage server, jump host |

---

## Key Outcomes and Skills Demonstrated

**Full Lifecycle Coverage**
Provisioned Jenkins from a bare OS, resolved non-trivial installation issues, and built out a complete CI/CD platform with no reliance on pre-configured environments.

**Distributed Build Infrastructure**
Designed and validated a multi-node agent architecture with SSH key-based authentication, Java version compatibility management, and label-pinned pipeline execution.

**Pipeline Engineering**
Authored production-pattern pipelines with parameterized inputs, branch-aware deployments, self-validating test stages, chained upstream/downstream job dependencies, and scheduled automation for backup and log collection workflows.

**Security and Access Control**
Enforced least-privilege access at both global and job scopes using Matrix Authorization Strategy, with per-user credential isolation and full elimination of anonymous access.

---

## How to Navigate

Start with the [Jenkins README](./jenkins/) for a full index of all implementations with purpose, approach, and outcome summaries for each project.

---

## Author

**Arinze Edeh**
Cloud Infrastructure and DevOps Engineering
[GitHub: arinze-edeh](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)





