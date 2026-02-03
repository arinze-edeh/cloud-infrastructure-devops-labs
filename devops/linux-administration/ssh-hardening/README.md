# Disable Root SSH Login (Linux Hardening)

## Overview
This lab demonstrates how to harden Linux servers by disabling direct SSH access for the root user.  
This is a standard security control to enforce least privilege and reduce brute-force attack risks.

---

## Lab Objective
- Prevent direct SSH login as root
- Enforce administrative access via sudo
- Apply changes across all Nautilus app servers

---

## Environment
- Project: Nautilus (Stratos Datacenter)
- Servers:
  - stapp01
  - stapp02
  - stapp03
- OS: Linux
- Service: OpenSSH

---

## Step 1: Access Jump Host

- CONNECT to jump host using SSH
- AUTHENTICATE with provided credentials

📸 Screenshot: Jump host login
<img width="1030" height="788" alt="image" src="https://github.com/user-attachments/assets/2762f39f-7c18-4d8c-aa31-6240f988eaed" />

## Step 2: Connect to App Server
- SSH into target app server (non-root user)

📸 Screenshot: SSH session to app server
<img width="1027" height="825" alt="image" src="https://github.com/user-attachments/assets/c81c1b0e-aa68-43b2-972e-1cfe9ae6bb5c" />

## Step 3: Elevate Privileges
- SWITCH to root using sudo

📸 Screenshot: Root shell confirmation
<img width="1032" height="764" alt="image" src="https://github.com/user-attachments/assets/7303bde5-2fe5-4d20-98f1-f669aeb14da2" />
<img width="1037" height="866" alt="image" src="https://github.com/user-attachments/assets/706dd8cd-f0cc-4743-9c5b-9e83153a7439" />

## Step 4: Edit SSH Configuration
- OPEN /etc/ssh/sshd_config
- LOCATE PermitRootLogin directive
- CHANGE value from "yes" to "no"
- SAVE configuration

📸 Screenshot: sshd_config edited
<img width="1038" height="863" alt="image" src="https://github.com/user-attachments/assets/5f94446f-2de4-4bc0-84d1-45773eaced14" />
<img width="1030" height="842" alt="image" src="https://github.com/user-attachments/assets/986f2b6c-4efd-4166-a38c-35f09d0fc897" />
<img width="1037" height="866" alt="image" src="https://github.com/user-attachments/assets/1746211a-dc77-4765-bed9-6a023843431f" />

## Step 5: Restart SSH Service
- RESTART sshd service to apply changes

📸 Screenshot: SSH service restart
<img width="1040" height="862" alt="image" src="https://github.com/user-attachments/assets/81fab739-cbc9-4b76-b70b-ec7bb10acd72" />
<img width="1037" height="864" alt="image" src="https://github.com/user-attachments/assets/28adb7cc-f0d6-4c62-9d85-38082c86fa7a" />

## Step 6: Verify Configuration
- QUERY SSH daemon runtime config
- CONFIRM PermitRootLogin is set to "no"

📸 Screenshot: Verification output
<img width="1032" height="866" alt="image" src="https://github.com/user-attachments/assets/b9d12071-30ba-4288-acba-31ff55f8b844" />
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/5a1f872b-7675-4043-b59a-420ceed13f12" />

## Result
- Root SSH login successfully disabled

- Servers hardened against direct root access

- SSH access preserved for non-root users

## Security Best Practices
- Disable direct root SSH access

- Use sudo for administrative actions

- Enforce least-privilege access

- Regularly audit SSH configurations

## Real-World Relevance
- This configuration mirrors real production security baselines used in enterprise Linux and cloud environments to prevent unauthorized access.

## Skills Demonstrated
- Linux server hardening

- SSH security configuration

- Privilege management

- Operational security best practices
