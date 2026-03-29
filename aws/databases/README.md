# AWS Databases

![AWS](https://img.shields.io/badge/Cloud-AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![MySQL](https://img.shields.io/badge/Database-MySQL%208.4-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![DynamoDB](https://img.shields.io/badge/AWS-DynamoDB-FF9900?style=for-the-badge&logo=amazondynamodb&logoColor=white)
![AWS CLI](https://img.shields.io/badge/AWS_CLI-v1%2Fv2-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production%20Ready-28a745?style=for-the-badge)

---

## Overview

This directory covers managed database provisioning, private networking, backup and recovery, and NoSQL table operations on AWS. Each project reflects a production-relevant scenario drawn from real DevOps workflows: provisioning RDS instances with least-privilege access controls, building snapshot-based recovery pipelines, connecting application tiers to private database backends, and managing serverless NoSQL tables via CLI.

All implementations use the AWS CLI exclusively, with no console access, mirroring the constraints of automated pipelines and Infrastructure-as-Code workflows.

---

## Directory Structure

```
databases/
├── dynamodb-table-provisioning-seeding/     # NoSQL table creation and data ingestion via CLI
├── rds-mysql-scalable-provisioning/         # Private RDS MySQL instance with storage autoscaling
├── rds-private-mysql-ec2-connectivity/      # EC2-to-RDS private connectivity with PHP app deployment
├── rds-snapshot-restore-ops/                # Manual snapshot creation and restore validation
└── README.md
```

---

## Project Summaries

### [DynamoDB Table Provisioning and Seeding](./dynamodb-table-provisioning-seeding/)

**Quick Summary:** Provisions a serverless `datacenter-tasks` DynamoDB table on-demand, inserts two operational task records, and verifies data integrity via targeted key lookups, all via AWS CLI without console access.

| | |
|---|---|
| **Purpose** | Deliver a low-latency, schema-flexible task tracking backend for datacenter operations without the overhead of capacity planning or schema rigidity. |
| **Approach** | Used `PAY_PER_REQUEST` billing to eliminate RCU/WCU configuration. Gated all write operations behind `aws dynamodb wait table-exists` to prevent `ResourceNotFoundException` during the `CREATING` transition. Applied `--query` JMESPath expressions for targeted field verification rather than parsing raw JSON output. |
| **Outcome** | Fully operational table with two verified records in `us-east-1`. All 10 validation steps passed. Clean, reproducible workflow suitable for CI/CD integration. |

---

### [RDS MySQL Scalable Provisioning](./rds-mysql-scalable-provisioning/)

**Quick Summary:** Provisions a private, storage-autoscaling-enabled MySQL 8.4.8 RDS instance (`datacenter-rds`) on `db.t3.micro` using AWS CLI v1, including full pre-flight network discovery and documented error resolution.

| | |
|---|---|
| **Purpose** | Provide the Nautilus Development Team with a sandboxed private RDS instance ready for application data storage, with autoscaling headroom and no manual capacity management. |
| **Approach** | Ran pre-flight checks across subnet groups, VPC, security group IDs, and available engine versions before issuing the create command. Documented and resolved two CLI v1-specific errors: the `--publicly-accessible false` boolean flag syntax failure and the `@` character rejection in `MasterUserPassword`. Storage autoscaling was enabled via `--max-allocated-storage 50` with no separate flag required. |
| **Outcome** | Instance reached `available` state with all seven validation attributes confirmed: status, engine, version, instance class, public accessibility, Multi-AZ, and autoscaling threshold. |

---

### [EC2 to RDS Private Connectivity](./rds-private-mysql-ec2-connectivity/)

**Quick Summary:** Provisions a private MySQL RDS instance and connects it to an existing Apache/PHP EC2 application tier, enforcing security group chaining to scope database access exclusively to the application layer.

| | |
|---|---|
| **Purpose** | Replace placeholder credentials in a non-functional PHP application with a live, privately accessible RDS backend, producing a browser-confirmed end-to-end connection. |
| **Approach** | Used security group chaining: the RDS ingress rule references the EC2 security group ID as its source rather than a CIDR block, scoping port 3306 access to the application tier only. Passwordless SSH was established via EC2 Instance Connect for initial key delivery, then permanently installed in `authorized_keys`. Credential injection into `index.php` used `sed` substitution at deploy time, keeping placeholder values out of source files until runtime. |
| **Outcome** | `curl http://<EC2_IP>` returns `Connected successfully`, confirming full stack connectivity from browser through Apache and PHP to a private RDS endpoint. |

---

### [RDS Snapshot and Restore Operations](./rds-snapshot-restore-ops/)

**Quick Summary:** Executes a full snapshot-to-restore pipeline against a live MySQL RDS instance (`xfusion-rds`), validating the organisation's backup and recovery mechanism before a major infrastructure update.

| | |
|---|---|
| **Purpose** | Confirm that a verified, recoverable snapshot exists and that the restore process produces a correctly configured, fully available instance before any production changes are applied. |
| **Approach** | Gated snapshot creation on confirmed `available` source status. Captured the full state transition sequence during restore: `creating` to `configuring-enhanced-monitoring` to `backing-up` to `available`, documenting each intermediate state as expected behavior rather than a failure condition. Passed `--db-instance-class db.t3.micro` at restore time to eliminate a separate modify cycle. |
| **Outcome** | Restored instance `xfusion-snapshot-restore` reached `available` state with all seven validation gates passed, including instance class, engine version, and VPC placement. Zero remediation steps required. |

---

## Technologies and Tools

| Technology | Usage |
|---|---|
| **Amazon RDS** | Managed MySQL 8.4.x provisioning, snapshot and restore, private networking |
| **Amazon DynamoDB** | Serverless NoSQL table provisioning, item insertion, key-based retrieval |
| **AWS CLI v1 / v2** | Sole interface for all provisioning, configuration, and verification operations |
| **MySQL 8.4.x** | Relational engine across all RDS projects |
| **EC2 / Apache / PHP** | Application tier for EC2-to-RDS connectivity validation |
| **IAM** | Least-privilege credential scoping per operation |
| **VPC / Security Groups** | Private network isolation, security group chaining for database access control |
| **EC2 Instance Connect** | Ephemeral SSH key delivery for passwordless access setup |
| **JMESPath (`--query`)** | Targeted field extraction to avoid raw JSON parsing in CLI operations |
| **Bash / `sed`** | Environment variable exports, runtime credential injection |

---

## Key Skills Demonstrated

**Infrastructure Provisioning**
- RDS instance creation with explicit engine version targeting, instance class selection, storage autoscaling, and private endpoint enforcement
- DynamoDB table provisioning with on-demand billing and mandatory state-gating before write operations

**Networking and Security**
- Security group chaining to scope database access to the application tier without CIDR-based rules
- Private RDS deployment with `--no-publicly-accessible` enforced across all projects
- Least-privilege IAM policy design scoped to individual table and instance ARNs

**Backup and Recovery**
- End-to-end snapshot-to-restore pipeline with state transition monitoring
- Manual snapshot management and restore targeting with instance class override at restore time

**CLI Proficiency and Debugging**
- AWS CLI v1 vs v2 behavioral differences, specifically boolean flag syntax (`--no-publicly-accessible` vs `--publicly-accessible false`)
- RDS password character restriction handling at the API validation layer
- `wait` command usage as a mandatory execution gate rather than an optional step

**Production Readiness**
- Pre-flight environment verification across identity, region, VPC, and resource state before every provisioning operation
- Runtime credential injection using `sed` substitution, keeping placeholder values in source until deploy time
- Documented error resolution with root cause analysis for every failure encountered

---

## How to Navigate

Each subdirectory contains a standalone `README.md` with:
- Architecture diagram
- Full CLI command reference with flag-level explanations
- Actual terminal output at every step
- Inline screenshots linked from GitHub
- A dedicated errors and resolutions section where applicable
- A validation summary table confirming all requirements were met

To reproduce any lab, start with the **Prerequisites** and **Environment Verification** sections in the relevant project README, then follow the numbered phases in order.

---

> Region: `us-east-1` | All projects executed in sandboxed AWS environments via KodeKloud platform.
