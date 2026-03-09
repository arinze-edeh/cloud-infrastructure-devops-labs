# Azure Container Registry (ACR) Setup and Docker Image Deployment

> **Platform:** Microsoft Azure | **Region:** East US | **Registry SKU:** Basic
> **Stack:** Azure CLI 2.67.0 | Docker 29.2.1 | Python 3.8 | Debian GNU/Linux 11

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Environment Verification](#environment-verification)
- [Implementation](#implementation)
  - [Phase 1: Azure Authentication](#phase-1-azure-authentication)
  - [Phase 2: Resource Group Identification](#phase-2-resource-group-identification)
  - [Phase 3: ACR Provisioning](#phase-3-acr-provisioning)
  - [Phase 4: Docker Authentication to ACR](#phase-4-docker-authentication-to-acr)
  - [Phase 5: Docker Image Build](#phase-5-docker-image-build)
  - [Phase 6: Image Push to ACR](#phase-6-image-push-to-acr)
  - [Phase 7: End-to-End Verification](#phase-7-end-to-end-verification)
- [Resolution Summary](#resolution-summary)
- [Repository Structure](#repository-structure)
- [References](#references)

---

## Problem Statement

The Nautilus DevOps team required a containerized application deployment pipeline with a centralized, secure image registry. The absence of a dedicated Azure Container Registry (ACR) created the following operational gaps:

- No centralized repository for storing and versioning Docker images
- No standardized image tagging strategy across environments
- No auditable, repeatable build and push workflow for containerized workloads

**Objective:** Provision an Azure Container Registry named `xfusionacr29762` under the `East US` region with a `Basic` SKU, build a Docker image from an existing Dockerfile at `/root/pyapp`, and push the image with the tag `xfusionacr29762:latest` to the newly provisioned registry.

---

## Architecture Overview

```
azure-client host
      |
      | az acr create
      v
Azure Container Registry
xfusionacr29762.azurecr.io
      ^
      | docker push
      |
Docker Build Context
/root/pyapp/
  - Dockerfile
  - app.py
  - requirements.txt
```

**Resource Details**

| Resource | Value |
|---|---|
| Registry Name | `xfusionacr29762` |
| Login Server | `xfusionacr29762.azurecr.io` |
| Resource Group | `kml_rg_main-ef0d358414644f07` |
| Location | `eastus` |
| SKU | `Basic` |
| Image Tag | `xfusionacr29762:latest` |
| Subscription | `Azure Free Labs` |

---

## Prerequisites

The following tools and access must be confirmed before execution:

| Requirement | Version Confirmed |
|---|---|
| Azure CLI | 2.67.0 |
| Docker Engine | 29.2.1 |
| Docker Compose Plugin | v5.1.0 |
| Python (on host) | 3.10.15 |
| OS | Debian GNU/Linux 11 (bullseye) |
| Architecture | x86_64 |

Access requirements:

- Active Azure subscription with permissions to create Container Registry resources
- Service principal or user credentials with `Contributor` role on the target resource group
- Dockerfile and application source present at `/root/pyapp/`

---

## Environment Verification

Before any provisioning, verify the host identity, toolchain, and source files.

```bash
# Verify host identity
hostname

# Verify Azure CLI installation
az --version

# Verify Docker installation
docker --version

# Verify Docker daemon is running
docker info

# Verify Dockerfile and application source exist
ls -la /root/pyapp/
cat /root/pyapp/Dockerfile
```

***Screenshots: Pre-flight environment verification output***
<img width="1034" height="848" alt="image" src="https://github.com/user-attachments/assets/15d7a72f-7bde-4672-b3ea-9521c7e388f1" />
<img width="1031" height="863" alt="image" src="https://github.com/user-attachments/assets/87205edb-135d-400f-845a-17f80a7438c5" />
<img width="1025" height="858" alt="image" src="https://github.com/user-attachments/assets/acd63f64-8aa5-4a34-ad76-d37c84d44afb" />
<img width="1030" height="370" alt="image" src="https://github.com/user-attachments/assets/8c4121dc-069e-4b08-a9ca-b66cc40f1258" />

**Expected Dockerfile contents:**

```dockerfile
# Sample Dockerfile
FROM python:3.8-slim
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

---

## Implementation

### Phase 1: Azure Authentication

Authenticate to Azure using the credentials associated with the lab environment.

```bash
az login -u "<USERNAME>" -p "<PASSWORD>"
```

Verify the active subscription:

```bash
az account show
az account show --query "[name, state, id]" --output table
```

***Screenshot: az account show output confirming Azure Free Labs subscription in Enabled state***
<img width="1033" height="485" alt="image" src="https://github.com/user-attachments/assets/e524a7f7-6174-409b-b5e6-15c43afb8a01" />

**Expected output:**

```
Result
------------------------------------
Azure Free Labs
Enabled
f0c3bcdd-5ce2-4fa0-8cf3-41559747512b
```

---

### Phase 2: Resource Group Identification

Do not create a new resource group. Identify the existing one and confirm it is in `eastus`.

```bash
az group list --output table
```

***Screenshot: az group list output showing kml_rg_main-ef0d358414644f07 in eastus***
<img width="1034" height="583" alt="image" src="https://github.com/user-attachments/assets/15957a51-b7a6-4d12-83b2-7c47038fff69" />

Set the variable for use in subsequent commands:

```bash
RESOURCE_GROUP="kml_rg_main-ef0d358414644f07"
echo $RESOURCE_GROUP
```

**Confirmed:**

| Name | Location | Status |
|---|---|---|
| `kml_rg_main-ef0d358414644f07` | `eastus` | `Succeeded` |

---

### Phase 3: ACR Provisioning

Set all variables explicitly before execution to eliminate inline typo risk.

```bash
ACR_NAME="xfusionacr29762"
SKU="Basic"
LOCATION="eastus"

az acr create \
  --resource-group "$RESOURCE_GROUP" \
  --name "$ACR_NAME" \
  --sku "$SKU" \
  --location "$LOCATION"
```

Verify the registry was provisioned correctly:

```bash
az acr show \
  --name "$ACR_NAME" \
  --resource-group "$RESOURCE_GROUP" \
  --output table
```

***Screenshots: az acr show table output confirming name, location, SKU, and login server***

<img width="1028" height="380" alt="image" src="https://github.com/user-attachments/assets/13ad81f9-a689-4d39-a816-378d5147a166" />

<img width="1035" height="862" alt="image" src="https://github.com/user-attachments/assets/207f692d-8ab0-4c87-b3f7-dc4d209f32a4" />
<img width="1029" height="860" alt="image" src="https://github.com/user-attachments/assets/6a2a80ba-7d8a-48b7-890c-c95dafa805b5" />
<img width="1074" height="863" alt="image" src="https://github.com/user-attachments/assets/915fc9ea-94ca-4293-a9bd-f82ff70267b5" />

**Expected verification output:**

```
NAME             RESOURCE GROUP                LOCATION    SKU    LOGIN SERVER                CREATION DATE         ADMIN ENABLED
---------------  ----------------------------  ----------  -----  --------------------------  --------------------  ---------------
xfusionacr29762  kml_rg_main-ef0d358414644f07  eastus      Basic  xfusionacr29762.azurecr.io  2026-03-09T01:38:39Z  False
```

**Resolution checkpoint:** `provisioningState: Succeeded` confirms the registry is live and reachable.

---

### Phase 4: Docker Authentication to ACR

Authenticate the local Docker daemon to the ACR login server using the active Azure CLI session.

```bash
az acr login --name "$ACR_NAME"
```

***Screenshot: az acr login output showing Login Succeeded***
<img width="1069" height="532" alt="image" src="https://github.com/user-attachments/assets/0edb9a90-27a2-449a-9904-bbb151fff3c2" />

**Expected output:**

```
Login Succeeded
```

**Fallback** (if `az acr login` is unavailable):

```bash
ACR_PASSWORD=$(az acr credential show \
  --name "$ACR_NAME" \
  --query "passwords[0].value" \
  --output tsv)

docker login "${ACR_NAME}.azurecr.io" \
  --username "$ACR_NAME" \
  --password "$ACR_PASSWORD"
```

---

### Phase 5: Docker Image Build

Navigate to the build context directory and build the image with the fully qualified ACR tag.

```bash
cd /root/pyapp
pwd
docker build -t "${ACR_NAME}.azurecr.io/${ACR_NAME}:latest" .
```

The tag format follows the ACR convention: `<login-server>/<repository>:<tag>`

Verify the image exists locally:

```bash
docker images | grep "$ACR_NAME"
```

***Screenshot: docker build output showing all 9 steps completing as FINISHED***

<img width="1075" height="764" alt="image" src="https://github.com/user-attachments/assets/813b69b5-b2a0-4b71-822a-bbeacfadaf73" />

***Screenshot: docker images output confirming the tagged image is present locally***

<img width="1073" height="851" alt="image" src="https://github.com/user-attachments/assets/510f9c6f-d3c9-45ac-b198-5deb9785aa59" />

**Build summary:**

| Step | Action | Result |
|---|---|---|
| 1/4 | Pull `python:3.8-slim` base image | 29.13 MB pulled |
| 2/4 | `COPY . /app` | Application source copied |
| 3/4 | `WORKDIR /app` | Working directory set |
| 4/4 | `RUN pip install -r requirements.txt` | Dependencies installed |
| Final | Image named to `xfusionacr29762.azurecr.io/xfusionacr29762:latest` | `sha256:79fd8cb95891` |

---

### Phase 6: Image Push to ACR

Push the locally built image to the ACR repository.

```bash
docker push "${ACR_NAME}.azurecr.io/${ACR_NAME}:latest"
```

***Screenshot: docker push output showing all 7 layers Pushed and the final digest line***
<img width="1073" height="863" alt="image" src="https://github.com/user-attachments/assets/b68b5c5f-5a64-47f4-8223-0e460f6d8300" />

**Expected output:**

```
The push refers to repository [xfusionacr29762.azurecr.io/xfusionacr29762]
1137565f6452: Pushed
5f70bf18a086: Pushed
3aa21ac95998: Pushed
d2a2207b52a4: Pushed
5d2d143f3d7f: Pushed
c3772b569c3a: Pushed
8d853c8add5d: Pushed
latest: digest: sha256:395c45bb3bc6f101e1aeee4634de0dbe9caa6df1a6201a0eb70244d113708734 size: 1783
```

**Resolution checkpoint:** The digest `sha256:395c45bb...` is the immutable identifier for this image version in the registry.

---

### Phase 7: End-to-End Verification

Run all three verification commands to confirm the full pipeline succeeded.

**1. Confirm repository is listed in ACR:**

```bash
az acr repository list \
  --name "$ACR_NAME" \
  --output table
```

**2. Confirm the `latest` tag is present:**

```bash
az acr repository show-tags \
  --name "$ACR_NAME" \
  --repository "$ACR_NAME" \
  --output table
```

**3. Confirm image manifest and digest:**

```bash
az acr repository show \
  --name "$ACR_NAME" \
  --image "${ACR_NAME}:latest"
```

***Screenshot: All three verification commands and their outputs in sequence***
<img width="1065" height="856" alt="image" src="https://github.com/user-attachments/assets/ad00498d-1b52-4e7e-8e73-4f02074f4913" />

**Expected manifest output:**

```json
{
  "changeableAttributes": {
    "deleteEnabled": true,
    "listEnabled": true,
    "readEnabled": true,
    "writeEnabled": true
  },
  "createdTime": "2026-03-09T01:44:32.300161Z",
  "digest": "sha256:395c45bb3bc6f101e1aeee4634de0dbe9caa6df1a6201a0eb70244d113708734",
  "lastUpdateTime": "2026-03-09T01:44:32.300161Z",
  "name": "latest",
  "signed": false
}
```

---

## Resolution Summary

All objectives defined in the problem statement were met in full.

| Phase | Action | Outcome |
|---|---|---|
| Environment Verification | Confirmed host, CLI, Docker, and source files | All checks passed |
| Azure Authentication | Logged in via service principal | Subscription `Azure Free Labs` active |
| Resource Group | Identified `kml_rg_main-ef0d358414644f07` | Confirmed `eastus` |
| ACR Provisioning | Created `xfusionacr29762` with `Basic` SKU | `provisioningState: Succeeded` |
| Docker Auth | Authenticated Docker to `xfusionacr29762.azurecr.io` | `Login Succeeded` |
| Image Build | Built `xfusionacr29762.azurecr.io/xfusionacr29762:latest` | `sha256:79fd8cb95891`, 131 MB |
| Image Push | Pushed all 7 layers to ACR | Digest `sha256:395c45bb...` confirmed |
| Verification | Repository list, tag list, manifest check | All three passed |

The image `xfusionacr29762.azurecr.io/xfusionacr29762:latest` is now stored in Azure Container Registry and is available for deployment to any Azure compute service including AKS, ACI, or App Service.

---

## Repository Structure

```
azure/
└── containers/
    └── acr-setup.md       <- This document
```

---

## References

- [Azure Container Registry documentation](https://learn.microsoft.com/en-us/azure/container-registry/)
- [az acr CLI reference](https://learn.microsoft.com/en-us/cli/azure/acr)
- [Docker push reference](https://docs.docker.com/reference/cli/docker/image/push/)
- [ACR authentication options](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-authentication)








<img width="1069" height="610" alt="image" src="https://github.com/user-attachments/assets/1e54354c-3206-4c32-9fef-28abcffa845e" />


