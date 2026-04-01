# Azure Compute

> Production-focused Azure compute provisioning, configuration, and lifecycle management executed entirely via Azure CLI and Azure Portal, across real lab environments with policy-constrained subscriptions.

---

## Overview

This directory covers the full compute lifecycle on Microsoft Azure: VM provisioning, runtime configuration, disk management, networking, SSH access hardening, tagging, resizing, and App Service deployment. All labs were executed against live Azure Free Labs subscriptions under organizational policy enforcement (Standard_LRS disk SKU, max 128 GB OS disk, no Premium storage), mirroring real-world restricted enterprise environments.

---

## Directory Structure
```
azure/compute/
├── appservice-python-deployment/
├── azure-vm-cli/
├── azure-vm-nginx-bootstrap/
├── secure-linux-vm-provisioning/
├── ssh-keypair-creation/
├── static-public-vm-deployment/
├── virtual-machine-creation/
├── vm-disk-attachment/
├── vm-nginx-web-server-provisioning/
├── vm-resizing/
├── vm-storage-expansion/
└── vm-tagging/
```

---

## Project Summaries

---

### [appservice-python-deployment](./appservice-python-deployment)

**Quick Summary:** Provisions a Python 3.11 web app on Azure App Service (Linux, Basic B1) via Azure CLI with resource tagging and full CLI-based state verification. No portal required.

| | |
|---|---|
| **Purpose** | Deploy a production-ready Python runtime environment on Azure App Service for the Nautilus DevOps team |
| **Approach** | Created a dedicated Linux App Service Plan in West US, deployed the web app with `PYTHON:3.11` runtime, applied governance tags at creation time, and verified running state with a targeted `az webapp show` query |
| **Outcome** | Web app confirmed `Running` with correct runtime, region, OS type, and tags; full CLI-only verification with no portal dependency |

**Key Technical Decision:** Python on Azure App Service requires a Linux-based plan (`--is-linux`). Tags were applied at provisioning time rather than post-deploy to ensure governance compliance from first boot.

---

### [azure-vm-cli](./azure-vm-cli)

**Quick Summary:** Creates an Ubuntu 22.04 VM (`xfusion-vm`, Standard_B2s) using Azure CLI with SSH key generation and confirmed running state.

| | |
|---|---|
| **Purpose** | Demonstrate CLI-only VM provisioning with SSH authentication and compliant storage configuration |
| **Approach** | Generated SSH keys automatically via `--generate-ssh-keys`, applied `Standard_LRS` storage and 30 GB disk to satisfy policy, and verified power state post-creation |
| **Outcome** | VM running with SSH access enabled and no portal interaction |

---

### [azure-vm-nginx-bootstrap](./azure-vm-nginx-bootstrap)

**Quick Summary:** Provisions an Ubuntu VM in East US with Nginx auto-installed at first boot via cloud-init, publicly accessible on port 80. Validated with `HTTP 200 OK`.

| | |
|---|---|
| **Purpose** | Zero-touch Nginx deployment using Azure `--custom-data` for the Nautilus project's web layer |
| **Approach** | Authored a cloud-init bootstrap script, passed it via `--custom-data`, opened port 80 via NSG rule, and confirmed live HTTP response with `curl -I` |
| **Outcome** | `nginx/1.18.0 (Ubuntu)` serving `HTTP/1.1 200 OK` from public IP; NSG rules confirmed; `Standard_LRS` policy compliance maintained |

**Key Technical Decision:** Validation via external `curl` rather than in-VM checks, which is the only definitive proof of NSG rule effectiveness and public reachability.

---

### [secure-linux-vm-provisioning](./secure-linux-vm-provisioning)

**Quick Summary:** Deploys a policy-constrained Azure VM with RSA 4096-bit SSH key authentication, recovering from three distinct Azure Policy enforcement failures during the process.

| | |
|---|---|
| **Purpose** | Provision a hardened, password-less SSH VM in a restricted Azure subscription from a designated `azure-client` landing host |
| **Approach** | Generated RSA 4096-bit key pair, reused pre-existing NIC resources after policy-blocked cleanup, and applied `Standard_LRS` + `--os-disk-size-gb 30` to bypass policy gates |
| **Outcome** | SSH login confirmed from `azure-client` to `devops-vm` with no password prompt; hostname verified inside the VM |

**Error Documentation:** Three policy failures captured with root cause and resolution: Premium disk blocked on creation, disk type immutable on existing resource, and policy-blocked disk deletion requiring `--force-deletion none`.

---

### [ssh-keypair-creation](./ssh-keypair-creation)

**Quick Summary:** Creates an Azure-managed RSA SSH key pair (`devops-kp`) via the Azure Portal as a native Azure resource.

| | |
|---|---|
| **Purpose** | Satisfy cloud-validation requirements for SSH key existence as an Azure resource, not just a local file |
| **Approach** | Used Azure Portal SSH Keys service to generate and store the key pair, downloading the private key on creation |
| **Outcome** | `devops-kp` confirmed in Azure SSH Keys inventory; public key managed by Azure control plane |

**Key Insight:** Local SSH key generation does not satisfy Azure-managed infrastructure validation. Keys must exist as Azure resources for control-plane-level confirmation.

---

### [static-public-vm-deployment](./static-public-vm-deployment)

**Quick Summary:** Deploys an Ubuntu VM in Central US with a Standard Static Public IP pre-provisioned and explicitly associated at creation time. Includes secondary SSH key rotation.

| | |
|---|---|
| **Purpose** | Ensure persistent, stable external IP for application access across VM restarts |
| **Approach** | Created `devops-pip` (Standard SKU, Static) independently before VM creation, bound it via `--public-ip-address`, then validated persistence with `az network public-ip show` |
| **Outcome** | Static IP confirmed unchanged post-deployment; secondary SSH key added via `az vm user update` and re-validated |

---

### [virtual-machine-creation](./virtual-machine-creation)

**Quick Summary:** Provisions `xfusion-vm` (Ubuntu 22.04, Standard_B1s, 30 GB Standard HDD) via Azure Portal with NSG-controlled SSH access and full validation.

| | |
|---|---|
| **Purpose** | Portal-driven VM provisioning aligned with Nautilus cloud migration requirements |
| **Approach** | Configured all VM properties through the portal wizard: basics, networking (SSH allowed), disk (30 GB Standard HDD), and review/deploy |
| **Outcome** | VM deployed, SSH private key downloaded, and connectivity verified |

---

### [vm-disk-attachment](./vm-disk-attachment)

**Quick Summary:** Attaches an existing Azure managed disk (`devops-disk`) to a running VM as a data disk via the Azure Portal.

| | |
|---|---|
| **Purpose** | Expand VM storage capacity using an existing managed disk without reprovisioning |
| **Approach** | Navigated to VM disk settings, attached `devops-disk` as a data disk, confirmed attachment status and disk region alignment |
| **Outcome** | Disk confirmed attached under Data Disks with no deployment errors |

---

### [vm-nginx-web-server-provisioning](./vm-nginx-web-server-provisioning)

**Quick Summary:** Full Nginx web server deployment on Azure VM (`datacenter-vm`) with pre-created NSG, cloud-init bootstrap, and documented recovery from four deployment blockers.

| | |
|---|---|
| **Purpose** | Deploy a publicly accessible Nginx server on a policy-constrained Azure subscription with explicit NSG control |
| **Approach** | Created NSG and HTTP rule before VM creation, used `--custom-data` for zero-touch Nginx install, recovered from policy-blocked disk, invalid CLI flag, VM stuck in Failed state, and orphaned ghost disk |
| **Outcome** | `curl http://20.127.103.60` returns full Nginx HTML; `HTTP 200` confirmed; NSG Allow-HTTP rule active |

**Error Documentation:** Four failure modes fully documented with commands for force-deletion cleanup, orphaned disk removal, and correct `--storage-sku` syntax.

---

### [vm-resizing](./vm-resizing)

**Quick Summary:** Resizes `xfusion-vm` from `Standard_B1s` to `Standard_B2s` using `az vm update` and verifies running state post-operation.

| | |
|---|---|
| **Purpose** | Optimize compute resource allocation for an underutilized VM |
| **Approach** | Verified current SKU, applied resize via `az vm update --size`, started VM, and confirmed both final size and power state via `az vm get-instance-view` |
| **Outcome** | VM resized and confirmed `VM running` with `Standard_B2s` hardware profile |

---

### [vm-storage-expansion](./vm-storage-expansion)

**Quick Summary:** Expands OS disk from 32 GiB to 64 GiB and provisions a new 64 GiB managed data disk (`devops-disk`) with persistent ext4 mount via `/etc/fstab`.

| | |
|---|---|
| **Purpose** | Increase storage capacity on `devops-vm` for growing application workloads without reprovisioning |
| **Approach** | Deallocated VM, resized OS disk via `az disk update`, created and attached `devops-disk`, partitioned with `fdisk`, formatted ext4, mounted by UUID with `nofail` for boot resilience |
| **Outcome** | OS disk at 64 GiB; `devops-disk` mounted at `/mnt/devops-disk`; write test passed; mount persists across reboots |

**Key Technical Decisions:** UUID-based fstab mounting (not device name) for reboot stability; `nofail` flag prevents boot failure if disk is unavailable; `blkid` piped through `head -1` to isolate filesystem UUID from PTUUID.

---

### [vm-tagging](./vm-tagging)

**Quick Summary:** Applies `Environment=dev` tag to `devops-vm` via `az vm update --set tags.Environment=dev` with verification.

| | |
|---|---|
| **Purpose** | Enforce governance metadata on an untagged VM discovered during infrastructure migration |
| **Approach** | Queried VM resource group dynamically, applied tag without downtime, and confirmed via `az vm show --query tags` |
| **Outcome** | Tag confirmed; VM remained running; provisioning state `Succeeded` |

---

## Technologies and Tools

| Category | Stack |
|---|---|
| Cloud Platform | Microsoft Azure |
| CLI | Azure CLI (`az`) 2.40+ |
| Compute | Azure Virtual Machines, Azure App Service |
| OS | Ubuntu 22.04 LTS |
| Web Server | Nginx 1.18.0 |
| Storage | Azure Managed Disks (Standard_LRS) |
| Networking | Azure NSG, VNet, Standard Public IP |
| Auth | RSA 4096-bit SSH key pairs, Service Principal |
| Init | cloud-init (`--custom-data`) |
| Filesystem | ext4, `fdisk`, `mkfs.ext4`, `blkid`, `/etc/fstab` |
| Shell | Bash |

---

## Key Outcomes and Skills Demonstrated

- **Policy-constrained provisioning:** Every lab was executed against subscriptions enforcing `Standard_LRS` disk SKU and max 128 GB OS disk, mirroring real enterprise Azure Policy environments.
- **Zero-touch configuration:** cloud-init bootstrap scripts for Nginx eliminate manual post-deploy steps, directly applicable to CI/CD pipelines.
- **Failure recovery:** Documented root causes and CLI resolutions for policy blocks, immutable disk types, stuck VM states, and orphaned ghost resources.
- **Storage lifecycle management:** Full disk expansion workflow covering Azure control plane operations and in-guest Linux partitioning, formatting, and persistent mounting.
- **Security posture:** Password authentication disabled across all applicable labs; RSA key-based SSH; NSG default-deny with explicit allow rules; `nofail` fstab entries for operational resilience.
- **CLI-only validation:** All post-deployment verification performed via `az` queries and `curl`, with no portal dependency, enabling pipeline-compatible confirmation.

---

## How to Navigate

Each subdirectory contains a standalone `README.md` with:

- Full CLI commands with flag-level explanations
- Expected command output for each step
- Inline screenshots at key checkpoints
- A dedicated error and troubleshooting section where applicable
- A final validation checklist

Start with [`azure-vm-nginx-bootstrap`](./azure-vm-nginx-bootstrap) or [`secure-linux-vm-provisioning`](./secure-linux-vm-provisioning) for the most comprehensive examples of policy-aware VM deployment. For storage operations, [`vm-storage-expansion`](./vm-storage-expansion) covers the full Azure-to-OS disk management stack.

---

*Part of the [cloud-infrastructure-devops-labs](../../) portfolio.*
