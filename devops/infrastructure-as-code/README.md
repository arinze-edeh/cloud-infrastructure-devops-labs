# Infrastructure as Code

> Production-style infrastructure provisioning across two workstreams: a horizontally scaled multi-tier web application deployed on bare-metal Linux hosts, and a 40+ project Terraform portfolio targeting AWS services via LocalStack emulation. Both reflect real DevOps workflows applied to the **Nautilus cloud migration initiative**.

---

## Table of Contents

- [Overview](#overview)
- [Directory Structure](#directory-structure)
- [Projects](#projects)
  - [Multi-Tier WordPress Architecture](#multi-tier-wordpress-architecture)
  - [Terraform AWS Portfolio](#terraform-aws-portfolio)
- [Technologies and Tools](#technologies-and-tools)
- [Key Skills Demonstrated](#key-skills-demonstrated)
- [How to Navigate](#how-to-navigate)

---

## Overview

This directory contains two infrastructure projects that together span the full provisioning spectrum: hands-on Linux systems administration with shared storage and load-balanced ingress, and declarative cloud infrastructure managed through the complete Terraform lifecycle (`init > validate > plan > apply > verify`).

Both projects prioritize production-relevant concerns: service consistency across distributed nodes, state-driven operations, dual-layer verification, and clean decommissioning workflows.

---

## Directory Structure

```
infrastructure-as-code/
├── multi-tier-wordpress-architecture/
├── terraform/
└── README.md
```

---

## Projects

---

### [Multi-Tier WordPress Architecture](./multi-tier-wordpress-architecture/)

**Quick summary:** A three-node Apache/PHP cluster backed by a dedicated MariaDB host, with a pre-mounted NFS volume providing shared document root consistency across all application servers. Validated end-to-end through a load balancer endpoint.

**Purpose:** Deploy WordPress consistently across horizontally scaled application servers without content drift, using a single centralized database and shared storage layer.

**Approach:**
- NFS mount at `/var/www/html` across all three app servers means `wp-config.php` is written once and immediately available to all nodes.
- Apache reconfigured to listen on port `3002` across the cluster to match load balancer routing rules.
- MariaDB provisioned with a wildcard host grant (`'%'`) to allow connections from any application server without per-host ACL management.
- Validation performed at two levels: HTTP header inspection confirming Apache and PHP are active, and load balancer UI confirming successful database connectivity.

**Outcome:** Full-stack deployment confirmed through the LBR endpoint. PHP application reads `wp-config.php` from shared NFS and returns successful database connection using `kodekloud_cap`.

**Stack:** Apache 2.4, PHP 8.0, MariaDB 10.5, CentOS Stream 9, NFS, systemd

---

### [Terraform AWS Portfolio](./terraform/)

**Quick summary:** 40+ discrete Terraform projects covering the core AWS service domains used in production cloud environments. Each project is a self-contained infrastructure unit with independent state, verified through both Terraform state inspection and AWS CLI.

**Purpose:** Provision and lifecycle-manage AWS resources as part of the Nautilus migration, applying IaC disciplines consistently across compute, networking, storage, IAM, databases, streaming, monitoring, and secrets management.

**Approach:**
- Every project follows the full `init > validate > plan > apply` gate before any resource is created.
- Implicit dependency resolution via attribute references is preferred throughout. Explicit `depends_on` is used only where provider ordering semantics require it.
- Post-apply `terraform plan` is run on all projects to confirm `No changes`, treating convergence as the acceptance criterion rather than apply success alone.
- Duplicate provider conflicts, tag drift loops, and deprecated argument patterns are resolved and documented within each project.

**Outcome:** 40+ AWS resources provisioned and verified across EC2, VPC, EBS, S3, IAM, DynamoDB, Kinesis, OpenSearch, CloudWatch, Secrets Manager, SSM, SNS, and CloudFormation. Full lifecycle coverage including targeted destroy, state inspection, and configuration preservation post-decommission.

**Stack:** Terraform v1.11.0, `hashicorp/aws` v5.91.0, LocalStack, AWS CLI, `hashicorp/tls`, `hashicorp/local`

---

## Technologies and Tools

| Category | Detail |
|---|---|
| IaC Engine | Terraform v1.11.0 |
| Cloud Provider | `hashicorp/aws` v5.91.0 |
| AWS Emulation | LocalStack (`http://aws:4566`) |
| Web Server | Apache HTTP Server 2.4 |
| Runtime | PHP 8.0 (`php-mysqlnd`, `php-gd`, `php-mbstring`) |
| Database | MariaDB 10.5 |
| Operating System | CentOS Stream 9 |
| Storage | NFS Shared Volume |
| Verification | AWS CLI, `curl`, systemd, `terraform state` |
| Shell | Bash (`sed`, heredoc authoring, `aws`, `terraform`) |

---

## Key Skills Demonstrated

**Multi-tier systems administration** NFS-backed shared storage, Apache cluster configuration, MariaDB remote access provisioning, and load balancer validation across distributed Linux hosts.

**Terraform lifecycle management** Full `init > validate > plan > apply > verify` discipline on every project. Targeted destroy, state backup awareness, and post-destroy configuration preservation applied consistently.

**Dependency modeling** Implicit resource dependencies via attribute references across all Terraform projects. Ordering enforced at the provider level where needed, not through redundant `depends_on` blocks.

**Dual-layer verification** Infrastructure confirmed through both Terraform state and independent AWS CLI queries, separating IaC-layer assertions from provider-layer ground truth.

**Conflict and drift resolution** Duplicate provider blocks, deprecated argument patterns, and LocalStack-specific tag drift loops identified and resolved across multiple projects.

**Idempotency discipline** Convergence confirmed via `No changes` on post-apply plans. End-to-end validation performed through the load balancer, not individual host checks, for the multi-tier deployment.

---

## How to Navigate

Each subdirectory contains its own `README.md` with full implementation detail, configuration reference, and verification output.

- For the WordPress deployment: start with [`multi-tier-wordpress-architecture/README.md`](./multi-tier-wordpress-architecture/README.md) for the architecture diagram, phased implementation steps, and validation checklist.
- For the Terraform portfolio: start with [`terraform/README.md`](./terraform/README.md) for the full project index and per-project summaries.

---

## Author

**Arinze Edeh** | Cloud and DevOps Engineer

[GitHub](https://github.com/arinze-edeh) | [LinkedIn](https://linkedin.com/in/arinze-edeh)
