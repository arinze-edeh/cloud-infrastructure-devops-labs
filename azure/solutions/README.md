# Azure Solutions

> **Production-aligned Azure infrastructure labs covering networking, compute, and storage integration patterns built and documented to professional portfolio standards.**

---

## Overview

This directory contains completed Azure infrastructure solutions developed against real-world KodeKloud/Nautilus lab environments. Each project reflects the constraints, trade-offs, and operational decisions present in production cloud work: policy-restricted subscriptions, dynamic resource naming, CLI-first automation, and end-to-end verification before sign-off.

The work targets the patterns most common in junior-to-mid cloud and DevOps roles: Layer 7 load balancing, secure content delivery pipelines, network segmentation, and infrastructure-as-code tooling via Azure CLI and ARM REST API.

---

## Directory Structure

```
azure/solutions/
├── application-gateway-vm-ingress-traffic-distribution/
│   └── README.md
├── private-blob-nginx-static-content-delivery/
│   └── README.md
└── README.md  <-- (this file)
```

---

## Project Summaries

### [Application Gateway with Load-Balanced Backend VMs](./application-gateway-vm-ingress-traffic-distribution/)

**Quick Summary:** Deployed an Azure Application Gateway (Basic SKU) distributing HTTP traffic across two Nginx-backed Ubuntu VMs using round-robin load balancing, with backend health verification and traffic distribution validated via curl.

| | |
|---|---|
| **Purpose** | Provision a Layer 7 load balancer in front of two differentiated backend VMs to demonstrate traffic distribution and gateway lifecycle management. |
| **Approach** | Infrastructure was built entirely via Azure CLI and `az rest` ARM API calls. The Application Gateway was deployed using a structured ARM JSON body rather than `az network application-gateway create`, providing full named-component control over every sub-resource. An asynchronous polling loop detected gateway readiness without arbitrary sleep delays. `cookieBasedAffinity` was explicitly disabled to ensure true round-robin distribution. |
| **Key Decisions** | Using `az rest` over the CLI shorthand command gave explicit control over listener, routing rule, and backend settings naming, matching what ARM templates produce internally. A dedicated `/24` subnet for the Application Gateway satisfied Azure's hard isolation requirement without disrupting VM networking. |
| **Outcome** | Gateway reached `operationalState: Running` with both backends reporting `Healthy`. Ten sequential curl requests confirmed alternating Version 1 and Version 2 responses, validating round-robin traffic distribution end-to-end. |

---

### [Private Blob Storage with Nginx Static Content Delivery](./private-blob-nginx-static-content-delivery/)

**Quick Summary:** Built a secure static web delivery pipeline where an Ubuntu VM fetches `index.html` from a private Azure Blob container using a storage account key, serves it via Nginx, and exposes it publicly over HTTP.

| | |
|---|---|
| **Purpose** | Decouple static web assets from the application codebase by using Azure Blob Storage as a centralized, independently managed content repository, with the VM fetching content at deployment time rather than embedding it in source control. |
| **Approach** | A private storage account (`--allow-blob-public-access false`) was provisioned with a dedicated blob container. The VM was bootstrapped via SSH using a 4096-bit RSA key pair, with password authentication explicitly disabled. Azure CLI was installed on the VM to authenticate blob downloads using a storage account key. File ownership and permissions were set to `www-data:www-data` and `chmod 644` before Nginx restart. A loopback `curl http://localhost` was run inside the VM prior to external verification to isolate any failure domain. |
| **Key Decisions** | SSH-only authentication was enforced at provisioning time rather than as a post-deployment hardening step, eliminating brute-force attack surface from the start. The storage account key was echoed as an 8-character prefix only, avoiding full key exposure in terminal history. The loopback test before external `curl` created a clean diagnostic boundary between web server issues and network/NSG issues. |
| **Outcome** | External HTTP request to the VM's public IP returned the correct 233-byte `index.html` content, confirming full pipeline integrity from blob upload through Nginx delivery. |

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Cloud Platform | Microsoft Azure (East US region) |
| CLI and API | Azure CLI (`az`), ARM REST API via `az rest` |
| Compute | Ubuntu 22.04 LTS, Standard_B1s, Azure VM Custom Script Extension |
| Networking | Azure VNet, Subnets, NSG, Application Gateway (Basic SKU), Public IP (Standard SKU) |
| Storage | Azure Blob Storage, Standard_LRS, Private containers |
| Web Server | Nginx 1.18.0 |
| Scripting | Bash, shell variable injection, polling loops |
| Authentication | SSH RSA 4096-bit key pair, Storage Account Key, Azure CLI session auth |

---

## Key Skills Demonstrated

**Infrastructure Design**
Isolated subnet architecture separating gateway and compute workloads. Static IP allocation for stable frontend endpoints. NSG rules with explicit priorities leaving headroom for future rule insertion.

**CLI-First Automation**
All resources provisioned and verified without portal interaction. Dynamic resource group resolution via `az group list` makes scripts portable across ephemeral lab environments. ARM resource IDs captured into shell variables and composed into structured API bodies.

**Production Operational Patterns**
Asynchronous provisioning managed with a controlled polling loop rather than arbitrary sleep. Backend health confirmed before traffic testing. Loopback verification used to isolate failure domains before external validation.

**Security Posture**
Public blob access disabled at the storage account level. SSH password authentication eliminated at provisioning time. File permissions aligned to the Nginx process identity (`www-data`). Storage key exposure limited in terminal output.

**Documentation Standards**
Each project includes an architecture diagram, phase-by-phase CLI reference with expected outputs, inline screenshot placeholders, an error and resolution log, best practices table, and a lessons learned section following FAANG-style portfolio conventions.

---

## How to Navigate

Each project folder contains a standalone `README.md` with the full implementation guide. The documentation is structured for two reading modes:

- **Quick scan:** Architecture diagram, Resource Summary table, and Key Decisions sections give a complete picture in under two minutes.
- **Deep reference:** Phase-by-phase CLI commands with expected outputs, error logs, and lessons learned support full reproduction or adaptation of each deployment.

To reproduce any project, resolve the target resource group name dynamically using:

```bash
RG=$(az group list --query "[0].name" --output tsv)
echo "Resource Group: $RG"
```

Then follow the phase sequence in the relevant project README. All commands are written for Bash with Azure CLI authenticated via `az login`.

---

## Author

**Arinze Edeh**
[GitHub: arinze-edeh](https://github.com/arinze-edeh) | [cloud-infrastructure-devops-labs](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)
