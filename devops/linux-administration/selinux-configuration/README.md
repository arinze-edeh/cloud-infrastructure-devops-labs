# Disable SELinux on App Server 1

## 📌 Lab Overview
- Following a security audit, xFusionCorp Industries initiated SELinux
testing on their application servers. For initial testing, SELinux
must be permanently disabled on App Server 1.

- This lab demonstrates how to safely disable SELinux without rebooting
the server immediately.

---

## 🎯 Objectives
- Install required SELinux packages
- Permanently disable SELinux via configuration
- Avoid immediate reboot
- Ensure SELinux is disabled after scheduled reboot

---

## 🧠 High-Level Logic

- CONNECT to App Server 1
- INSTALL required SELinux packages

- OPEN SELinux configuration file
- SET SELINUX=disabled
- SAVE configuration

- DO NOT reboot system
- CONFIRM configuration is applied

## 🛠️ Implementation Steps

## Step 1: Connect to App Server
- `ssh tony@stapp01`

📸 screenshot:
<img width="1031" height="556" alt="image" src="https://github.com/user-attachments/assets/d4340256-8c8b-426d-b807-6570015e8a36" />

## Step 2: Install SELinux Packages
- `sudo yum install -y selinux-policy selinux-policy-targeted policycoreutils`

📸 screenshot:
<img width="1033" height="856" alt="image" src="https://github.com/user-attachments/assets/0d5526fa-ae42-423b-9b77-8af470abf278" />
<img width="1030" height="864" alt="image" src="https://github.com/user-attachments/assets/b1072b6b-359f-4fb4-b969-ab8d223a0f29" />

## Step 3: Modify SELinux Configuration
- `sudo vi /etc/selinux/config`
- Set:

  -  SELINUX=disabled

📸 screenshot:
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/26cb4d73-0391-4c5f-9390-b418e9caee07" />

## Step 4: No Reboot Required
- Server reboot is not performed

📸 screenshot:
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/26cb4d73-0391-4c5f-9390-b418e9caee07" />

## Step 5: Status Check (Informational)
- `sestatus`

📸 screenshot:
<img width="1031" height="859" alt="image" src="https://github.com/user-attachments/assets/6d417c93-378a-4dc2-9d1d-237124c85035" />

## ✅ Final Outcome
- SELinux packages installed

- SELinux permanently disabled

- No immediate reboot performed

- System compliant with security audit requirements

## 🏷️ Tags
`linux` `selinux` `security` `system-administration` `devops`





