# Azure SSH Key Pair Creation (Azure Portal)

## Overview
This lab demonstrates the creation of an **Azure-managed SSH key pair** using the **Azure Portal UI**.  
The objective is to ensure the SSH key exists as a **native Azure resource**, aligning with Azure’s managed infrastructure standards.

---

## 🎯 Objective
- Create an SSH key pair in Azure
- SSH key name: **`devops-kp`**
- Key type: **RSA**
- Method: **Azure Portal**

---

## ☁️ Cloud Platform
- **Microsoft Azure**
- Azure Portal (Web UI)

---

## High-Level Workflow

- Login to Azure Portal
  -  → Navigate to SSH Keys service
  -  → Create new SSH key
  -  → Configure key properties
  -  → Generate and store key in Azure
  -  → Verify key existence

## Step-by-Step Implementation

- Step 1: Log in to Azure Portal
  -  Open https://portal.azure.com
  -  Authenticate using provided Azure credentials

📸 Screenshot:
<img width="1736" height="955" alt="image" src="https://github.com/user-attachments/assets/e420a513-502a-42a0-b3bb-f241060b19bf" />

- Step 2: Navigate to SSH Keys Service
  -  Azure Portal Home
  -  → Search bar
  -  → Type "SSH keys"
  -  → Select SSH keys service

📸 Screenshot:
<img width="1750" height="946" alt="image" src="https://github.com/user-attachments/assets/91d6c39d-068e-4268-951e-0c5133f7e41e" />

- Step 3: Create a New SSH Key
- SSH Keys page
  -  → Click "+ Create"

- Step 4: Configure SSH Key Properties
  -  Name        = devops-kp
  -  Key type    = RSA
  -  Source      = Generate key pair

📸 Screenshot:
<img width="1749" height="950" alt="image" src="https://github.com/user-attachments/assets/969681b1-babd-42dc-a5fc-ee842c6ee10c" />

## Step 5: Create and Download Key
- Click "Create"
- Download private key when prompted
- Azure stores public key as a managed resource

📸 Screenshot:
<img width="1742" height="945" alt="image" src="https://github.com/user-attachments/assets/a70469d7-bcbb-40ec-9830-fd25af60ab70" />

Step 6: Verify SSH Key Creation
- Return to SSH Keys service
- Confirm "devops-kp" appears in the list

## Validation Outcome
- SSH key "devops-kp" exists in Azure
- Key type is RSA
- Key is managed by Azure
- Task successfully completed

## Key Learnings

- Azure SSH keys must exist as Azure-managed resources

- Local SSH key generation does not satisfy cloud validation

- Azure Portal is the authoritative control plane for managed services

- Correct naming is critical for automated task validation

## Conclusion
- This lab reinforces the importance of using the Azure Portal for infrastructure components that require control-plane validation.
Creating SSH keys directly within Azure ensures compatibility, security, and compliance with Azure-managed workflows.
