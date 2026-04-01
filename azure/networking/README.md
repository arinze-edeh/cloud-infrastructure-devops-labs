# Azure Networking

![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?style=flat-square&logo=microsoftazure&logoColor=white)
![CLI](https://img.shields.io/badge/Azure_CLI-Automation-blue?style=flat-square&logo=windowsterminal&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Labs-Production_Documented-brightgreen?style=flat-square)

---

## Overview

This directory documents a series of Azure networking labs executed on live Azure Free Labs subscriptions. Each lab addresses a real-world infrastructure scenario: provisioning isolated or public-facing network environments, diagnosing and remediating connectivity failures, deploying load balancing and gateway solutions, and enforcing traffic policy at the NSG and routing layers.

All tasks were completed via Azure CLI with full documentation of commands, errors encountered, root cause analysis, and verification steps. The work reflects operational patterns found in production cloud environments, including policy-constrained subscription behavior and multi-layer debugging workflows.

---

## Directory Structure

```
azure/networking/
├── application-gateway-vm-backend-orchestration/
├── azure-vnet-provisioning/
├── ingress-configuration/
├── ingress-traffic-orchestration/
├── nic-attachment-to-vm/
├── nsg-http-ssh-access/
├── private-vnet-compute-isolation/
├── public-ip-allocation/
├── public-vnet-ingress/
├── virtual-network-creation/
├── vm-egress-diagnostics/
├── vm-public-ip-association/
├── vnet-arm-datacenter-deployment/
├── vnet-peering-connectivity/
├── vnet-subnets/
└── README.md
```

---

## Project Summaries

---

### [application-gateway-vm-backend-orchestration](./application-gateway-vm-backend-orchestration/)

**Quick Summary:** Deployed an Ubuntu VM running Nginx behind an Azure Application Gateway (Basic SKU) using a full CLI-driven workflow, including a three-attempt SKU resolution that required bypassing the Azure CLI entirely via `az rest`.

**Purpose:** Provision a private VM accessible only through a public-facing Application Gateway, enforcing centralized traffic control and eliminating direct VM exposure.

**Approach:** Built layered infrastructure (NSG, VNet, two subnets, VM with cloud-init, Standard public IP, Application Gateway) with explicit attention to Azure Free Labs policy constraints. When both `Standard_v2` and `Standard_Small` were blocked by subscription policy and `Basic` was rejected by the CLI's local argument validator, the AGW was deployed using a raw ARM PUT request via `az rest` at `api-version=2023-05-01`.

**Outcome:** Full end-to-end traffic path validated with `curl` returning HTTP 200 and `show-backend-health` returning `Healthy`. All five errors encountered are documented with root cause and resolution.

**Key Skills:** Application Gateway configuration, ARM API workarounds, cloud-init, NSG design, policy-constrained environments.

---

### [azure-vnet-provisioning](./azure-vnet-provisioning/)

**Quick Summary:** Provisioned a Virtual Network with a subnet using Azure CLI as the foundational networking layer for a phased cloud migration.

**Purpose:** Establish a private VNet with a correctly scoped IPv4 CIDR block in the target region before deploying any compute or security resources.

**Approach:** Dynamic resource group resolution, VNet and subnet created atomically in a single CLI command to avoid intermediate incomplete state.

**Outcome:** VNet confirmed with `az network vnet show`, subnet embedded and verified. Infrastructure ready for subsequent NSG and VM deployment.

---

### [ingress-configuration](./ingress-configuration/)

**Quick Summary:** Configured inbound traffic rules and network security group policies to control public access to Azure-hosted services.

**Purpose:** Enforce least-privilege inbound access by explicitly defining allowed protocols and ports rather than relying on Azure defaults.

**Approach:** NSG rule creation with priority-ordered inbound rules scoped to required ports, verified against effective security rule output.

**Outcome:** Controlled ingress path established and validated at the NSG layer.

---

### [ingress-traffic-orchestration](./ingress-traffic-orchestration/)

**Quick Summary:** Deployed a Standard Azure Load Balancer fronting an Nginx VM, with a health probe, TCP load balancing rule, and NSG inbound rule all provisioned and validated.

**Purpose:** Replace direct VM public IP access with a production-grade Layer 4 ingress path that supports health monitoring and horizontal scaling.

**Approach:** Created a static Standard public IP, Standard Load Balancer, HTTP health probe (15-second interval), TCP LB rule on port 80, and registered the VM NIC IP configuration to the backend pool. NSG inbound rule added to permit HTTP from any source.

**Outcome:** `curl -v http://52.188.1.76` returned HTTP 200 with the Nginx welcome page, confirming the full traffic path through the load balancer to the backend VM.

**Key Skills:** Azure Load Balancer, health probes, NIC backend pool registration, NSG orchestration.

---

### [nic-attachment-to-vm](./nic-attachment-to-vm/)

**Quick Summary:** Attached an existing secondary NIC to a running Azure VM using CLI, following the required deallocate-attach-start workflow.

**Purpose:** Expand VM network interfaces to support additional subnet connectivity without redeploying the VM.

**Approach:** Identified NIC and resource group, deallocated the VM, attached the NIC with `az vm nic add`, restarted, and confirmed `powerState: VM running`.

**Outcome:** Secondary NIC attached successfully with primary NIC unchanged and VM restored to running state.

---

### [nsg-http-ssh-access](./nsg-http-ssh-access/)

**Quick Summary:** Created an NSG with explicit inbound Allow rules for HTTP (port 80) and SSH (port 22), backed by a reusable Bash automation script.

**Purpose:** Establish a baseline inbound access policy for web-serving VMs using least-privilege rules with correct priority ordering.

**Approach:** Rules created at priorities 100 (HTTP) and 110 (SSH) to evaluate before the default `DenyAllInBound` at priority 65500. Full configuration automated in `az-nsg-nautilus.sh`.

**Outcome:** NSG rule table confirmed with both custom rules active. Script tested for repeatability.

---

### [private-vnet-compute-isolation](./private-vnet-compute-isolation/)

**Quick Summary:** Provisioned a fully private VM with no public IP, locked down to SSH access from within the VNet CIDR only, using dual NSG binding at both subnet and NIC layers.

**Purpose:** Eliminate the external attack surface entirely for backend workloads by removing public IP allocation and restricting SSH to internal VNet sources.

**Approach:** NSG created with an `AllowSSH` rule scoped to `10.0.0.0/16` source and destination. NSG applied at both the subnet level (`az network vnet subnet update`) and NIC level (`az vm create --nsg`). VM deployed with `--public-ip-address ""` to suppress default public IP assignment.

**Outcome:** VM confirmed with `privateIpAddress: 10.0.1.4` and `publicIpAddress: ""`. NIC NSG binding verified with `az network nic show`.

**Key Skills:** Defense in depth, private compute isolation, NSG dual-layer binding, SSH key authentication.

---

### [public-ip-allocation](./public-ip-allocation/)

**Quick Summary:** Allocated a Standard SKU static public IP after resolving a subscription policy that blocked Dynamic allocation with Basic SKU.

**Purpose:** Provision a reusable static public IP as a prerequisite for load balancer or Application Gateway frontend configuration.

**Approach:** Initial Dynamic allocation attempt failed. Recreated with `--sku Standard --allocation-method Static`, which satisfies both the subscription quota constraint and the AGW/LB dependency requirement.

**Outcome:** Public IP provisioned with `provisioningState: Succeeded` and static address confirmed via `az network public-ip show`.

---

### [public-vnet-ingress](./public-vnet-ingress/)

**Quick Summary:** Deployed a public-facing VNet with a subnet and VM via Azure Portal, resolving a `RequestDisallowedByPolicy` disk type error and a Spot pricing validation failure.

**Purpose:** Provision an internet-accessible VM for a public-facing workload using the Portal workflow, with SSH on port 22 open from the internet.

**Approach:** VNet and subnet configured with public outbound access enabled. VM deployment failed on first attempt due to Premium SSD being blocked by subscription policy. Resolved by selecting Standard HDD and redeploying. Spot pricing error cleared by unchecking the discount option.

**Outcome:** VM deployed at `13.68.188.64` with SSH access confirmed and NSG inbound rule for port 22 verified.

---

### [virtual-network-creation](./virtual-network-creation/)

**Quick Summary:** Created a named VNet in Central US with a `/16` address space as the networking foundation for upcoming infrastructure deployments.

**Purpose:** Establish a VNet baseline with enough address space for future subnet segmentation across multiple workload tiers.

**Approach:** Dynamic resource group resolution via `az group list`, VNet created with `az network vnet create --address-prefixes 10.0.0.0/16`.

**Outcome:** VNet `xfusion-vnet` confirmed active in `centralus` with correct CIDR assignment.

---

### [vm-egress-diagnostics](./vm-egress-diagnostics/)

**Quick Summary:** Diagnosed and resolved a complete outbound connectivity failure on an Azure VM caused by an NSG `Deny *` rule at priority 200 blocking all egress traffic before the default internet allow rule could be evaluated.

**Purpose:** Restore `apt-get` package installation capabilities on a production VM by identifying and removing the offending NSG rule without disrupting inbound SSH access.

**Approach:** Resolved NIC and NSG programmatically, listed rules sorted by priority, confirmed `Block-All-Outbound` at priority 200 was intercepting all traffic before `AllowInternetOutBound` at 65001. Documented the broken state inside the VM (100% packet loss, `apt-get` returning all `Ign`) before deletion. Chained `az network nsg rule delete` with immediate re-list using `&&` to confirm atomically.

**Outcome:** Post-fix SSH session confirmed `0% packet loss`, `apt-get update` fetching 44 MB, `curl` installed at version `7.81.0`.

**Key Skills:** NSG rule priority debugging, non-interactive SSH verification, outbound traffic diagnosis.

---

### [vm-public-ip-association](./vm-public-ip-association/)

**Quick Summary:** Attached an orphaned public IP from `westus` to a VM NIC in `eastus` using the resource ID to handle cross-region association.

**Purpose:** Restore public accessibility to a VM whose public IP existed as a standalone resource but was not attached to its NIC IP configuration.

**Approach:** Discovered the correct NIC IP configuration name (`ipconfigdatacenter-vm-pip`) via `az network nic show` after an initial attempt with `ipconfig1` returned `ResourceNotFoundError`. Used the full resource ID (`$PIP_ID`) for the attachment to handle the cross-region dependency.

**Outcome:** Public IP `23.100.42.173` confirmed attached and VM accessible. Addresses the assumption pattern that commonly causes `ResourceNotFoundError` errors.

---

### [vnet-arm-datacenter-deployment](./vnet-arm-datacenter-deployment/)

**Quick Summary:** Modified and deployed an ARM template to provision a standardized VNet with updated naming, address space, and environment tagging using `sed` and `az deployment group create`.

**Purpose:** Apply Infrastructure as Code practices to VNet provisioning by modifying an existing ARM template rather than running ad-hoc CLI commands.

**Approach:** Used `sed` for in-place template modifications (VNet name, CIDR `192.168.0.0/16`, environment tag `KKE-datacenter`). Validated JSON structure in `vi` before deployment. Deployed with `az deployment group create` and verified with `az network vnet show --query`.

**Outcome:** VNet `arm-vnet-datacenter` deployed with correct address space, display name, and environment tag confirmed via query output.

**Key Skills:** ARM template modification, `sed` substitution, IaC deployment workflow.

---

### [vnet-peering-connectivity](./vnet-peering-connectivity/)

**Quick Summary:** Configured bidirectional Azure VNet Peering between a public VM VNet (`10.2.0.0/16`) and a private VM VNet (`10.1.0.0/16`), then validated with live ICMP traffic from the public VM to the private VM's internal IP.

**Purpose:** Enable private network communication between two isolated VNets without a VPN gateway, using the Azure backbone network.

**Approach:** Verified non-overlapping address spaces before creation. Created peering in both directions using full resource IDs for `--remote-vnet`. First peering showed `peeringState: Initiated`; second peering transitioned both to `peeringState: Connected, peeringSyncLevel: FullyInSync`. SSH'd into the public VM and pinged `10.1.1.4` directly.

**Outcome:** `ping -c 4 10.1.1.4` returned 0% packet loss with average latency of 1.417 ms, confirming full inter-VNet routing over the Azure backbone.

**Key Skills:** VNet Peering, bidirectional peering creation, private routing validation.

---

### [vnet-subnets](./vnet-subnets/)

**Quick Summary:** Provisioned a VNet with an embedded subnet in a single atomic CLI command, targeting the South Central US region.

**Purpose:** Demonstrate correct subnet-level CIDR scoping within a VNet address space, with the subnet and VNet provisioned together to avoid intermediate incomplete state.

**Approach:** `az network vnet create` with `--subnet-name` and `--subnet-prefix` flags to create both resources in one operation.

**Outcome:** VNet `nautilus-vnet` (`10.0.0.0/16`) confirmed with subnet `nautilus-subnet` (`10.0.1.0/24`) embedded and both showing `provisioningState: Succeeded`.

---

## Technologies and Tools

| Category | Tools / Services |
|---|---|
| Cloud Platform | Microsoft Azure (Azure Free Labs subscriptions) |
| CLI | Azure CLI 2.67.0, Bash |
| Networking | VNet, Subnet, NSG, Route Table, Public IP, NIC, VNet Peering |
| Load Balancing | Azure Load Balancer (Standard SKU), Application Gateway (Basic SKU) |
| API Access | Azure REST API (`az rest --method PUT`), ARM templates |
| Compute | Ubuntu 22.04 LTS, Standard_B1s, cloud-init |
| Web Server | Nginx (auto-deployed via cloud-init `custom-data`) |
| Infrastructure as Code | ARM JSON templates, Bash automation scripts |
| Verification | `curl`, `ping`, `ssh`, `watch`, `az network ... show-backend-health` |

---

## Key Outcomes and Skills Demonstrated

**Network Architecture**
Designed and deployed segmented VNet environments with correctly scoped subnets, dedicated AGW subnets, and dual NSG binding patterns for defense in depth.

**Traffic Control**
Implemented inbound and outbound NSG rules with correct priority ordering, diagnosed rule conflicts causing connectivity failures, and restored internet egress by removing a misplaced deny rule.

**Load Balancing and Gateway Deployment**
Provisioned a Standard Azure Load Balancer with health probes and LB rules, and deployed an Application Gateway (Basic SKU) using a raw ARM API workaround when CLI validation conflicted with subscription policy.

**Incident Diagnosis and Remediation**
Resolved three compounding misconfigurations (blocked route table, missing NSG rule, detached public IP) and a complete VM egress failure, each documented with root cause, fix, and post-remediation verification.

**Infrastructure as Code Practices**
Modified and deployed ARM templates, parameterized all CLI workflows with environment variables, and authored reusable Bash scripts for repeatable NSG provisioning.

**Policy-Constrained Environment Navigation**
Identified and resolved subscription-level Azure Policy conflicts, SKU quota restrictions, and CLI argument validator gaps using `az rest` as a controlled escape hatch.

---

## How to Navigate

Each subdirectory contains a `README.md` with the full implementation walkthrough for that task, including CLI commands, expected outputs, screenshots, error documentation, and verification steps.

To reproduce any lab:

1. Authenticate with `az login` or a pre-configured service principal
2. Set your subscription: `az account set --subscription <id>`
3. Export the resource group: `RG=$(az group list --query "[0].name" -o tsv)`
4. Follow the step-by-step commands in the target directory's README

> All labs were executed on Azure Free Labs subscriptions. Some commands reflect policy-specific workarounds for that environment. Review the Errors section in each README before adapting commands for other subscription tiers.

---

*Region: Multi-region (East US, West US, Central US, South Central US) | Stack: Azure CLI, Bash, ARM*
