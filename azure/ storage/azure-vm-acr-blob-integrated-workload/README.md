# Azure VM Containerized Deployment with ACR and Blob Storage

> **DevOps Implementation** | Containerized Python Flask App on Azure IaaS with Private Container Registry and Blob-Backed Configuration Management

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation Walkthrough](#implementation-walkthrough)
  - [Phase 1 -- Azure Virtual Machine Setup](#phase-1----azure-virtual-machine-setup)
  - [Phase 2 -- Azure Container Registry Setup](#phase-2----azure-container-registry-setup)
  - [Phase 3 -- Azure Blob Storage and Config Management](#phase-3----azure-blob-storage-and-config-management)
  - [Phase 4 -- Remote VM Provisioning](#phase-4----remote-vm-provisioning)
  - [Phase 5 -- Container Deployment on VM](#phase-5----container-deployment-on-vm)
  - [Phase 6 -- Validation and Incident Resolution](#phase-6----validation-and-incident-resolution)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Resource Summary](#resource-summary)

---

## Overview

This project demonstrates an end-to-end enterprise level containerized deployment pipeline on Microsoft Azure using CLI-driven Infrastructure as Code (IaC) principles. A Python Flask application is containerized, pushed to a private Azure Container Registry (ACR), deployed onto an Azure Virtual Machine (VM), and configured dynamically via a config file retrieved from Azure Blob Storage.

**Core Problem Solved:** Eliminate hardcoded configuration inside container images by externalizing runtime config to Azure Blob Storage, enabling environment-specific deployments without image rebuilds.

**Tech Stack:**
- **Compute:** Azure VM (Ubuntu 22.04 LTS, Standard_B1s)
- **Container Registry:** Azure Container Registry (ACR) -- Basic SKU
- **Storage:** Azure Blob Storage (Standard_LRS, StorageV2)
- **Application Runtime:** Python 3.9 + Flask (Docker containerized)
- **Tooling:** Azure CLI, Docker, SSH

---

## Architecture

```
+---------------------+        push image         +---------------------------+
|   Local Dev Machine |  -----------------------> |  Azure Container Registry  |
|  (Docker + AZ CLI)  |                           |   datacenteracr15620       |
+---------------------+                           +---------------------------+
         |                                                    |
         | upload config.json                                 | pull image
         v                                                    v
+---------------------+                           +---------------------------+
|  Azure Blob Storage |                           |     Azure Virtual Machine  |
|  datacenterstor15620|  <-- az blob download --- |   datacenter-vm (Ubuntu)   |
|  [datacenter-config]|                           |   Port 80 open (NSG)       |
+---------------------+                           |   Docker runtime           |
                                                  |   python-app container     |
                                                  +---------------------------+
                                                            |
                                                  Public IP: 138.91.112.58
                                                  HTTP Response: 200 OK
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure Subscription | Active subscription with Contributor access |
| Azure CLI | Installed and authenticated (`az login` or Service Principal) |
| Docker Engine | Installed locally for image build and push |
| SSH Key Pair | RSA 4096-bit generated locally |
| Python App Source | Flask app under `/root/pyapp/` with `Dockerfile` |
| Config File | `/root/config.json` for runtime configuration |

---

## Implementation Walkthrough

---

### Phase 1 -- Azure Virtual Machine Setup

#### 1.1 Verify Active Azure Subscription

Confirm the authenticated session and target subscription before provisioning any resources.

```bash
az account show
```

**Expected Output:**
```json
{
  "name": "Azure Free Labs",
  "state": "Enabled",
  "user": {
    "type": "servicePrincipal"
  }
}
```

> **Note:** Authentication is via Service Principal (`type: servicePrincipal`). Ensure the principal has at minimum Contributor role on the target resource group.

---

#### 1.2 Generate SSH Key Pair

Generate a 4096-bit RSA key pair for passwordless VM authentication.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

**Expected Output:**
```
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
SHA256:9Z549iiHRe2oexc7Ped6H4gYWLf8L7BUHej7Gw0rqLw root@azure-client
```

> **Security Note:** The `-N ""` flag sets an empty passphrase for automation contexts. In production, always protect private keys with a strong passphrase and use an SSH agent.

**SCREENSHOTS** 

<img width="1060" height="851" alt="image" src="https://github.com/user-attachments/assets/e7239d8e-2aa9-473c-8b52-fb398bba1677" />
<img width="1061" height="836" alt="image" src="https://github.com/user-attachments/assets/37f8cfa9-830d-4d31-95be-d318b3dc2a67" />

>Terminal: `ssh-keygen` output showing key saved to `/root/.ssh/id_rsa`, SHA256 fingerprint, and randomart image

---

#### 1.3 Provision the Azure Virtual Machine

Create the VM dynamically resolving the resource group using a CLI query to avoid hardcoding group names.

```bash
az vm create \
  --resource-group $(az group list --query "[0].name" -o tsv) \
  --name datacenter-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --authentication-type ssh \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --location eastus \
  --public-ip-sku Standard \
  --storage-sku Standard_LRS \
  --os-disk-size-gb 30
```

**Key Parameters:**

| Parameter | Value | Reason |
|---|---|---|
| `--image` | Ubuntu2204 | LTS, stable, widely supported |
| `--size` | Standard_B1s | Cost-optimized burstable for dev/lab |
| `--authentication-type` | ssh | Eliminates password-based attack surface |
| `--public-ip-sku` | Standard | Required for zone-redundant deployments |
| `--storage-sku` | Standard_LRS | Locally redundant, cost-effective for lab |

**Expected VM Provisioning Result:**
```json
{
  "publicIpAddress": "138.91.112.58",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4"
}
```

**SCREENSHOT** 

<img width="1061" height="836" alt="image" src="https://github.com/user-attachments/assets/6913f9ad-1c47-42c5-81ce-a5dd9c8860ce" />

>Terminal: `az vm create` JSON output showing `"powerState": "VM running"`, `"publicIpAddress": "138.91.112.58"`, and `"privateIpAddress": "10.0.0.4"`

---

#### 1.4 Open Port 80 on the Network Security Group (NSG)

Allow inbound HTTP traffic to enable external access to the containerized application.

```bash
az vm open-port \
  --resource-group $(az group list --query "[0].name" -o tsv) \
  --name datacenter-vm \
  --port 80
```

**Resulting NSG Security Rules (Custom):**

| Rule Name | Priority | Port | Direction | Access |
|---|---|---|---|---|
| `default-allow-ssh` | 1000 | 22 | Inbound | Allow |
| `open-port-80` | 900 | 80 | Inbound | Allow |

> **Note:** The `open-port-80` rule is assigned priority 900, higher precedence than the SSH rule at 1000. Lower number = higher precedence in Azure NSG evaluation.

**SCREENSHOTS**

<img width="1061" height="862" alt="image" src="https://github.com/user-attachments/assets/11112c7b-75c0-4db4-a975-8e0c0c3f9c06" />
<img width="1062" height="860" alt="image" src="https://github.com/user-attachments/assets/d3c3760c-e166-4cf8-82dd-25ec87360ac2" />
<img width="1062" height="866" alt="image" src="https://github.com/user-attachments/assets/a1c157b7-333c-4e95-a3a2-81816e3df13e" />
<img width="1060" height="861" alt="image" src="https://github.com/user-attachments/assets/7ecad713-ab2a-4cc8-b87b-fa374c7c77b1" />
<img width="1060" height="857" alt="image" src="https://github.com/user-attachments/assets/656f6240-994d-4bfa-88b2-4411af745209" />

>Terminal: `az vm open-port` JSON output showing `securityRules` array with `open-port-80` (priority 900, port 80) and `default-allow-ssh` (priority 1000, port 22) both with `"access": "Allow"`

---

### Phase 2 -- Azure Container Registry Setup

#### 2.1 Create the Azure Container Registry

Provision a Basic SKU ACR with admin user enabled for credential-based Docker login.

```bash
az acr create \
  --resource-group $(az group list --query "[0].name" -o tsv) \
  --name datacenteracr15620 \
  --sku Basic \
  --location eastus \
  --admin-enabled true
```

**Key Output Fields:**
```json
{
  "loginServer": "datacenteracr15620.azurecr.io",
  "adminUserEnabled": true,
  "provisioningState": "Succeeded"
}
```

> **Security Note:** `--admin-enabled true` is acceptable for lab environments. In production, prefer Azure AD-based authentication using managed identities or service principals scoped with `AcrPull` role. Disable admin credentials post-deployment.

**SCREENSHOTS**

<img width="1062" height="866" alt="image" src="https://github.com/user-attachments/assets/e6acf9c3-2e65-498c-98b1-40c0d526edec" />
<img width="1053" height="862" alt="image" src="https://github.com/user-attachments/assets/bcca578a-369a-44d4-8975-9fe97f4e8fa1" />

>Terminal: `az acr create` JSON output showing `"loginServer": "datacenteracr15620.azurecr.io"`, `"adminUserEnabled": true`, and `"provisioningState": "Succeeded"`
---

#### 2.2 Build, Tag, and Push the Docker Image

Navigate to the application directory, authenticate to ACR, build the image, apply the remote registry tag, and push.

```bash
cd /root/pyapp

# Step 1: Authenticate Docker to ACR
az acr login --name datacenteracr15620

# Step 2: Build the image locally
docker build -t datacenter/python-app:latest .

# Step 3: Tag with the full ACR registry path
docker tag datacenter/python-app:latest \
  datacenteracr15620.azurecr.io/datacenter/python-app:latest

# Step 4: Push to ACR
docker push datacenteracr15620.azurecr.io/datacenter/python-app:latest
```

**Build Summary:**
- Base image: `python:3.9-slim`
- Layers built: 4 (WORKDIR, COPY app.py, RUN pip install flask, FROM)
- Total build time: ~12.7 seconds
- Final image digest: `sha256:cef0472130d9a5308e7714a2aeecb1ed77c1e3f557d79a9996c690372ec366a0`

**SCREENSHOTS** 

<img width="1064" height="863" alt="image" src="https://github.com/user-attachments/assets/2652be4a-66c3-4e87-8a46-403ae10825b5" />
<img width="1061" height="852" alt="image" src="https://github.com/user-attachments/assets/7f8bc9f6-9d09-4597-b3a9-a28af92a03f4" />

> Terminal: `docker build` output showing all 9 build steps with `FINISHED` status, `docker:default` backend, base image `python:3.9-slim` resolved, and final line `naming to docker.io/datacenter/python-app:latest`, and `docker push` output showing all 7 layers with `Pushed` status and the final digest line `latest: digest: sha256:cef0472130d9a5308e7714a2aeecb1ed77c1e3f557d79a9996c690372ec366a0 size: 1783`

---

#### 2.3 Verify Image in ACR

Confirm the image tag is present in the registry before deploying to the VM.

```bash
az acr repository show-tags \
  --name datacenteracr15620 \
  --repository datacenter/python-app \
  --output table
```

**Expected Output:**
```
Result
--------
latest
```

**SCREENSHOT**

<img width="1061" height="866" alt="image" src="https://github.com/user-attachments/assets/4f520840-7a7b-4db3-9f3e-e64f1e924acd" />

>Terminal: `az acr repository show-tags` table output showing `Result / -------- / latest`

---

### Phase 3 -- Azure Blob Storage and Config Management

#### 3.1 Create the Storage Account

Provision a StorageV2 account with locally redundant storage in East US.

```bash
az storage account create \
  --name datacenterstor15620 \
  --resource-group $(az group list --query "[0].name" -o tsv) \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

---

#### 3.2 Export the Storage Connection String

Retrieve and export the connection string as an environment variable for subsequent CLI operations.

```bash
export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string \
  --name datacenterstor15620 \
  --resource-group $(az group list --query "[0].name" -o tsv) \
  --query connectionString \
  --output tsv)
```

> **Security Note:** Exporting connection strings to shell environment variables is acceptable for ephemeral automation sessions. In production, use Azure Key Vault references or Managed Identity with RBAC instead of connection string auth.

---

#### 3.3 Create the Blob Container

Create the private container that will hold runtime configuration files.

```bash
az storage container create \
  --name datacenter-config \
  --connection-string "$AZURE_STORAGE_CONNECTION_STRING"
```

**Expected Output:**
```json
{ "created": true }
```

---

#### 3.4 Upload Configuration File to Blob Storage

Upload `config.json` from the local filesystem to the `datacenter-config` container.

```bash
az storage blob upload \
  --container-name datacenter-config \
  --file /root/config.json \
  --name config.json \
  --connection-string "$AZURE_STORAGE_CONNECTION_STRING"
```

**Expected Output:**
```json
{
  "lastModified": "2026-03-29T02:43:29+00:00",
  "request_server_encrypted": true
}
```

**SCREENSHOTS** 

<img width="1057" height="854" alt="image" src="https://github.com/user-attachments/assets/d4b2abfa-f0f6-4c84-bcf7-79c573648670" />
<img width="1061" height="859" alt="image" src="https://github.com/user-attachments/assets/c11b4236-0d6e-43b9-b114-5eba3fc8fd1a" />
<img width="1058" height="867" alt="image" src="https://github.com/user-attachments/assets/d4f2945f-c35d-4bcf-b92f-969bd5d655af" />
<img width="1064" height="866" alt="image" src="https://github.com/user-attachments/assets/26f348c0-586f-4b06-8a0e-9f1004dc3a05" />

>Terminal: `az storage account create` JSON output showing `"name": "datacenterstor15620"`, `"sku": {"name": "Standard_LRS"}`, `"kind": "StorageV2"`, and `"provisioningState": "Succeeded"`

>`az storage blob upload` JSON output showing progress bar at 100%, `"lastModified"`, `"request_server_encrypted": true`, and `"content_md5"` fields confirming successful upload

---

### Phase 4 -- Remote VM Provisioning

#### 4.1 Install Docker and Azure CLI on the VM via SSH

Remotely provision the VM with all runtime dependencies in a single SSH session.

```bash
# Capture the ACR admin password for later use
ACR_PASSWORD=$(az acr credential show \
  --name datacenteracr15620 \
  --query "passwords[0].value" -o tsv)

PUBLIC_IP="138.91.112.58"

ssh -o StrictHostKeyChecking=no azureuser@$PUBLIC_IP '
  sudo apt-get update -y
  sudo apt-get install -y docker.io
  sudo systemctl start docker
  sudo systemctl enable docker
  curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
'
```

**Packages Installed on VM:**

| Package | Version | Purpose |
|---|---|---|
| docker.io | 28.2.2 | Container runtime |
| containerd | 1.7.28 | Container execution layer |
| azure-cli | 2.84.0 | Blob Storage access from VM |
| runc | 1.3.3 | OCI container runtime |

> **Note:** `StrictHostKeyChecking=no` is used here for first-time connection automation. This bypasses host key verification and should not be used in production. Instead, pre-populate `~/.ssh/known_hosts` with the host fingerprint.

**SCREENSHOTS**

<img width="1054" height="861" alt="image" src="https://github.com/user-attachments/assets/f82c38ad-1022-4ab5-b82a-338884f8d7a8" />
<img width="1061" height="863" alt="image" src="https://github.com/user-attachments/assets/1f3e54be-4c51-4e50-8a02-be677ee80fbc" />
<img width="1050" height="862" alt="image" src="https://github.com/user-attachments/assets/9dbfada9-da6c-4fb3-b2b0-04b88aa34b91" />
<img width="1063" height="869" alt="image" src="https://github.com/user-attachments/assets/8d7b40a7-73ea-40cd-8221-9600476d8f5a" />
<img width="1059" height="855" alt="image" src="https://github.com/user-attachments/assets/561bab44-6d0e-4e5b-bb34-30b7125c9329" />
<img width="1061" height="846" alt="image" src="https://github.com/user-attachments/assets/8e56d0ee-50cb-4f99-b06b-8c6f01301bb5" />


>Terminal: SSH session output showing `docker.io 28.2.2` and `azure-cli 2.84.0` packages installed, systemctl docker enabled, and `No services need to be restarted` confirmation at session end

---

### Phase 5 -- Container Deployment on VM

#### 5.1 Login to ACR and Pull the Image on the VM

Authenticate Docker on the remote VM to the ACR and pull the application image.

```bash
ssh azureuser@$PUBLIC_IP "
  sudo docker login datacenteracr15620.azurecr.io \
    --username datacenteracr15620 \
    --password '$ACR_PASSWORD'

  sudo docker pull datacenteracr15620.azurecr.io/datacenter/python-app:latest

  sudo docker run -d \
    --name python-app \
    -p 80:80 \
    datacenteracr15620.azurecr.io/datacenter/python-app:latest
"
```

**Expected Pull Output:**
```
Status: Downloaded newer image for datacenteracr15620.azurecr.io/datacenter/python-app:latest
```

**Expected Container State:**
```
CONTAINER ID   IMAGE                                                         COMMAND           STATUS
b5e121df1922   datacenteracr15620.azurecr.io/datacenter/python-app:latest   "python app.py"   Up About a minute
```

**SCREENSHOTS**

<img width="1055" height="841" alt="image" src="https://github.com/user-attachments/assets/2e352f9c-fcb5-4b80-a41c-52f205438585" />
<img width="1065" height="848" alt="image" src="https://github.com/user-attachments/assets/0a5520b7-b56e-4614-9507-8bff8cbca8ed" />

<img width="1060" height="382" alt="image" src="https://github.com/user-attachments/assets/1de40005-1fd3-41d8-a3f7-d9a43803ee2a" />

>Terminal: `sudo docker ps` output showing container ID `b5e121df1922`, image `datacenteracr15620.azurecr.io/datacenter/python-app:latest`, command `"python app.py"`, status `Up About a minute`, and ports `0.0.0.0:80->80/tcp`

---

### Phase 6 -- Validation and Incident Resolution

#### 6.1 Initial Health Check -- HTTP 500 Error Detected

After the container started, a health check via `curl` returned an unexpected error.

```bash
curl -I http://$PUBLIC_IP
```

**Actual Response (FAILURE):**
```
HTTP/1.1 500 INTERNAL SERVER ERROR
Server: Werkzeug/3.1.7 Python/3.9.25
```

> This is an **application-level failure**, not an infrastructure failure. The VM, NSG, Docker, and network were all functioning correctly. The root cause required container log inspection.

**SCREENSHOT**

<img width="1061" height="421" alt="image" src="https://github.com/user-attachments/assets/6a146f3f-424e-4772-8eff-f987638e1488" />

>Terminal: `curl -I http://138.91.112.58` output showing `HTTP/1.1 500 INTERNAL SERVER ERROR`, `Server: Werkzeug/3.1.7 Python/3.9.25`, and `Connection: close`

---

#### 6.2 Root Cause Analysis -- Container Log Inspection

```bash
ssh azureuser@$PUBLIC_IP "sudo docker logs python-app"
```

**Container Log Output (Error):**
```
[ERROR] Exception on / [HEAD]
Traceback (most recent call last):
  File "/app/app.py", line 8, in home
    with open("config.json", "r") as f:
FileNotFoundError: [Errno 2] No such file or directory: 'config.json'
```

**Root Cause:** The Flask application attempts to read `config.json` from its working directory (`/app/`) at runtime. The file was uploaded to Azure Blob Storage but was never mounted or copied into the running container. The container image itself did not include `config.json` (correctly, by design -- config should not be baked into images).

**[SCREENSHOT PLACEHOLDER -- Terminal showing docker logs python-app with the FileNotFoundError traceback visible]**

---

#### 6.3 Resolution -- Download Config from Blob and Mount into Container

The fix involves three steps: download the config file from Blob Storage onto the VM host, stop and remove the broken container, and relaunch with a volume mount binding the config into the container's expected path.

```bash
ssh azureuser@$PUBLIC_IP "
  # Step 1: Download config.json from Azure Blob Storage to VM host
  az storage blob download \
    --container-name datacenter-config \
    --name config.json \
    --file /home/azureuser/config.json \
    --connection-string '$AZURE_STORAGE_CONNECTION_STRING'

  # Step 2: Stop and remove the broken container
  sudo docker stop python-app
  sudo docker rm python-app

  # Step 3: Relaunch with config mounted into /app/config.json
  sudo docker run -d \
    --name python-app \
    -p 80:80 \
    -v /home/azureuser/config.json:/app/config.json \
    datacenteracr15620.azurecr.io/datacenter/python-app:latest
"
```

**Key Fix:** The `-v /home/azureuser/config.json:/app/config.json` flag mounts the host-side file directly into the container filesystem at the exact path the application expects.

**[SCREENSHOT PLACEHOLDER -- Terminal showing docker stop, docker rm, and docker run -v commands executing successfully with new container ID returned]**

---

#### 6.4 Post-Fix Validation -- HTTP 200 OK Confirmed

```bash
ssh azureuser@$PUBLIC_IP "sudo docker ps"
```

```
CONTAINER ID   IMAGE                                                         STATUS              PORTS
549d2ead609d   datacenteracr15620.azurecr.io/datacenter/python-app:latest   Up About a minute   0.0.0.0:80->80/tcp
```

```bash
curl -I http://$PUBLIC_IP
```

**Final Response (SUCCESS):**
```
HTTP/1.1 200 OK
Server: Werkzeug/3.1.7 Python/3.9.25
Content-Type: text/html; charset=utf-8
Content-Length: 57
```

**[SCREENSHOT PLACEHOLDER -- Terminal showing curl -I returning HTTP 200 OK with Werkzeug/Python server headers]**

**[SCREENSHOT PLACEHOLDER -- Browser screenshot of the Flask application responding at http://138.91.112.58 showing the application page loaded]**

---

## Errors Encountered and Resolutions

| # | Error | Root Cause | Resolution |
|---|---|---|---|
| 1 | `HTTP/1.1 500 INTERNAL SERVER ERROR` on initial deploy | `config.json` not present inside the running container at `/app/config.json` | Downloaded config from Azure Blob Storage to VM host; relaunched container with `-v` volume mount binding host file into container path |
| 2 | `WARNING: Using --password via the CLI is insecure` on docker login | ACR password passed as a plain CLI argument, visible in shell history and process list | Acceptable for lab use. Production fix: pipe password via `--password-stdin` using `echo "$ACR_PASSWORD" \| sudo docker login ... --password-stdin` |
| 3 | `debconf: unable to initialize frontend: Dialog` during apt-get over SSH | Non-interactive SSH session has no TTY for debconf dialog frontend | Non-blocking warning; apt-get falls back to Teletype frontend automatically. For clean suppression use `DEBIAN_FRONTEND=noninteractive` env var |

---

## Best Practices Applied

### Security

* SSH key-based authentication used instead of passwords, eliminating brute-force attack surface on port 22.
* ACR admin credentials retrieved dynamically via `az acr credential show` rather than hardcoded in scripts.
* Azure Blob Storage HTTPS-only traffic enforced (`enableHttpsTrafficOnly: true` confirmed in provisioning output).
* Server-side encryption enabled on all blob data (`request_server_encrypted: true`).
* Configuration data externalized from the container image -- secrets and config are never baked into Docker layers.

### Infrastructure as Code Patterns

* Resource group name resolved dynamically via `$(az group list --query "[0].name" -o tsv)` -- no hardcoded group names in any command.
* All resources provisioned in the same region (`eastus`) to eliminate cross-region latency and egress charges.
* ACR and storage account names include unique numeric suffixes (`15620`) to ensure global namespace uniqueness.

### Container Hygiene

* Lightweight base image (`python:3.9-slim`) used to minimize attack surface and image size.
* Container runs on port 80 with explicit `-p 80:80` host-to-container port mapping.
* Container named explicitly (`--name python-app`) for deterministic management commands.
* Old containers stopped and removed before redeployment to prevent port conflicts (`docker stop` + `docker rm` before `docker run`).

### Operational Discipline

* Container logs inspected (`docker logs`) before making any changes -- root cause confirmed before resolution applied.
* Image digest verified post-push (`sha256:cef0472...`) providing an immutable reference for auditability.
* `docker ps` verified after each deployment to confirm container is in `Up` state.
* End-to-end HTTP validation performed with `curl -I` confirming 200 OK at the public IP.

---

## Lessons Learned

### 1. Externalized Config Requires a Runtime Delivery Mechanism

Externalizing configuration from Docker images is correct practice -- it enables the same image to be deployed across environments. However, the config must be actively delivered to the runtime environment. In this case, the gap between uploading to Blob Storage and making it available to the container was not bridged in the initial deployment. The fix (download + volume mount) works for single-VM deployments. At scale, consider Azure App Configuration, Kubernetes ConfigMaps, or mounting Azure Files shares directly.

### 2. Container Logs Are the First Diagnostic Tool

The `HTTP 500` response gave no application-level detail. The `docker logs` command immediately surfaced the exact exception, file path, and line number. Never rely solely on HTTP status codes for diagnosis -- always correlate with container logs, especially in non-orchestrated single-container deployments.

### 3. `--password-stdin` Over Inline Passwords

Passing Docker registry passwords inline (`--password '$ACR_PASSWORD'`) exposes credentials in shell history, process listings, and audit logs. The correct pattern is:

```bash
echo "$ACR_PASSWORD" | sudo docker login datacenteracr15620.azurecr.io \
  --username datacenteracr15620 \
  --password-stdin
```

### 4. Production Workloads Require a WSGI Server

The Flask development server (`Werkzeug`) was used in this deployment. The container logs explicitly warned: `WARNING: This is a development server. Do not use it in a production deployment.` For production, replace with `gunicorn` or `uvicorn`:

```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:80", "--workers", "4", "app:app"]
```

### 5. `StrictHostKeyChecking=no` Is a Lab-Only Pattern

Disabling host key checking eliminates protection against man-in-the-middle attacks during SSH connection establishment. In production CI/CD pipelines, pre-populate the known_hosts file with the VM's host fingerprint retrieved immediately post-provisioning:

```bash
ssh-keyscan -H $PUBLIC_IP >> ~/.ssh/known_hosts
```

### 6. Admin-Enabled ACR Should Be Disabled Post-Lab

The `--admin-enabled true` flag on ACR creates shared username/password credentials. For production, assign the `AcrPull` role to the VM's managed identity and remove admin credentials entirely, enabling credential-free image pulls via `az acr login` backed by Azure AD.

---

## Resource Summary

| Resource | Name | SKU / Size | Region |
|---|---|---|---|
| Virtual Machine | datacenter-vm | Standard_B1s | East US |
| OS Image | Ubuntu 22.04 LTS | -- | -- |
| OS Disk | -- | Standard_LRS, 30 GB | East US |
| Public IP | datacenter-vmPublicIP | Standard SKU | East US |
| NSG | datacenter-vmNSG | -- | East US |
| Container Registry | datacenteracr15620 | Basic | East US |
| Storage Account | datacenterstor15620 | Standard_LRS, StorageV2 | East US |
| Blob Container | datacenter-config | Private | -- |
| Docker Image | datacenter/python-app | latest | ACR hosted |

---











<img width="1058" height="666" alt="image" src="https://github.com/user-attachments/assets/f7ed9eb5-bda9-4191-8e41-6cf60a48b7c2" />
<img width="1062" height="864" alt="image" src="https://github.com/user-attachments/assets/1fccc4a7-dc91-4891-b4a9-eea7d9236728" />
<img width="1063" height="855" alt="image" src="https://github.com/user-attachments/assets/1b9aec67-5aed-4230-b9e9-ca6cee57bfaa" />
<img width="1060" height="553" alt="image" src="https://github.com/user-attachments/assets/fda04578-546a-4912-80eb-6f528fe5375f" />
<img width="1059" height="553" alt="image" src="https://github.com/user-attachments/assets/3bc78e07-76c5-437f-980f-1e8d710489e9" />
<img width="1059" height="507" alt="image" src="https://github.com/user-attachments/assets/1233e566-6999-4f52-aa7e-895a5309acc9" />
