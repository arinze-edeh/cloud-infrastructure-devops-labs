# Containers

> **Azure container infrastructure projects covering registry management, image lifecycle, and managed Kubernetes provisioning.**
> Work reflects production constraints: policy-restricted environments, private networking requirements, and explicit monitoring controls.

---

## Overview

This directory documents hands-on container infrastructure work performed against live Azure environments. Each project addresses a real operational requirement: establishing a secure image registry with a repeatable build-and-push pipeline, and provisioning a private, autoscaling Kubernetes cluster with full observability control.

Both projects were executed under lab policy constraints that mirror enterprise restrictions, including pre-existing resource groups, SKU limitations, and region-specific version availability. Outputs follow a documentation standard optimized for audit trails, onboarding, and portfolio review.

---

## Directory Structure

```
azure/containers/
├── acr-setup/
│   └── README.md
├── aks-private-cluster-provisioning/
│   └── README.md
└── README.md
```

---

## Project Summaries

### [ACR Setup and Docker Image Deployment](./acr-setup)

**Quick Summary:** Provisioned an Azure Container Registry (Basic SKU, East US) and executed a full Docker build-and-push pipeline for a Python application. Verified the image via repository list, tag confirmation, and manifest digest check.

| | |
|---|---|
| **Purpose** | Establish a centralized, versioned image registry with an auditable push workflow for a containerized Python workload. |
| **Approach** | Provisioned ACR via Azure CLI with explicitly set variables to eliminate inline typo risk. Built the image using the full ACR login server tag format (`<login-server>/<repository>:<tag>`), authenticated Docker using `az acr login`, and pushed with full layer verification. Completed a three-point end-to-end check: repository list, tag presence, and manifest digest. |
| **Outcome** | Image `xfusionacr29762.azurecr.io/xfusionacr29762:latest` confirmed in registry with digest `sha256:395c45bb...`. All seven layers pushed successfully. Registry provisioning state: `Succeeded`. |
| **Key Decision** | Included a documented fallback authentication path (`az acr credential show` + `docker login`) for environments where `az acr login` is blocked, covering real-world CI/CD constraints. |

---

### [Private AKS Cluster Provisioning](./aks-private-cluster-provisioning)

**Quick Summary:** Deployed a private AKS cluster (Kubernetes 1.33.0, Central US) with cluster autoscaler, a pinned node pool configuration, and explicitly disabled Azure Monitor Managed Prometheus. Resolved a silent variable failure and a monitoring audit discrepancy during execution.

| | |
|---|---|
| **Purpose** | Provision a cost-optimized, private-access Kubernetes cluster with no public API server exposure and zero monitoring overhead. |
| **Approach** | Queried available Kubernetes versions region-specifically before provisioning. Deployed with `--enable-private-cluster`, `--enable-cluster-autoscaler`, and explicit min/max node bounds in a single `az aks create` call. Post-creation, audited both `addonProfiles` and `azureMonitorProfile` separately, discovering that Managed Prometheus was silently active despite a null addon profile response. Disabled it explicitly with `--disable-azure-monitor-metrics`. |
| **Outcome** | Cluster running at `provisioningState: Succeeded` with `azureMonitorProfile.metrics.enabled: false` confirmed. Node pool `agentpool` configured with autoscaler bounds of 1 to 2 nodes on `Standard_D2s_v3`. |
| **Key Decisions** | (1) Hardcoded resource group after confirming a JMESPath region filter silently returned empty, which would have cascaded into misleading downstream failures. (2) Treated `addonProfiles: null` as insufficient evidence of full monitoring disable, requiring a separate API surface check. |

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Cloud Platform | Microsoft Azure |
| CLI | Azure CLI 2.67.0 |
| Container Runtime | Docker Engine 29.2.1 |
| Registry | Azure Container Registry (ACR), Basic SKU |
| Orchestration | Azure Kubernetes Service (AKS), Kubernetes 1.33.0 |
| Networking | Private Cluster, Azure CNI Overlay, System-Assigned Managed Identity |
| Autoscaling | AKS Cluster Autoscaler |
| Monitoring | Azure Monitor Managed Prometheus (explicitly disabled) |
| OS | Debian GNU/Linux 11, AKSUbuntu-2204gen2containerd |

---

## Key Outcomes and Skills Demonstrated

**Registry and Image Lifecycle**
- Provisioned ACR and executed a full build-tag-push pipeline with digest-level verification
- Applied the correct ACR tag convention and documented a fallback auth path for restricted CI environments

**Kubernetes Infrastructure**
- Deployed a private AKS cluster with no public API server, respecting production security posture
- Configured cluster autoscaler with explicit node bounds, pinned Kubernetes version, and correct node pool naming
- Confirmed version availability region-specifically, not from the resource group region

**Operational Troubleshooting**
- Identified and resolved a silent variable assignment failure caused by an empty JMESPath query result
- Caught and corrected a monitoring audit gap where `addonProfiles: null` masked an active Managed Prometheus deployment on a separate API surface

**Documentation Standards**
- All projects include error sections with root cause, resolution, and prevention patterns
- Verification commands are structured for both manual and scripted use
- Best practices and lessons learned are written for knowledge transfer, not narration

---

## How to Navigate

Each subdirectory contains a self-contained `README.md` with the full implementation record for that project, including:

- Problem statement and resource configuration table
- Step-by-step CLI commands with expected outputs
- Screenshot placeholders at every verification point
- Errors encountered, root causes, and resolutions
- Best practices and lessons learned

Start with the project `README.md` for execution context, then reference the command blocks directly for replication or adaptation.

---

> Part of the [`cloud-infrastructure-devops-labs`](../../) portfolio.
> Parent: [`azure/`](../)
