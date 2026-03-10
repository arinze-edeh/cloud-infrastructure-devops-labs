# AWS Cloud Engineering

<div align="center">

**Production-aligned AWS infrastructure implementations across compute, networking, security, storage, containers, databases, serverless, monitoring, and automation.**

[![AWS](https://img.shields.io/badge/Amazon_AWS-FF9900?style=flat-square&logo=amazonaws&logoColor=white)](#)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)](./automation-iac)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](./containers)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](./containers)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat-square&logo=linux&logoColor=black)](./compute)

</div>

---

## Overview

This directory contains **50 production-grade AWS implementations** organised by service domain. Each implementation addresses a specific infrastructure, security, networking, or architecture challenge modelled on real-world AWS environments. The work spans foundational cloud engineering through advanced multi-service architectures covering compute, identity, networking, storage, containers, serverless, databases, monitoring, and infrastructure automation.

Every implementation is fully documented with configuration details, architectural decisions, and validation outcomes.

---

## Repository Structure

```
aws/
├── automation-iac/          # Terraform and CloudFormation infrastructure provisioning
├── compute/                 # EC2, Auto Scaling, AMI, and instance lifecycle management
├── containers/              # ECS, EKS, ECR, and containerised application deployments
├── databases/               # RDS, DynamoDB provisioning and operations
├── identity-and-security/   # IAM, KMS, key pairs, and security group management
├── monitoring-and-logging/  # CloudWatch alarms, metrics, and centralised audit logging
├── networking/              # VPC, subnets, ALB, NAT, VPC Peering, and routing
├── serverless/              # Lambda functions and event-driven architectures
├── storage/                 # S3, EBS, snapshots, and static website hosting
├── troubleshooting/         # Incident diagnosis and connectivity fault resolution
└── README.md
```

---

## Domain Implementations

### Compute

EC2 instance provisioning and lifecycle management across multiple configurations. Deployed instances with key pair authentication, configured stop and termination protection for critical workload protection, and performed live instance type changes for right-sizing without data loss. Provisioned Elastic IP addresses for stable public endpoints, attached Elastic Network Interfaces for secondary network connectivity, and created Amazon Machine Images (AMIs) for repeatable instance deployment and disaster recovery. Managed EBS volume attachment to running instances and configured Auto Scaling groups with target tracking policies to maintain availability and optimise compute cost under variable load.

**Key implementations:** EC2 provisioning and lifecycle, Elastic IP attachment, ENI configuration, AMI creation, instance type migration, stop and termination protection, Auto Scaling group setup, EBS volume attachment.

---

### Networking

Designed and implemented AWS network architectures for secure, segmented, and highly available application delivery. Provisioned VPCs with public and private subnet tiers, internet gateway attachment, and route table configurations for controlled traffic flow. Configured VPC Peering for private cross-VPC communication without traffic traversing the public internet, enabling secure inter-environment connectivity. Deployed Application Load Balancers for HTTP and HTTPS traffic distribution across EC2 target groups with health check-based failover. Configured NAT Instances and NAT Gateways to provide outbound internet access for private subnet workloads while blocking unsolicited inbound connections. Diagnosed and resolved an EC2-hosted application internet accessibility failure by systematically inspecting VPC route tables, internet gateway associations, security group rules, and public IP configurations.

**Key implementations:** VPC and subnet design, internet gateway and route table configuration, VPC Peering, Application Load Balancer, NAT Instance, NAT Gateway, security group engineering, EC2 connectivity troubleshooting.

---

### Identity and Security

Designed and enforced AWS IAM architectures aligned to the principle of least privilege across users, groups, roles, and policies. Provisioned IAM users and groups with scoped permission boundaries, authored read-only IAM policies for EC2 console access, and created service-specific policies for DynamoDB operations. Attached policies to IAM users for direct permission grants and to EC2 instance profiles for credential-free service-to-service authentication. Created IAM roles with managed policy attachments for EC2 workloads requiring AWS service access. Configured AWS KMS for envelope encryption of data at rest, and managed EC2 key pairs and security group ingress and egress rules for secure remote access control.

**Key implementations:** IAM user, group, and role provisioning, least-privilege policy authoring, EC2 instance profile configuration, read-only EC2 console policy, DynamoDB-scoped IAM policy, KMS encryption setup, key pair management, security group rule engineering.

---

### Storage

Implemented AWS storage architectures covering object storage, block storage, and static content delivery. Configured S3 bucket versioning for object-level change history and accidental deletion recovery, managed bucket policies for access control enforcement, and deployed S3-hosted static websites for frontend application delivery. Executed cross-bucket data migration using AWS CLI scripting for bulk object transfer operations. Managed EBS GP3 volume provisioning for cost-optimised IOPS performance, performed volume attachment to running EC2 instances, and created point-in-time EBS snapshots for backup and cross-AZ volume restoration. Provisioned private ECR repositories for secure container image storage with access-controlled pull operations.

**Key implementations:** S3 versioning, bucket policy management, static website hosting, cross-bucket CLI migration, GP3 volume provisioning, EBS attachment and snapshot management, ECR private repository setup.

---

### Databases

Provisioned and operated AWS managed database services for relational and NoSQL workloads. Deployed Amazon RDS instances within private subnets with restricted public access and application-level connectivity, configured security group rules to enforce database-tier network isolation, and performed RDS snapshot backup and point-in-time restore operations to validate recovery procedures. Designed and provisioned DynamoDB NoSQL tables with partition key schemas and IAM-scoped access policies for least-privilege application access.

**Key implementations:** RDS private subnet deployment, RDS security group configuration, RDS snapshot and restore, DynamoDB table provisioning, DynamoDB IAM policy scoping.

---

### Containers

Deployed and managed containerised workloads across Amazon ECS and Amazon EKS. Provisioned ECS task definitions and services for containerised application orchestration, managed container lifecycle operations, and configured service-level networking and IAM role bindings. Provisioned Amazon EKS clusters for production Kubernetes workloads, configured managed node groups, and deployed application Deployment and Service objects within the cluster. Established private ECR repositories for container image versioning and access-controlled distribution to ECS and EKS workloads.

**Key implementations:** ECS task definition and service deployment, EKS cluster provisioning, managed node group configuration, Kubernetes workload deployment on EKS, ECR integration with ECS and EKS.

---

### Serverless

Developed and deployed AWS Lambda functions using both the AWS Management Console and AWS CLI for environment-specific deployment workflows. Implemented event-driven compute pipelines triggered by S3 object creation events for automated data processing without persistent infrastructure. Integrated Amazon SQS queues and SNS topics to build decoupled asynchronous messaging architectures with guaranteed delivery semantics and fan-out notification patterns for multi-subscriber event distribution.

**Key implementations:** Lambda function authoring and deployment via Console and CLI, S3 event trigger configuration, SQS queue provisioning and integration, SNS topic and subscription setup, event-driven pipeline design.

---

### Monitoring and Logging

Implemented AWS observability infrastructure for proactive infrastructure health monitoring and audit compliance. Configured CloudWatch metric alarms with threshold-based alerting for EC2 instance health and resource utilisation, and defined alarm actions for automated notification and response workflows. Established centralised audit and network flow logging using VPC Flow Logs with cross-VPC visibility via VPC Peering to support security monitoring and compliance requirements across environments.

**Key implementations:** CloudWatch alarm configuration, metric threshold and alerting setup, alarm action definitions, VPC Flow Log enablement, centralised cross-VPC audit logging.

---

### Automation and IaC

Provisioned and managed AWS infrastructure using Terraform and AWS CloudFormation for repeatable, version-controlled, and auditable infrastructure delivery. Authored Terraform configurations covering VPC and subnet architecture, security group rules, EC2 instance provisioning with IAM instance profile bindings, DynamoDB-scoped IAM policy creation and attachment, private subnet EC2 deployments with NAT-based egress routing, and CloudWatch alarm configuration. Automated full infrastructure stack deployments using CloudFormation templates to enable consistent environment replication and change management.

**Key implementations:** Terraform VPC and subnet provisioning, Terraform security group and EC2 configuration, Terraform IAM policy authoring, Terraform DynamoDB access policy attachment, Terraform CloudWatch alarm setup, CloudFormation stack deployment and management.

---

### Troubleshooting

Systematic fault diagnosis and resolution across AWS networking, compute, and connectivity layers. Diagnosed an EC2-hosted application internet accessibility failure by methodically inspecting the VPC route table for missing internet gateway routes, validating internet gateway attachment state, auditing security group ingress rules for the required application port, and confirming public IP association on the target instance. Identified and resolved the root cause misconfiguration, restored application accessibility, and documented the fault pattern and resolution for operational reference.

**Key implementations:** EC2 internet connectivity fault diagnosis, VPC route table inspection, internet gateway association validation, security group rule auditing, public IP binding verification, post-resolution validation.

---

## AWS Services Coverage

```
Compute          EC2, Auto Scaling, AMI, Elastic IP, ENI
Networking       VPC, Subnets, Route Tables, Internet Gateway, NAT Instance,
                 NAT Gateway, VPC Peering, Application Load Balancer, Security Groups
Identity         IAM Users, Groups, Roles, Policies, Instance Profiles, KMS
Storage          S3, EBS (GP3), EBS Snapshots, ECR, Static Website Hosting
Databases        Amazon RDS, DynamoDB
Containers       Amazon ECS, Amazon EKS, Amazon ECR
Serverless       AWS Lambda, Amazon SQS, Amazon SNS, S3 Event Triggers
Monitoring       Amazon CloudWatch (Alarms, Metrics, Logs), VPC Flow Logs
Automation       Terraform, AWS CloudFormation, AWS CLI
```

---

## Related Repositories

| Repository | Description |
|---|---|
| [Cloud and DevOps Engineering](../README.md) | Full portfolio README covering DevOps, AWS, and Azure |
| [Azure Cloud Engineering](../azure/README.md) | 50 Azure implementations across VMs, networking, storage, containers, and databases |
| [DevOps Engineering](../devops/README.md) | 100 implementations across Linux, Docker, Kubernetes, Jenkins, Ansible, and Terraform |

---

<div align="center">

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/arinze-edeh)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/arinze-edeh)
[![Email](https://img.shields.io/badge/Email-D14836?style=flat-square&logo=gmail&logoColor=white)](mailto:edeharinze389@gmail.com)

<sub>50 production-grade AWS implementations across 10 service domains.</sub>

</div>
