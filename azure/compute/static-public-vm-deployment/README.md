# Azure VM Deployment with Static Public IP (DevOps Lab)

## Project Overview

This project documents the end-to-end provisioning of an **Azure Virtual Machine** with a **Standard Static Public IP address** to guarantee stable and consistent external access for application workloads. The deployment was executed entirely via the **Azure CLI**, following infrastructure-as-code principles commonly adopted by FAANG-scale DevOps teams.

The final setup delivers:
- A Ubuntu-based VM (`devops-vm`)
- A Standard SKU Static Public IP (`devops-pip`)
- Secure SSH access using public key authentication
- Deployment scoped to the **Central US** Azure region

---

## Architecture Summary

- **Cloud Provider:** Microsoft Azure  
- **Region:** Central US  
- **Compute:** Azure Virtual Machine (Standard_B1s)  
- **OS Image:** Ubuntu 22.04 LTS  
- **Networking:** Standard Static Public IP  
- **Access Method:** SSH (Key-based authentication)

---

## Prerequisites

- Azure CLI installed and authenticated
- Valid Azure subscription credentials
- Access to `azure-client` host
- SSH client available

---

## Step 1: Verify Existing Resource Group

Validate the target resource group to ensure it is available and healthy.

```
az group list --output table
```
### Expected Outcome

- Resource group exists

- Status shows Succeeded

Screenshot: `Resource Group Validation Output`
<img width="1030" height="622" alt="image" src="https://github.com/user-attachments/assets/fec68cb4-d0d9-4da0-bd24-0f6f4548f329" />

## Step 2: Generate SSH Key Pair for VM Access

- Create a dedicated SSH key pair to securely access the VM.
```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/devops-key -N ""
```

Artifacts Created
```
Private Key: ~/.ssh/devops-key

Public Key: ~/.ssh/devops-key.pub
```
Screenshot: `SSH Key Generation Output`
<img width="1031" height="634" alt="image" src="https://github.com/user-attachments/assets/65a92e57-f870-4de3-8159-8a869ccb5903" />

## Step 3: Retrieve Azure Environment Credentials

- Confirm Azure authentication context using provided lab credentials.
```
showcreds
```
- Validated Information

- Azure Portal URL

- Username

- Active session window

Screenshot: `Azure Credentials Output`


<img width="968" height="278" alt="image" src="https://github.com/user-attachments/assets/de7cfe57-b167-403f-8047-dd1bdced1ccc" />

## Step 4: Create a Standard Static Public IP Address

- Provision a Standard SKU Static Public IP to ensure IP persistence across VM restarts.
```
az network public-ip create \
  --resource-group <RESOURCE_GROUP> \
  --name devops-pip \
  --location centralus \
  --allocation-method Static \
  --sku Standard
```

### Key Properties

- Allocation Method: `Static`

- IP Version: `IPv4`

- SKU: `Standard`

Screenshot: `Public IP Creation Output`

## Step 5: Create the Azure Virtual Machine

- Deploy the VM and explicitly associate it with the previously created static public IP.
```
az vm create \
  --resource-group <RESOURCE_GROUP> \
  --name devops-vm \
  --location centralus \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/devops-key.pub \
  --public-ip-address devops-pip \
  --public-ip-sku Standard \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS \
  --nsg-rule SSH
```
- Provisioning Results

- VM successfully created

- Public IP attached at creation time

- SSH allowed via Network Security Group

Screenshot: `VM Creation Output`


## Step 6: Validate SSH Access to the VM

- Test secure connectivity using the generated SSH private key.
```
ssh -i ~/.ssh/devops-key azureuser@<STATIC_PUBLIC_IP>
```
- Verification Checks

- Successful login

- Ubuntu 22.04 LTS banner visible

- VM hostname resolves correctly

Screenshot: `Initial SSH Login to VM`


## Step 7: Add an Additional SSH Key (User Access Hardening)

- Generate a secondary SSH key and update the VM user profile.
```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
az vm user update \
  --resource-group <RESOURCE_GROUP> \
  --name devops-vm \
  --username azureuser \
  --ssh-key-value ~/.ssh/id_rsa.pub
```

Purpose

- Enables key rotation

- Supports multiple secure access paths

Screenshot: `VM User SSH Key Update`
<img width="1037" height="576" alt="image" src="https://github.com/user-attachments/assets/1ef78f27-88ad-417c-96c3-6fb328fca3bb" />

## Step 8: Revalidate SSH Access Using Updated Configuration

- Confirm uninterrupted access after user key update.

- ssh azureuser@<STATIC_PUBLIC_IP>

Expected Outcome

- SSH access remains functional

- No password authentication required

Screenshot: `Post-Update SSH Login`

## Step 9: Confirm Static Public IP Persistence

- Verify that the public IP remains static and unchanged.
```
az network public-ip show \
  --resource-group <RESOURCE_GROUP> \
  --name devops-pip \
  --query "{Name:name, IPAddress:ipAddress, Allocation:publicIPAllocationMethod}" \
  --output table
```

Validation Result

- Allocation Method: `Static`

- IP Address unchanged

Screenshot: `Static Public IP Verification`
<img width="1036" height="859" alt="image" src="https://github.com/user-attachments/assets/df63d106-0926-4e45-afe8-c217d784c2f5" />


## Final State Summary

| Component       | Status                |
| --------------- | --------------------- |
| Virtual Machine | Running               |
| OS              | Ubuntu 22.04 LTS      |
| VM Size         | Standard_B1s          |
| Public IP       | Static (Standard SKU) |
| SSH Access      | Key-Based             |
| Region          | Central US            |


## Key Takeaways

- Static Public IPs ensure consistent external access for applications

- Standard SKU is required for production-grade workloads

- SSH key-based authentication aligns with DevOps security best practices

- CLI-driven deployments promote repeatability and auditability



<img width="1032" height="709" alt="image" src="https://github.com/user-attachments/assets/f78d97be-84a3-41c0-b9dc-81e795ecf825" />
<img width="1037" height="862" alt="image" src="https://github.com/user-attachments/assets/056ba576-455c-466f-96b9-80bbbcef4e61" />
<img width="1028" height="845" alt="image" src="https://github.com/user-attachments/assets/770ff500-1fe4-4d55-92ca-ec13d1e5a1dc" />
<img width="1030" height="409" alt="image" src="https://github.com/user-attachments/assets/f39f547d-2fee-43cf-aba1-ac8d80e74a9f" />





