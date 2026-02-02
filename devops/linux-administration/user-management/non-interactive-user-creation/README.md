# Linux User Management – Non-Interactive User Creation

## Overview
This lab demonstrates how to create a Linux user with a non-interactive shell.
Non-interactive users are commonly used for system services, backup agents,
and automation tools where shell access is not required or allowed.

---

## Lab Context
Project Nautilus – xFusionCorp Industries  
Requirement: Create a service user for a backup agent with no login capability.

Target Server:
- App Server 1 (stapp01)

---

## Objective
- Create a user named `kareem`
- Assign a non-interactive shell
- Prevent SSH and terminal access
- Verify correct configuration

---

## Environment
- OS: Linux
- Server Role: Application Server
- Privileges: sudo access

---

## Step 1: Connect to Target Server

CONNECT to jump host
SSH into App Server 1 as authorized user

📸 Screenshot:

<img width="1039" height="846" alt="image" src="https://github.com/user-attachments/assets/962db087-ee56-4ffe-a993-148b17d66c36" />

SSH login to stapp01

## Step 2: Create User with Non-Interactive Shell
- EXECUTE useradd command
- SET login shell to /sbin/nologin
- USERNAME = kareem
- sudo useradd -s /sbin/nologin kareem

📸 Screenshot:

<img width="1029" height="847" alt="image" src="https://github.com/user-attachments/assets/673bea01-484d-43c7-887b-37f1558f8190" />

Command execution output

## Step 3: Verify User Configuration
- QUERY system user database
- CONFIRM assigned shell is non-interactive
- getent passwd kareem
- Expected Output (example):

kareem:x:1005:1005::/home/kareem:/sbin/nologin

📸 Screenshot:

<img width="1033" height="815" alt="image" src="https://github.com/user-attachments/assets/a1bf8a6e-5e55-434f-affc-d67ca1bd42bf" />

passwd entry showing /sbin/nologin

## Step 4: Confirm Login Is Disabled
- ATTEMPT user login
- EXPECT access denial
- su - kareem

Expected Result:

This account is currently not available.

📸 Screenshot:

<img width="1043" height="429" alt="image" src="https://github.com/user-attachments/assets/fd9ee0da-2144-4556-98f2-c6b64760b416" />

Login attempt blocked

## Result
- Non-interactive user created successfully

- Shell access restricted

- System security requirement satisfied

## Security Best Practices
- Use non-interactive shells for service accounts
  
- Prevent unnecessary SSH access

- Follow least-privilege principles

- Avoid assigning passwords to service users

## Real-World Relevance

- This mirrors real production environments where:

- Backup agents run under restricted users

- Automation tools require system access without login

- Security teams enforce strict account controls

## Skills Demonstrated
- Linux user management

- System security hardening

- Service account configuration

-- Enterprise Linux administration
