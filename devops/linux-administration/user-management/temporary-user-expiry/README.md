# Linux Temporary User Provisioning with Account Expiry Enforcement

> **Enterprise-Style Access Control | Automated Deprovisioning | Time-Bound Credential Management**

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Environment](#architecture-and-environment)
- [Objectives](#objectives)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Connect to the Jump Host](#step-1-connect-to-the-jump-host)
  - [Step 2: SSH into the Target Application Server](#step-2-ssh-into-the-target-application-server)
  - [Step 3: Escalate to Root](#step-3-escalate-to-root)
  - [Step 4: Create the Temporary User with Expiry Date](#step-4-create-the-temporary-user-with-expiry-date)
  - [Step 5: Verify Account Expiry Configuration](#step-5-verify-account-expiry-configuration)
  - [Step 6: Validate User Entry in System Database](#step-6-validate-user-entry-in-system-database)
- [Outcome](#outcome)
- [Security Best Practices](#security-best-practices)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting](#troubleshooting)
- [Real-World Relevance](#real-world-relevance)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This document details the provisioning of a **time-bound Linux user account** with an enforced expiry date on a production application server within the xFusionCorp Industries infrastructure. The account is created for a temporary developer assigned to **Project Nautilus** and is configured to automatically expire on `2026-12-07`, ensuring access is revoked without requiring manual intervention.

This procedure follows enterprise-grade access control principles, aligning with **least-privilege**, **zero-standing-access**, and **automated deprovisioning** policies.

---

## Problem Statement

Organizations frequently onboard contractors, short-term developers, and project-scoped personnel who require temporary system access. Without enforced expiry policies, these accounts often persist beyond their intended lifecycle, creating **orphaned credentials**, expanding the **attack surface**, and violating compliance requirements such as SOC 2, ISO 27001, and CIS Benchmarks.

**Solution:** Leverage Linux native `useradd -e` and `chage` tooling to provision accounts with built-in expiry, ensuring automatic deprovisioning without relying on manual audits or external tooling.

---

## Architecture and Environment

| Attribute | Value |
|---|---|
| **Project** | Nautilus - xFusionCorp Industries |
| **Target Host** | App Server 3 (stapp03) |
| **Target IP** | 172.17.0.7 |
| **Jump Host** | jump_host.stratos.xfusioncorp.com (172.16.238.2) |
| **Operating System** | Linux (RHEL/CentOS-based) |
| **Privilege Model** | sudo escalation via `banner` user |
| **New Username** | `mariyam` |
| **Account Expiry Date** | `2026-12-07` |

---

## Objectives

- Create a user account named `mariyam` (lowercase) on `stapp03`
- Enforce an account expiry date of `2026-12-07` using `useradd -e`
- Verify expiry configuration using `chage -l`
- Confirm user entry exists in the system identity database using `getent passwd`

---

## Prerequisites

- SSH access to the jump host (`jump_host.stratos.xfusioncorp.com`) as user `thor`
- Valid credentials for user `banner` on `stapp03`
- `sudo` privileges for `banner` on the target server
- Network path from jump host to `stapp03` (172.17.0.7) is open on port 22

---

## Implementation

### Step 1: Connect to the Jump Host

Initiate an SSH session from the local workstation to the designated jump host. This is the secure ingress point for all internal infrastructure access.

```bash
ssh thor@jump_host.stratos.xfusioncorp.com
```

On first connection, SSH will present a host key fingerprint for verification. Accept and proceed. The key is then permanently added to `~/.ssh/known_hosts`.

**What happens here:**
- ED25519 host key fingerprint is presented for verification
- Answer `yes` to proceed and persist the key to `known_hosts`
- Authenticate with the password for user `thor`

![Step 1 - SSH connection to jump host](https://github.com/user-attachments/assets/4fdaa99f-8114-4747-a020-95d183d34738)
*SSH session established to the jump host. Host key fingerprint is verified and recorded in known_hosts.*

---

### Step 2: SSH into the Target Application Server

From the jump host, establish a secondary SSH hop directly to `stapp03` using the `banner` account. The jump host acts as a bastion, isolating direct public access to application servers.

```bash
ssh banner@stapp03.stratos.xfusioncorp.com
```

**What happens here:**
- A new host key fingerprint for `stapp03` (172.17.0.7) is presented
- Accept and authenticate with the password for `banner`
- Shell session opens on `stapp03`

![Step 2 - SSH from jump host to stapp03](https://github.com/user-attachments/assets/66e1854a-ccb3-44ea-9604-4421c8d85d89)
*SSH hop from jump host to stapp03 completed. Session is now active on the target application server as user `banner`.*

---

### Step 3: Escalate to Root

To create system users, root-level privileges are required. Use `sudo -i` to open an interactive root shell.

```bash
sudo -i
```

Enter the password for `banner` when prompted. The system presents the standard sudo policy reminder.

**Why `sudo -i` over `sudo su`:**
- `sudo -i` simulates a clean root login shell, loading root's environment properly
- Avoids inheriting the unprivileged user's environment variables
- Preferred in enterprise environments for predictable and auditable privilege escalation

![Step 3 - Privilege escalation via sudo -i](https://github.com/user-attachments/assets/e42b3d22-a924-41d9-b5ab-4c9971dd5017)
*Root shell obtained via `sudo -i`. Prompt changes from `[banner@stapp03 ~]$` to `[root@stapp03 ~]#`, confirming full privilege escalation.*

---

### Step 4: Create the Temporary User with Expiry Date

Create the user `mariyam` with the `-e` flag to set an account expiry date. The `useradd` command atomically creates the user and binds the expiry at creation time.

```bash
useradd -e 2026-12-07 mariyam
```

**Flag breakdown:**

| Flag | Purpose |
|---|---|
| `-e 2026-12-07` | Sets the account expiry date in `YYYY-MM-DD` format |
| `mariyam` | Username in lowercase as per naming convention |

**What happens under the hood:**
- A new entry is written to `/etc/passwd` and `/etc/shadow`
- The shadow entry encodes the expiry date as days since the Unix epoch (1970-01-01)
- The home directory `/home/mariyam` is created with default skeleton files from `/etc/skel`
- No password is set by default; the account is locked until a password is assigned

> **Note:** A silent return (no output) after `useradd` indicates successful execution on RHEL/CentOS-based systems.

![Step 4 - useradd command execution](https://github.com/user-attachments/assets/a5a58504-a67e-4fc4-ac76-fe00c147bb05)
*`useradd -e 2026-12-07 mariyam` executed successfully. No output confirms the command completed without errors.*

---

### Step 5: Verify Account Expiry Configuration

Use the `chage` utility to query and confirm the account aging parameters for `mariyam`. This is the authoritative method for verifying expiry settings against the `/etc/shadow` database.

```bash
chage -l mariyam
```

**Expected output:**

```
Last password change                : Feb 02, 2026
Password expires                    : never
Password inactive                   : never
Account expires                     : Dec 07, 2026
Minimum number of days between password change : 0
Maximum number of days between password change : 99999
Number of days of warning before password expires : 7
```

**Key field to validate:**

- **`Account expires`** must read `Dec 07, 2026` to confirm the expiry is correctly encoded

![Step 5 - chage output showing account expiry](https://github.com/user-attachments/assets/5ab00ae9-4f77-4560-ae7b-969ca4f247a9)
*`chage -l mariyam` output confirms the account expiry is correctly set to `Dec 07, 2026`. All other aging fields reflect system defaults.*

---

### Step 6: Validate User Entry in System Database

Confirm the user exists as a valid system identity by querying the Name Service Switch (NSS) via `getent`. This validates both the `/etc/passwd` entry and ensures the system resolver recognizes the account.

```bash
getent passwd mariyam
```

**Expected output:**

```
mariyam:x:1002:1002::/home/mariyam:/bin/bash
```

**Field breakdown:**

| Field | Value | Meaning |
|---|---|---|
| Username | `mariyam` | Account name (lowercase confirmed) |
| Password | `x` | Password hash stored in `/etc/shadow` |
| UID | `1002` | User ID assigned by system |
| GID | `1002` | Primary group ID |
| GECOS | (empty) | Full name / comment field |
| Home | `/home/mariyam` | Home directory path |
| Shell | `/bin/bash` | Default login shell |

![Step 6 - getent passwd confirming user creation](https://github.com/user-attachments/assets/590e2931-6957-4019-a427-007da6eccfea)
*`getent passwd mariyam` returns a valid passwd-format entry, confirming the account is correctly provisioned in the system identity database.*

---

## Outcome

| Requirement | Status |
|---|---|
| User `mariyam` created on `stapp03` | Completed |
| Username in lowercase | Confirmed |
| Account expiry set to `2026-12-07` | Confirmed via `chage -l` |
| User exists in system database | Confirmed via `getent passwd` |
| Automatic access revocation configured | Enforced by kernel-level account expiry |

---

## Security Best Practices

- **Always set expiry dates for temporary users.** Do not rely on manual deprovisioning; encode the expiry at account creation using `useradd -e` or `chage -E`.
- **Assign minimal privileges.** Temporary users should have no `sudo` access unless explicitly required. Audit `sudoers` after each provisioning cycle.
- **Set a strong initial password or use SSH key-only authentication.** Password-less accounts with an unlocked shell are a security risk.
- **Audit expiring accounts regularly.** Use `chage -l <username>` or automate with scripts that check `/etc/shadow` for approaching expiry dates.
- **Remove home directories when deprovisioning.** After account expiry, manually remove `/home/<username>` and any user-owned cron jobs or processes.
- **Log all privileged actions.** Ensure `auditd` or equivalent is capturing `useradd` and `sudo` events for compliance traceability.

---

## Operational Considerations

- **Account expiry vs. password expiry are distinct.** The `-e` flag in `useradd` sets account expiry (system-level lockout). Password expiry is a separate mechanism controlled via `chage -M`. Ensure both policies are aligned for temporary accounts.
- **Time zone awareness.** Account expiry is evaluated at login time relative to the system clock and timezone. Confirm the server timezone matches the intended expiry interpretation.
- **Locked vs. expired accounts.** An expired account (`Account expires`) prevents login even if the password is valid. This is distinct from a locked account (`passwd -l`). Both mechanisms can coexist.
- **Automation at scale.** In environments with multiple temporary users, consider integrating with an Identity Provider (IdP) or using Ansible/Terraform to codify provisioning and enforce expiry policies consistently.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `useradd: user 'mariyam' already exists` | Account was previously created | Verify with `getent passwd mariyam`, then use `usermod -e 2026-12-07 mariyam` to update expiry |
| `chage` shows `Account expires: never` | `-e` flag was not passed, or wrong format used | Run `chage -E 2026-12-07 mariyam` to correct |
| `getent passwd mariyam` returns nothing | User creation failed silently, or NSS cache stale | Re-run `useradd`, or run `nscd -i passwd` to flush the NSS cache |
| SSH to stapp03 refused | Firewall rule or SSH daemon not running | Confirm port 22 is open from jump host; check `sshd` status on target |
| `sudo -i` fails for banner | banner not in sudoers | Escalate to system owner; do not attempt to bypass |

---

## Real-World Relevance

This procedure directly mirrors enterprise operational patterns including:

- **Contractor onboarding:** Temporary developers receive time-bound accounts that expire automatically at the end of their engagement, reducing the risk of orphaned credentials.
- **Compliance requirements:** Frameworks such as SOC 2 Type II, PCI-DSS, and CIS Controls mandate regular access reviews and automated deprovisioning for non-permanent staff.
- **Zero-trust principles:** Time-limited access aligns with zero-trust architecture by ensuring no standing access persists beyond the minimum required window.
- **Security incident reduction:** Expired accounts cannot be used for lateral movement or privilege escalation, directly reducing breach impact in the event of credential compromise.

---

## Skills Demonstrated

- **Bastion host navigation:** Multi-hop SSH access through a secure jump host to isolated internal servers
- **Linux user lifecycle management:** Account creation, expiry enforcement, and validation using native Linux tooling (`useradd`, `chage`, `getent`)
- **Privilege escalation:** Secure and auditable root shell access via `sudo -i`
- **Account expiration policy:** Understanding and applying the distinction between account expiry, password expiry, and account locking
- **System identity verification:** Querying NSS to confirm user provisioning at the OS resolver level
- **Enterprise access control:** Applying time-bound access patterns consistent with zero-trust and compliance-driven security postures
































# Linux User Management – Temporary User with Expiry Date

## Overview
This lab demonstrates how to create a temporary Linux user account with
a predefined expiry date. Expiry-based users are commonly used for
contractors, temporary developers, and short-term project assignments
to enforce automatic access revocation.

---

## Lab Context
Project Nautilus – xFusionCorp Industries  
Requirement: Grant temporary access to a developer assigned to the Nautilus project.

Target Server:
- App Server 3 (stapp03)

---

## Objective
- Create a user named `mariyam` (lowercase)
- Set an account expiry date of `2026-12-07`
- Ensure the account automatically disables after expiry
- Verify correct configuration

---

## Environment
- OS: Linux
- Server Role: Application Server
- Privileges: sudo access

---

## Step 1: Connect to Target Server

- CONNECT to jump host
- SSH into App Server 3 using provided credentials

📸 Screenshot:

<img width="1035" height="743" alt="image" src="https://github.com/user-attachments/assets/4fdaa99f-8114-4747-a020-95d183d34738" />
<img width="1034" height="832" alt="image" src="https://github.com/user-attachments/assets/66e1854a-ccb3-44ea-9604-4421c8d85d89" />

SSH session connected to stapp03

## Step 2: Create Temporary User with Expiry Date
- EXECUTE useradd command
- SET account expiry date
- USERNAME = mariyam
- EXPIRY_DATE = 2026-12-07
- sudo useradd -e 2026-12-07 mariyam

📸 Screenshot: 
<img width="1044" height="750" alt="image" src="https://github.com/user-attachments/assets/e42b3d22-a924-41d9-b5ab-4c9971dd5017" />


useradd command execution

## Step 3: Verify Account Expiry Configuration
- QUERY account aging information
- CONFIRM expiry date is correctly applied
- sudo chage -l mariyam

Expected Output:

Account expires : Dec 07, 2026

📸 Screenshot:
<img width="1032" height="765" alt="image" src="https://github.com/user-attachments/assets/a5a58504-a67e-4fc4-ac76-fe00c147bb05" />
<img width="1040" height="784" alt="image" src="https://github.com/user-attachments/assets/5ab00ae9-4f77-4560-ae7b-969ca4f247a9" />
chage output showing expiry date

## Step 4: Validate User Creation
- CONFIRM user exists in system
- VERIFY username is lowercase
- getent passwd mariyam

📸 Screenshot:
<img width="1028" height="840" alt="image" src="https://github.com/user-attachments/assets/590e2931-6957-4019-a427-007da6eccfea" />


passwd entry for mariyam

## Result
- Temporary user created successfully

- Account expiry date enforced

- Automatic access revocation configured

## Security Best Practices
- Always set expiry dates for temporary users

- Review and audit expiring accounts regularly

- Combine expiry with least-privilege access

- Remove unused accounts promptly

## Real-World Relevance
This mirrors enterprise practices where:

- Contractors receive time-bound access

- Compliance requires automatic deprovisioning

- Security teams reduce orphaned accounts

## Skills Demonstrated
- Linux user lifecycle management

- Account expiration policies

- Secure access control

- Enterprise system administration


