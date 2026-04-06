# Provisioning Non-Interactive Service Accounts on Linux

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment and Prerequisites](#environment-and-prerequisites)
- [Architecture](#architecture)
- [Step 1: Connect to the Target Server](#step-1-connect-to-the-target-server)
- [Step 2: Create the User with a Non-Interactive Shell](#step-2-create-the-user-with-a-non-interactive-shell)
- [Step 3: Verify User Configuration](#step-3-verify-user-configuration)
- [Step 4: Confirm Login Access Is Disabled](#step-4-confirm-login-access-is-disabled)
- [Result](#result)
- [Security Best Practices](#security-best-practices)
- [Operational Considerations and Edge Cases](#operational-considerations-and-edge-cases)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This document describes the process of creating a **non-interactive Linux user account** for use as a service or automation identity. Non-interactive accounts are a foundational security control in enterprise Linux environments, ensuring that system-level processes can operate with scoped privileges while **preventing unauthorized shell access**.

This procedure follows the principle of least privilege and is directly applicable to production environments managing backup agents, CI/CD runners, monitoring daemons, and other automated workloads.

**Project Context:** Project Nautilus, xFusionCorp Industries
**Target Server:** App Server 1 (`stapp01.stratos.xfusioncorp.com`)
**Service Account Name:** `kareem`

---

## Problem Statement

Automated services and background agents running on Linux systems often require a dedicated system identity to:

- Isolate process ownership and file permissions
- Scope sudo or ACL grants to a single, auditable principal
- Prevent the service account from being used as an interactive login vector

Assigning a regular user account (with `/bin/bash` or similar) to a service carries significant risk: if the account is compromised or misconfigured, an attacker gains an interactive shell on the system. The solution is to create the account with `/sbin/nologin` as its shell, which **blocks all interactive login attempts** while still allowing the account to own processes and files.

---

## Environment and Prerequisites

| Parameter | Value |
|---|---|
| Jump Host | `jumphost` (accessed as user `thor`) |
| Target Server | `stapp01.stratos.xfusioncorp.com` |
| Target Server IP | `172.17.0.7` |
| Operating System | Linux (RHEL/CentOS-compatible) |
| Authorized User | `tony` (sudo-enabled) |
| Service Account | `kareem` |
| Shell Assignment | `/sbin/nologin` |

**Prerequisites:**
- SSH access from the jump host to the target server
- `sudo` privileges on the target server for the authenticating user (`tony`)
- Standard `useradd` and `getent` utilities available (default on all major Linux distributions)

---

## Architecture

```
[ Jump Host: thor@jumphost ]
          |
          |  SSH (password auth)
          v
[ App Server 1: tony@stapp01 ]
          |
          |  sudo useradd
          v
[ /etc/passwd: kareem:x:1002:1002::/home/kareem:/sbin/nologin ]
```

The workflow is executed entirely through SSH from the designated jump host, following a standard bastion-host access pattern used in enterprise network segmentation.

---

## Step 1: Connect to the Target Server

Initiate an SSH session from the jump host to App Server 1 using the authorized user account `tony`. On first connection, SSH will prompt for host key verification. Confirm and proceed. The host key is then permanently added to the known hosts file for future sessions.

```bash
ssh tony@stapp01.stratos.xfusioncorp.com
```

**Expected Prompts and Responses:**

- Host authenticity warning: type `yes` to accept and permanently record the ED25519 fingerprint
- Password prompt: enter `tony`'s password

Once authenticated, you will land at the `stapp01` shell prompt, confirming successful access.

**Screenshot: SSH login to stapp01 from jump host**

![SSH login to stapp01](https://github.com/user-attachments/assets/962db087-ee56-4ffe-a993-148b17d66c36)

*The terminal shows a successful SSH connection from the jump host to `stapp01.stratos.xfusioncorp.com (172.17.0.7)`, with the host key accepted and the user `tony` authenticated.*

> **Operational Note:** In production, prefer SSH key-based authentication over passwords. Key-based auth eliminates credential exposure in transit and is enforced by most enterprise security policies (CIS Benchmark L1, NIST 800-53 IA-5).

---

## Step 2: Create the User with a Non-Interactive Shell

Execute the `useradd` command with the `-s` flag to specify `/sbin/nologin` as the account's login shell. This is the critical control that disables interactive access.

```bash
sudo useradd -s /sbin/nologin kareem
```

**Command Breakdown:**

| Flag/Argument | Purpose |
|---|---|
| `sudo` | Elevates to root privilege for system-level account creation |
| `useradd` | Standard Linux utility for adding new user accounts |
| `-s /sbin/nologin` | Sets the login shell to the nologin binary, blocking interactive sessions |
| `kareem` | The username to be created |

On first `sudo` invocation in a new session, the system displays the standard sudo usage reminder. Enter `tony`'s password when prompted. A silent return to the shell prompt (no error output) confirms the account was created successfully.

**Screenshot: Executing `sudo useradd -s /sbin/nologin kareem`**

![useradd command execution](https://github.com/user-attachments/assets/673bea01-484d-43c7-887b-37f1558f8190)

*The command completes without errors. The absence of output after sudo password entry confirms successful user creation.*

> **Edge Case:** If the username already exists, `useradd` will return `useradd: user 'kareem' already exists`. In that case, use `usermod -s /sbin/nologin kareem` to update the shell of the existing account rather than recreating it.

---

## Step 3: Verify User Configuration

Query the system's Name Service Switch (NSS) database using `getent` to confirm the account was created with the correct attributes, including the expected UID, GID, home directory, and shell assignment.

```bash
getent passwd kareem
```

**Expected Output:**

```
kareem:x:1002:1002::/home/kareem:/sbin/nologin
```

**Output Field Reference:**

| Field | Value | Meaning |
|---|---|---|
| Username | `kareem` | Account name |
| Password | `x` | Password hash stored in `/etc/shadow` |
| UID | `1002` | Unique user identifier |
| GID | `1002` | Primary group identifier |
| GECOS | *(empty)* | Comment/description field (intentionally blank for service accounts) |
| Home Directory | `/home/kareem` | Default home path created by `useradd` |
| Shell | `/sbin/nologin` | Non-interactive shell, blocking login |

**Screenshot: `getent passwd kareem` confirming `/sbin/nologin`**

![getent passwd output](https://github.com/user-attachments/assets/a1bf8a6e-5e55-434f-affc-d67ca1bd42bf)

*The `getent` output confirms the `kareem` account entry in the system password database, with `/sbin/nologin` correctly set as the shell.*

> **Why `getent` over `cat /etc/passwd`?** `getent` queries NSS, which includes LDAP, NIS, and other directory sources in addition to local files. This makes it the correct tool for verifying user existence in any environment, not just purely local systems.

---

## Step 4: Confirm Login Access Is Disabled

Attempt to switch to the `kareem` account using `su`. This test validates that the non-interactive shell configuration is enforced at the OS level.

```bash
su - kareem
```

**Expected Output:**

```
Password:
su: Authentication failure
```

The `su` command prompts for a password. Even if a password were supplied, `/sbin/nologin` would reject the session. Since no password has been set for this service account, authentication fails immediately, confirming that **interactive access is fully blocked**.

**Screenshot: `su - kareem` resulting in authentication failure**

![su authentication failure](https://github.com/user-attachments/assets/fd9ee0da-2144-4556-98f2-c6b64760b416)

*The `su - kareem` attempt fails with `su: Authentication failure`, demonstrating that the account cannot be used for interactive login under any circumstance.*

> **Note:** Even if a password were set on this account, `/sbin/nologin` would still reject interactive sessions initiated via SSH or TTY. The shell binary itself prints a configurable message (set in `/etc/nologin.txt`) and exits with a non-zero status.

---

## Result

| Objective | Status |
|---|---|
| User `kareem` created | Confirmed via `getent passwd` |
| Shell set to `/sbin/nologin` | Confirmed in passwd entry |
| Home directory provisioned | `/home/kareem` created by default |
| Interactive login blocked | Confirmed via `su` rejection |
| System security requirement met | Validated end-to-end |

The service account `kareem` is fully provisioned and hardened for use in automated, non-interactive workloads on `stapp01`.

---

## Security Best Practices

- **Always use `/sbin/nologin` or `/bin/false` for service accounts.** `/sbin/nologin` is preferred as it provides a user-facing message; `/bin/false` exits silently.
- **Never assign a password to service accounts** unless a specific inter-process authentication mechanism requires it.
- **Do not add service accounts to sudoers** unless absolutely required by the workload, and if so, scope the grant to specific commands only.
- **Audit service accounts regularly.** Use `awk -F: '$7 ~ /nologin|false/ {print}' /etc/passwd` to list all non-interactive accounts and confirm they are still necessary.
- **Restrict home directory permissions** on service account home directories to `700` or remove the home directory entirely for accounts that do not require filesystem state: `useradd -M -s /sbin/nologin kareem`.
- **Follow CIS Benchmark guidelines** for account management on all Linux servers in production.

---

## Operational Considerations and Edge Cases

**If the account already exists:**
```bash
usermod -s /sbin/nologin kareem
```
Use `usermod` to update the shell without recreating the account, preserving UID, GID, and any existing file ownership.

**If `/sbin/nologin` is not present on the system:**
```bash
which nologin
# Common paths: /sbin/nologin, /usr/sbin/nologin
```
Verify the correct path before executing `useradd`. Using an invalid shell path will still create the account but may cause unexpected behavior.

**For accounts that should never own files:**
```bash
sudo useradd -r -M -s /sbin/nologin kareem
```
The `-r` flag creates a system account (UID below the `UID_MIN` threshold defined in `/etc/login.defs`) and `-M` skips home directory creation, which is appropriate for daemon-style service accounts.

**Troubleshooting sudo first-use delay:**
The sudo usage reminder (the "three rules" message) only appears on the first `sudo` invocation per session. If it blocks automation scripts, suppress it with `sudo -n` or configure `Defaults:tony !lecture` in `/etc/sudoers`.

---

## Real-World Relevance

This pattern is directly applicable across a wide range of production scenarios:

- **Backup agents** (Veeam, Bacula, Amanda) run under restricted identities to access filesystem paths without login capability
- **CI/CD runners** (Jenkins agents, GitLab runners) operate as service accounts with tightly scoped permissions
- **Monitoring collectors** (Prometheus node exporter, Datadog agent) require system-level read access but no interactive shell
- **Database service accounts** for PostgreSQL, MySQL, and similar services are created with non-interactive shells as a baseline OS-level control
- **Security compliance frameworks** (SOC 2, ISO 27001, PCI-DSS) mandate that service accounts not have interactive login capability as a baseline control

---

## Skills Demonstrated

- **Linux User Management:** `useradd`, `usermod`, `getent`, `/etc/passwd` interpretation
- **Bastion Host Access Patterns:** Jump host to target server SSH chaining
- **System Security Hardening:** Enforcing non-interactive shells for service identities
- **Service Account Lifecycle Management:** Creation, verification, and access validation
- **Principle of Least Privilege:** Scoping account capabilities to operational requirements only
- **Enterprise Linux Administration:** Production-grade account provisioning aligned with security policy



























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

kareem:x:1002:1002::/home/kareem:/sbin/nologin

📸 Screenshot:

<img width="1033" height="815" alt="image" src="https://github.com/user-attachments/assets/a1bf8a6e-5e55-434f-affc-d67ca1bd42bf" />

passwd entry showing /sbin/nologin

## Step 4: Confirm Login Is Disabled
- ATTEMPT user login
- EXPECT access denial
- su - kareem

Expected Result:

Login attempt blocked.

📸 Screenshot:

<img width="1043" height="429" alt="image" src="https://github.com/user-attachments/assets/fd9ee0da-2144-4556-98f2-c6b64760b416" />

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

- Enterprise Linux administration
