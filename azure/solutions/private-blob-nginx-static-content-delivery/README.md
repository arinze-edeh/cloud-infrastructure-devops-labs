# Static Web Hosting on Azure (VNet + Blob Storage + Nginx + VM)

> **Enterprise-style static web application deployment on Microsoft Azure using a hardened Virtual Network, Azure Blob Storage as a secure content repository, and an Nginx-powered Ubuntu VM as the web server.**

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Overview](#solution-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Environment Bootstrap](#phase-1-environment-bootstrap)
  - [Phase 2: Network Infrastructure](#phase-2-network-infrastructure)
  - [Phase 3: Azure Blob Storage Setup](#phase-3-azure-blob-storage-setup)
  - [Phase 4: Virtual Machine Provisioning](#phase-4-virtual-machine-provisioning)
  - [Phase 5: Web Server Configuration and Content Deployment](#phase-5-web-server-configuration-and-content-deployment)
  - [Phase 6: End-to-End Verification](#phase-6-end-to-end-verification)
- [Best Practices Applied](#best-practices-applied)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Lessons Learned](#lessons-learned)
- [Resource Inventory](#resource-inventory)

---

## Problem Statement

The Nautilus DevOps team required a secure, repeatable, and independently deployable pipeline for hosting a static web application on Azure. The primary constraints were:

* The static asset (`index.html`) must **not** reside in the application source code repository, as it may contain configuration or presentation logic that must be managed and distributed independently.
* The VM must **securely fetch** the content directly from Azure Blob Storage at deployment time rather than relying on a mounted storage volume or the Azure Static Website feature.
* All resources must follow production best practices for **security, performance, and accessibility**.
* Public blob access must be **explicitly disabled** to prevent unauthorized retrieval of storage objects.

---

## Solution Overview

The solution establishes a layered architecture:

1. A **Virtual Network (VNet) with a dedicated subnet** isolates compute resources and forms the network boundary.
2. An **Azure Storage Account** with public blob access disabled acts as the secure, centralized content repository. The `index.html` file is uploaded to a private Blob container.
3. An **Ubuntu 22.04 LTS Virtual Machine** running **Nginx** is provisioned inside the VNet. The VM authenticates to Blob Storage using a storage account key and downloads the `index.html` directly to Nginx's web root at deployment time.
4. A **Network Security Group (NSG)** controls inbound traffic, permitting only SSH (port 22) and HTTP (port 80).

This pattern decouples the static asset lifecycle from the application codebase while preserving full auditability, reproducibility, and security.

---

## Architecture

```
Internet
    |
    | HTTP (port 80) / SSH (port 22)
    |
[NSG: nautilus-vmNSG]
    |
[Public IP: 40.87.17.28]
    |
[VM: nautilus-vm (Ubuntu 22.04, Standard_B1s)]
    |   - Nginx serving /var/www/html/index.html
    |   - index.html fetched from Blob Storage at deploy time
    |
[VNet: nautilus-vnet (10.0.0.0/16)]
    |
[Subnet: nautilus-subnet (10.0.0.0/24)]

[Storage Account: nautilusstor6471]  <-- Private, no public blob access
    |
[Container: nautilus-container]
    |
[Blob: index.html (233 bytes)]
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Authenticated session with a valid subscription |
| Subscription | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` (Azure Free Labs) |
| Resource Group | Pre-existing; discovered dynamically via `az group list` |
| Local static asset | `/root/index.html` present on the Azure client host |
| SSH tooling | `ssh-keygen` available on the client host |

---

## Implementation Guide

### Phase 1: Environment Bootstrap

**Objective:** Confirm the Azure CLI session is active and the target resource group is identified. Generate an SSH key pair for VM authentication.

**1.1 Verify Azure CLI Authentication**

```bash
az account show
```

*Expected output confirms the active subscription, tenant, and service principal identity.*

**1.2 Verify the Static Asset Exists on the Client Host**

```bash
ls -lh /root/index.html
```

*Confirms the 233-byte `index.html` is present and readable before any upload is attempted.*

**1.3 Generate an RSA 4096-bit SSH Key Pair**

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

* `-t rsa -b 4096`: Selects RSA with a 4096-bit key length for strong asymmetric encryption.
* `-f ~/.ssh/id_rsa`: Writes the private key to the default SSH identity path.
* `-N ""`: Sets an empty passphrase for non-interactive automation; in production, a passphrase or a hardware-backed key is preferred.

The public key at `~/.ssh/id_rsa.pub` will be injected into the VM at provisioning time.

Screenshot: 

<img width="1033" height="786" alt="image" src="https://github.com/user-attachments/assets/18cefc47-6a93-483d-b528-f334b94d0ac3" />

---

### Phase 2: Network Infrastructure

**Objective:** Create the VNet and subnet that will host the VM, providing network isolation and a defined address space.

**2.1 Resolve the Resource Group Name Dynamically**

```bash
RG=$(az group list --query "[0].name" -o tsv)
echo "Resource group: $RG"
```

This avoids hardcoding the resource group name, making the script portable across lab environments where the group name is generated dynamically.

*Resolved value:* `kml_rg_main-534f47daa771482e`

**2.2 Create the Virtual Network**

```bash
az network vnet create \
  --name nautilus-vnet \
  --resource-group $RG \
  --location eastus \
  --address-prefix 10.0.0.0/16
```

* `10.0.0.0/16` provides 65,536 addresses, giving headroom for future subnet segmentation.
* `eastus` is selected for proximity and service availability alignment.

Screenshot: 

<img width="1037" height="862" alt="image" src="https://github.com/user-attachments/assets/b1ae8297-df8a-4d82-8b94-2788daa11d8e" />

**2.3 Create the Subnet**

```bash
az network vnet subnet create \
  --name nautilus-subnet \
  --resource-group $RG \
  --vnet-name nautilus-vnet \
  --address-prefix 10.0.0.0/24
```

* `10.0.0.0/24` carves out a 256-address subnet for VM workloads.

**2.4 Verify VNet and Subnet Provisioning**

```bash
az network vnet show \
  --name nautilus-vnet \
  --resource-group $RG \
  --query "{vnet:name, subnets:subnets[].name}" \
  -o table
```

Screenshot: 



---

### Phase 3: Azure Blob Storage Setup

**Objective:** Provision a private storage account, create a blob container, upload the static asset, and capture the storage access key for later use on the VM.

**3.1 Create the Storage Account**

```bash
az storage account create \
  --name nautilusstor6471 \
  --resource-group $RG \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access false
```

* `Standard_LRS`: Locally Redundant Storage provides three synchronous copies within a single region, appropriate for this workload.
* `--allow-blob-public-access false`: Critical security control. Ensures no blob in any container within this account can be exposed publicly, even if a container's access level is misconfigured.

**3.2 Verify Public Blob Access is Disabled**

```bash
az storage account show \
  --name nautilusstor6471 \
  --resource-group $RG \
  --query "allowBlobPublicAccess" \
  -o tsv
```

*Expected output: `false`*

Screenshot Placeholder: `05-storage-account-public-access-false.png`

**3.3 Capture the Storage Account Key**

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name nautilusstor6471 \
  --resource-group $RG \
  --query "[0].value" \
  -o tsv)
echo "Key captured: ${STORAGE_KEY:0:8}..."
```

Only the first 8 characters are echoed to confirm capture without exposing the full key in terminal output or logs.

**3.4 Create the Blob Container**

```bash
az storage container create \
  --name nautilus-container \
  --account-name nautilusstor6471 \
  --account-key $STORAGE_KEY
```

*Expected output: `{ "created": true }`*

**3.5 Verify Container Exists**

```bash
az storage container show \
  --name nautilus-container \
  --account-name nautilusstor6471 \
  --account-key $STORAGE_KEY \
  --query "name" \
  -o tsv
```

**3.6 Upload the Static Asset to Blob Storage**

```bash
az storage blob upload \
  --account-name nautilusstor6471 \
  --account-key $STORAGE_KEY \
  --container-name nautilus-container \
  --file /root/index.html \
  --name index.html
```

**3.7 Verify Blob Upload**

```bash
az storage blob show \
  --account-name nautilusstor6471 \
  --account-key $STORAGE_KEY \
  --container-name nautilus-container \
  --name index.html \
  --query "{name:name, size:properties.contentLength}" \
  -o table
```

*Expected output confirms blob name `index.html` and size `233` bytes.*

Screenshot Placeholder: `06-blob-upload-verification.png`

---

### Phase 4: Virtual Machine Provisioning

**Objective:** Deploy an Ubuntu 22.04 LTS VM inside the VNet, using SSH public key authentication, and open the required network ports.

**4.1 Create the Virtual Machine**

```bash
az vm create \
  --name nautilus-vm \
  --resource-group $RG \
  --location eastus \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name nautilus-vnet \
  --subnet nautilus-subnet \
  --authentication-type ssh \
  --ssh-key-values "$(cat ~/.ssh/id_rsa.pub)" \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS \
  --public-ip-sku Standard
```

* `Standard_B1s`: A burstable compute tier appropriate for lightweight web serving workloads.
* `--authentication-type ssh`: Disables password-based login entirely, enforcing key-based authentication.
* `--ssh-key-values "$(cat ~/.ssh/id_rsa.pub)"`: Injects the previously generated public key at VM creation.
* `--public-ip-sku Standard`: Standard SKU public IPs support availability zones and are required for production-aligned deployments.
* `--os-disk-size-gb 30`: Provides adequate root disk capacity for the OS, Nginx, and Azure CLI.

> **Note:** Azure rejected `root` as the VM username as it is a reserved system account. The VM was created with the default `azureuser` account instead. All subsequent SSH commands target `azureuser@$PUBLIC_IP`.

*VM Public IP assigned:* `40.87.17.28`

Screenshot Placeholder: `07-vm-create-output.png`

**4.2 Retrieve and Confirm the VM Public IP**

```bash
PUBLIC_IP=$(az vm show \
  --name nautilus-vm \
  --resource-group $RG \
  --show-details \
  --query "publicIps" \
  -o tsv)
echo "VM Public IP: $PUBLIC_IP"

az vm show \
  --name nautilus-vm \
  --resource-group $RG \
  --show-details \
  --query "{name:name, powerState:powerState, publicIp:publicIps}" \
  -o table
```

Screenshot Placeholder: `08-vm-public-ip-power-state.png`

**4.3 Open Port 80 for HTTP Traffic**

```bash
az vm open-port \
  --name nautilus-vm \
  --resource-group $RG \
  --port 80
```

This creates an NSG rule named `open-port-80` with priority `900` (lower than the SSH rule at `1000`, giving it higher evaluation precedence) allowing all inbound TCP traffic on port 80.

**4.4 Verify NSG Rule for Port 80**

```bash
az network nsg rule show \
  --nsg-name nautilus-vmNSG \
  --resource-group $RG \
  --name open-port-80 \
  --query "{name:name, port:destinationPortRange, access:access}" \
  -o table
```

*Expected output: `open-port-80 | 80 | Allow`*

Screenshot Placeholder: `09-nsg-rule-port-80-verification.png`

---

### Phase 5: Web Server Configuration and Content Deployment

**Objective:** Install Nginx and Azure CLI on the VM via SSH, download the `index.html` blob directly to the Nginx web root, set correct file permissions, and restart the web server.

**5.1 Install Nginx and Azure CLI on the VM**

```bash
ssh -i ~/.ssh/id_rsa azureuser@$PUBLIC_IP \
  "sudo apt-get update -y && \
   sudo apt-get install -y nginx && \
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash && \
   az --version"
```

This single SSH session performs the following sequence:
* Updates the APT package index.
* Installs Nginx (version `1.18.0`), which is automatically enabled and started via systemd.
* Installs Azure CLI (version `2.84.0`) using the official Microsoft installation script.
* Verifies the Azure CLI installation by printing the version string.

Screenshot Placeholder: `10-nginx-azcli-install-verification.png`

**5.2 Download Blob, Set Permissions, Enable and Restart Nginx, Verify Locally**

```bash
ssh -i ~/.ssh/id_rsa azureuser@$PUBLIC_IP \
  "sudo az storage blob download \
    --account-name nautilusstor6471 \
    --account-key '$STORAGE_KEY' \
    --container-name nautilus-container \
    --name index.html \
    --file /var/www/html/index.html && \
  sudo chown www-data:www-data /var/www/html/index.html && \
  sudo chmod 644 /var/www/html/index.html && \
  sudo systemctl enable nginx && \
  sudo systemctl restart nginx && \
  curl http://localhost"
```

This single SSH session performs the following sequence:

* **`az storage blob download`**: Authenticates to Blob Storage using the account key and downloads `index.html` directly to `/var/www/html/index.html`, Nginx's default document root.
* **`chown www-data:www-data`**: Transfers file ownership to the `www-data` user and group, which is the process identity under which Nginx runs on Debian-based systems.
* **`chmod 644`**: Sets read permissions for owner (write + read) and group/other (read only), following the principle of least privilege for static web content.
* **`systemctl enable nginx`**: Registers Nginx as a systemd service to start automatically on VM reboot.
* **`systemctl restart nginx`**: Applies any configuration changes and starts serving content.
* **`curl http://localhost`**: Performs an in-VM HTTP loopback test to confirm Nginx is serving the correct content before any external verification.

*Expected local curl output:*

```html
<!DOCTYPE html>
<html lang=en>
<head>
    <meta charset=UTF-8>
    <meta name=viewport content=width=device-width, initial-scale=1.0>
    <title>Welcome</title>
</head>
<body>
    <h1>Welcome to KKE Azure Labs!</h1>
</body>
</html>
```

Screenshot Placeholder: `11-blob-download-nginx-restart-local-curl.png`

---

### Phase 6: End-to-End Verification

**Objective:** Confirm the static web application is publicly accessible from the client host via the VM's public IP address.

**6.1 HTTP Request from the Client Host**

```bash
curl http://$PUBLIC_IP
```

*Expected output:*

```html
<!DOCTYPE html>
<html lang=en>
<head>
    <meta charset=UTF-8>
    <meta name=viewport content=width=device-width, initial-scale=1.0>
    <title>Welcome</title>
</head>
<body>
    <h1>Welcome to KKE Azure Labs!</h1>
</body>
</html>
```

A successful HTTP 200 response returning the exact content of the uploaded blob confirms the end-to-end pipeline is functioning correctly: Blob Storage uploaded and preserved the asset, the VM fetched it securely, Nginx is serving it from the correct root, and the NSG is permitting inbound port 80 traffic.

Screenshot Placeholder: `12-external-curl-public-ip-success.png`

---

## Best Practices Applied

| Domain | Practice | Implementation Detail |
|---|---|---|
| **Security** | Disabled public blob access | `--allow-blob-public-access false` at storage account creation |
| **Security** | SSH-only VM authentication | `--authentication-type ssh`; password login disabled |
| **Security** | RSA 4096-bit key pair | `ssh-keygen -t rsa -b 4096` |
| **Security** | Least-privilege file permissions | `chmod 644` and `chown www-data:www-data` for served file |
| **Reliability** | Nginx enabled at boot | `systemctl enable nginx` ensures service survives VM restarts |
| **Reliability** | Loopback test before external verification | `curl http://localhost` validates serving before public exposure |
| **Networking** | Standard SKU public IP | Supports zone redundancy and production-grade SLA |
| **Networking** | Explicit NSG rules | Only ports 22 and 80 opened; all other inbound denied by default |
| **Portability** | Dynamic resource group resolution | `az group list --query "[0].name"` removes hardcoded dependencies |
| **Auditability** | Partial key echo | `${STORAGE_KEY:0:8}...` confirms capture without exposing the full key |
| **Separation of Concerns** | Asset decoupled from codebase | `index.html` managed independently in Blob Storage |

---

## Errors Encountered and Resolutions

### Error 1: Reserved Username Rejection During VM Creation

**Symptom:**

```
Default username root is a reserved username. Use azureuser instead.
```

**Root Cause:** Azure explicitly prohibits the use of `root` as a VM OS username because it is a reserved system account on Linux. This is a platform-level restriction enforced during VM provisioning.

**Resolution:** The VM was provisioned without specifying `--admin-username`, defaulting to `azureuser`. All subsequent SSH commands used `azureuser@$PUBLIC_IP` as the connection target. The `root` user on the VM is still accessible via `sudo` from the `azureuser` session.

**Preventive Measure:** Always specify a non-reserved admin username explicitly using `--admin-username <name>` to avoid implicit defaults and improve clarity in team environments.

---

### Error 2: Non-Interactive TTY Warning During APT Package Installation

**Symptom:**

```
debconf: unable to initialize frontend: Dialog
debconf: falling back to frontend: Teletype
dpkg-preconfigure: unable to re-open stdin:
```

**Root Cause:** The APT `debconf` system attempted to render an interactive dialog UI over the SSH session, which does not have a controlling TTY in non-interactive mode. This is expected behavior when running `apt-get install` over a non-interactive SSH command.

**Resolution:** The warnings were non-fatal. Package installation completed successfully. The `-y` flag on `apt-get install` pre-answered all prompts, and `debconf` fell back to the Teletype frontend automatically.

**Preventive Measure:** For fully suppressed output, prefix interactive SSH commands with `DEBIAN_FRONTEND=noninteractive` to prevent `debconf` from attempting dialog rendering:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y nginx
```

---

## Lessons Learned

**1. Key-Based VM Authentication Eliminates a Major Attack Surface**
Disabling password authentication at provisioning time (`--authentication-type ssh`) is significantly more effective than relying on post-deployment hardening. Brute-force attacks against SSH are one of the most common cloud VM attack vectors, and removing the password vector entirely at the network identity layer is the correct first control.

**2. Blob Storage as a Secure Content Staging Layer is a Production Pattern**
Keeping static assets in a private Blob container decoupled from the VM filesystem or application repository provides a clean operational boundary. Assets can be updated in Blob Storage independently of the VM or the codebase, and the VM can re-fetch updated content without redeployment.

**3. Dynamic Resource Group Resolution Improves Lab and Pipeline Portability**
Using `az group list --query "[0].name" -o tsv` instead of hardcoding the resource group name allows the same script to function across different Azure lab environments where group names are generated dynamically. This pattern is applicable to any multi-tenant or ephemeral environment toolchain.

**4. Loopback Verification Before External Testing Isolates Failure Domains**
Running `curl http://localhost` inside the VM before testing `curl http://$PUBLIC_IP` from the client host creates a clear debugging boundary. If the loopback succeeds but the external request fails, the issue is definitively in the NSG, routing, or public IP assignment rather than the web server configuration.

**5. File Ownership and Permissions Must Match the Web Server Process Identity**
Setting `chown www-data:www-data` and `chmod 644` is not optional on production systems. Nginx's worker processes run as `www-data`, and if the served file is owned by `root` with restrictive permissions, Nginx will silently return a 403 Forbidden response. Establishing the correct ownership at deployment time prevents this class of silent failure.

**6. Storage Account Key Exposure in Shell History is a Security Risk**
Passing `$STORAGE_KEY` directly as a CLI argument results in the key appearing in the shell history of both the client host and the remote VM. In production, this should be replaced with Azure Managed Identity assigned to the VM, eliminating the need to handle storage keys entirely.

---

## Resource Inventory

| Resource | Name | Type | Location |
|---|---|---|---|
| Resource Group | `kml_rg_main-534f47daa771482e` | Resource Group | East US |
| Virtual Network | `nautilus-vnet` | VNet (`10.0.0.0/16`) | East US |
| Subnet | `nautilus-subnet` | Subnet (`10.0.0.0/24`) | East US |
| Storage Account | `nautilusstor6471` | Standard LRS, Private | East US |
| Blob Container | `nautilus-container` | Private | N/A |
| Blob | `index.html` | Block Blob (233 bytes) | N/A |
| Virtual Machine | `nautilus-vm` | Standard_B1s, Ubuntu 22.04 | East US |
| Public IP | `40.87.17.28` | Standard SKU | East US |
| NSG | `nautilus-vmNSG` | Port 22 + Port 80 inbound | East US |

---

*Produced by the Nautilus DevOps Team. Follow Azure best practices for security, accessibility, and performance when adapting this guide to production environments.*





<img width="1033" height="864" alt="image" src="https://github.com/user-attachments/assets/595adc1f-618a-49cd-a17e-31d05da4d8b8" />
<img width="1036" height="862" alt="image" src="https://github.com/user-attachments/assets/5815c3f7-6934-432e-be7d-13edc084c48f" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/ee19999f-67d2-4f02-aa5a-ded42a559d14" />
<img width="1028" height="863" alt="image" src="https://github.com/user-attachments/assets/de066204-0fe6-4606-a396-fccdb215a1ae" />
<img width="1035" height="367" alt="image" src="https://github.com/user-attachments/assets/bb4bfc32-f121-4f90-a758-b956e4d33b1f" />
<img width="1030" height="510" alt="image" src="https://github.com/user-attachments/assets/d0cb6dcc-c785-400e-8b0d-b20732b3c6b5" />
<img width="1033" height="512" alt="image" src="https://github.com/user-attachments/assets/aae3bd4b-11f4-43d1-9f0a-757e0184a5f1" />
<img width="1028" height="801" alt="image" src="https://github.com/user-attachments/assets/a0f885cb-6257-4426-8996-15b3fb3b036f" />
<img width="1028" height="863" alt="image" src="https://github.com/user-attachments/assets/62c607ef-2063-40f6-82c9-ed815a939451" />
<img width="1031" height="627" alt="image" src="https://github.com/user-attachments/assets/4963abd5-bca2-4810-8f0e-573b8cb56da3" />
<img width="1033" height="853" alt="image" src="https://github.com/user-attachments/assets/8f5cf328-0fbc-4933-b781-f31ff75c0856" />
<img width="1034" height="866" alt="image" src="https://github.com/user-attachments/assets/c19d4516-03ef-4a2e-a005-e2e953bf7341" />
<img width="1027" height="857" alt="image" src="https://github.com/user-attachments/assets/a68e732a-027e-4d2c-8768-f79f7639422f" />
<img width="1029" height="858" alt="image" src="https://github.com/user-attachments/assets/2c14e93d-c792-4676-aa17-fad89b4f58b2" />
<img width="1035" height="865" alt="image" src="https://github.com/user-attachments/assets/b41bf0b4-3999-4d85-8b04-501db291d5bb" />
<img width="1033" height="864" alt="image" src="https://github.com/user-attachments/assets/f92d99e7-e5d2-443a-bda9-1096351e076a" />
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/4c64c2a0-67a0-48f4-a06c-1ac5b8e1cb2b" />
<img width="1031" height="863" alt="image" src="https://github.com/user-attachments/assets/6046306e-4f41-4eb3-b6a9-7da651125323" />
<img width="1025" height="860" alt="image" src="https://github.com/user-attachments/assets/1bc1ee75-c6f1-4b34-a639-64b66fa0f4f1" />
<img width="1034" height="851" alt="image" src="https://github.com/user-attachments/assets/5a3fc563-37b0-4715-962a-16aee554b7f5" />
<img width="1031" height="861" alt="image" src="https://github.com/user-attachments/assets/4646d053-e7d1-473c-a50e-61a45a8d284b" />
<img width="1033" height="836" alt="image" src="https://github.com/user-attachments/assets/349b99ed-6727-4ed7-8be6-bff7964e29af" />
<img width="1037" height="865" alt="image" src="https://github.com/user-attachments/assets/01fc67d0-595e-4393-b723-3d847697656f" />
<img width="1021" height="867" alt="image" src="https://github.com/user-attachments/assets/a7802422-146d-4690-8d19-f1d0077fe801" />
<img width="1033" height="647" alt="image" src="https://github.com/user-attachments/assets/a213aeb6-7ba4-4944-b9ec-3acd2f603489" />
