# Azure Virtual Machine Creation – Ubuntu 22.04

## 📌 Overview
This lab documents the creation of an **Azure Virtual Machine** using the **Azure Portal UI** as part of the Nautilus DevOps team’s incremental cloud migration strategy.

---

## 🎯 Objective
Provision an Azure VM with the following specifications:

- VM Name: `xfusion-vm`
- Region: `East US (eastus)`
- OS Image: Ubuntu Server 22.04 LTS
- VM Size: Standard_B1s
- Disk: 30 GB Standard HDD
- Network: Default NSG allowing SSH (port 22)
- Access: SSH connectivity verified

---

## ☁️ Cloud Platform
- Microsoft Azure
- Azure Portal (UI)

---

## 🧭 High-Level Workflow (Pseudo-Code)

Login to Azure Portal
→ Create Virtual Machine
→ Configure basics (name, region, image, size)
→ Configure networking (allow SSH)
→ Configure disk (30 GB Standard HDD)
→ Review and deploy VM
→ SSH into VM to validate access
🛠️ Step-by-Step Implementation
Step 1: Login
Access https://portal.azure.com
Authenticate with provided credentials
📸 Screenshot: Azure Portal login page

Step 2: Create Virtual Machine
Navigate to Virtual Machines
Click "+ Create" → Azure Virtual Machine
📸 Screenshot: Virtual Machines dashboard

Step 3: Configure Basics
Resource Group: Existing RG
VM Name: xfusion-vm
Region: East US
Image: Ubuntu Server 22.04 LTS
Size: Standard_B1s
📸 Screenshot: Basics tab configuration

Step 4: Networking
Use default VNet and subnet
Allow inbound SSH (port 22)
Attach default NSG
📸 Screenshot: Networking tab with SSH allowed

Step 5: Disk Configuration
OS Disk Type: Standard HDD
Disk Size: 30 GB
📸 Screenshot: Disk configuration tab

Step 6: Review and Create
Leave remaining settings as default
Validate configuration
Deploy VM
📸 Screenshot: Validation passed screen

Step 7: SSH Verification
Download SSH private key
Connect to VM via SSH
Confirm successful login
📸 Screenshot: SSH connection or terminal access

✅ Validation Checklist
✔ VM exists with correct name
✔ Region is eastus
✔ Ubuntu 22.04 installed
✔ VM size is Standard_B1s
✔ SSH access confirmed
✔ Disk size is 30 GB Standard HDD

🧠 Key Learnings
Azure VM provisioning via Portal

Importance of correct region and VM sizing

NSG rules control access, not the VM itself

SSH verification is mandatory for validation

🏁 Conclusion
This lab validates the ability to provision and access an Azure Virtual Machine using best practices, forming a core building block for cloud infrastructure migration.




<img width="1740" height="949" alt="image" src="https://github.com/user-attachments/assets/4abac257-12d7-4c63-9e74-abfed162ac04" />
<img width="1781" height="950" alt="image" src="https://github.com/user-attachments/assets/38a332d6-c73e-4323-9a68-6753e13a661b" />
<img width="1761" height="951" alt="image" src="https://github.com/user-attachments/assets/8bc3d40a-0529-4f80-9f17-1e6fbf4b368a" />
<img width="1749" height="945" alt="image" src="https://github.com/user-attachments/assets/d0b77cc1-9876-4330-a833-ad82eb1ab5a5" />
<img width="1747" height="943" alt="image" src="https://github.com/user-attachments/assets/69e781e6-d20c-42cb-9924-7e36faa93c59" />
<img width="1760" height="951" alt="image" src="https://github.com/user-attachments/assets/f5be68e5-085e-4ee5-99c4-b913e6e86175" />
<img width="1783" height="943" alt="image" src="https://github.com/user-attachments/assets/77949321-80e8-4872-88b0-ba7574d026ea" />
<img width="1760" height="947" alt="image" src="https://github.com/user-attachments/assets/29d18b57-43f2-46d6-9c7a-8a99fd67a63d" />
<img width="1758" height="948" alt="image" src="https://github.com/user-attachments/assets/748f9958-3c5a-46a9-9ee7-45ef484f5563" />
<img width="1734" height="951" alt="image" src="https://github.com/user-attachments/assets/92942e70-9fd8-4a72-95a4-7df876050060" />
<img width="1792" height="947" alt="image" src="https://github.com/user-attachments/assets/808b1bd5-23ff-45d2-bad4-0babc2e0dda0" />
<img width="1754" height="954" alt="image" src="https://github.com/user-attachments/assets/0d03d98a-ba47-4622-bfc8-ed42c05d8a08" />
<img width="1762" height="951" alt="image" src="https://github.com/user-attachments/assets/87f58cf9-271b-486c-8bfb-58b94e25a516" />
<img width="1756" height="945" alt="image" src="https://github.com/user-attachments/assets/c64c2007-d8a1-44dd-a519-0a2e26b3f1e1" />
<img width="1789" height="948" alt="image" src="https://github.com/user-attachments/assets/eefcac26-9b57-4a9c-a512-4f4a7a62a3c8" />
<img width="1870" height="952" alt="image" src="https://github.com/user-attachments/assets/b93cbadc-5683-4966-ac97-87888c3ca912" />
<img width="1779" height="947" alt="image" src="https://github.com/user-attachments/assets/b761740b-30b5-4ab2-8f4e-3ff9d63ed9c7" />
<img width="1917" height="904" alt="image" src="https://github.com/user-attachments/assets/7ccf9e53-b4a8-44d6-9c8b-424b2666d618" />
<img width="1797" height="947" alt="image" src="https://github.com/user-attachments/assets/42c71886-3210-4224-9fd6-de1305170fb8" />
<img width="1829" height="949" alt="image" src="https://github.com/user-attachments/assets/fb929034-c46b-43da-9585-765f3d847868" />
<img width="1048" height="882" alt="image" src="https://github.com/user-attachments/assets/e0856254-42d0-4e40-8143-0986680e26b5" />
<img width="1031" height="875" alt="image" src="https://github.com/user-attachments/assets/e852c320-8da5-4c9c-9c6f-dbc18400b2e6" />


