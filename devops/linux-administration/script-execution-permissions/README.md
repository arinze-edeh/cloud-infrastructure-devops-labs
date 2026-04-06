# Linux File Permission Hardening: Granting Execute Access to a Shell Script Across All Users

## Table of Contents

- [Overview](#overview)
- [Environment Details](#environment-details)
- [Objective](#objective)
- [Problem Statement](#problem-statement)
- [Architecture and Access Model](#architecture-and-access-model)
- [Implementation](#implementation)
  - [Step 1: Connect to the Jump Host](#step-1-connect-to-the-jump-host)
  - [Step 2: SSH into App Server 1 (stapp01)](#step-2-ssh-into-app-server-1-stapp01)
  - [Step 3: Verify Script Existence and Inspect Initial Permissions](#step-3-verify-script-existence-and-inspect-initial-permissions)
  - [Step 4: Grant Execute Permission to All Users via sudo chmod](#step-4-grant-execute-permission-to-all-users-via-sudo-chmod)
  - [Step 5: Verify Final Permission State](#step-5-verify-final-permission-state)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Lessons Learned](#lessons-learned)
- [Result](#result)
- [Tags](#tags)

---

## Overview

This document details the end-to-end process of remedying missing execute permissions on a shell script deployed to a Linux application server within the Stratos Datacenter environment. The script `/tmp/xfusioncorp.sh` existed on the target host but lacked execute bits for all user classes, rendering it non-runnable by any system process or user. The fix was applied via a jump host using standard SSH access and elevated privilege through `sudo`, following the principle of least privilege.

---

## Environment Details

| Component | Value |
|---|---|
| Jump Host | `jumphost.stratos.xfusioncorp.com` |
| Jump Host User | `thor` |
| Target Server | App Server 1 (`stapp01`) |
| Target Server IP | `172.16.238.10` |
| Target User | `tony` |
| Script Name | `xfusioncorp.sh` |
| Script Path | `/tmp/xfusioncorp.sh` |
| Required Permission | Execute for owner, group, and others (`a+rx`) |
| Platform | KodeKloud / Stratos Datacenter (xFusion Corp context) |

---

## Objective

- Confirm the presence of `/tmp/xfusioncorp.sh` on the target server
- Identify and remediate the missing execute permission
- Ensure **owner, group, and all other users** can execute the script
- Validate the final permission state before closing the session

---

## Problem Statement

A shell script (`xfusioncorp.sh`) was pre-deployed to `/tmp` on App Server 1 by the platform provisioning process. However, the file was created without any execute permissions set, resulting in a permission string of `----------`. Any attempt to execute the script by any user or automated process would fail with `Permission denied`. The task requires correcting the permission model to allow universal execution while preserving the existing ownership by `root`.

---

## Architecture and Access Model

Access to the Stratos Datacenter application servers is mediated through a dedicated jump host. Direct inbound SSH from external networks is not permitted to application servers. The flow is as follows:

```
Engineer Workstation
        |
        v
jumphost.stratos.xfusioncorp.com  (thor)
        |
        v (SSH over internal network)
stapp01 - 172.16.238.10  (tony)
```

All administrative commands on `stapp01` that require elevated access are executed via `sudo`, following the principle of least privilege. The user `tony` is a member of the `sudoers` group on this host.

---

## Implementation

### Step 1: Connect to the Jump Host

Initiate an SSH session to the jump host using the `thor` account. On first connection, SSH presents the remote host's ED25519 key fingerprint for verification. Accept and add the key to the local `known_hosts` file to establish trust for subsequent connections.

```bash
ssh thor@jumphost.stratos.xfusioncorp.com
```

**Expected behaviour:** SSH prompts for host key confirmation on first connection. Respond `yes` to persist the key. Authentication proceeds with the account password.

> **Screenshot: Jump host authentication and initial session established**

![Jump host SSH session established showing ED25519 fingerprint acceptance and successful login as thor](https://github.com/user-attachments/assets/400833d8-d281-4262-9834-bfab5a99dafd)

---

### Step 2: SSH into App Server 1 (stapp01)

From the jump host, initiate an SSH session to App Server 1 using its internal IP address and the `tony` account. As with the jump host, the host key is verified and accepted on first connection.

```bash
ssh tony@172.16.238.10
```

**Expected behaviour:** SSH presents the ED25519 fingerprint for `172.16.238.10`. Accept and authenticate with the `tony` account password. A successful login lands in tony's home directory on `stapp01`.

> **Screenshot: SSH from jump host to stapp01 showing host key acceptance and successful authentication**

![SSH connection from thor@jumphost to tony@stapp01 showing host key fingerprint acceptance and password authentication](https://github.com/user-attachments/assets/a752e1df-151a-476c-adc9-e82b0b8d609a)

![Continued view of the SSH session showing successful login and shell prompt on stapp01](https://github.com/user-attachments/assets/06281ee9-e830-4eb5-8fef-832ac18e9f8d)

---

### Step 3: Verify Script Existence and Inspect Initial Permissions

Before making any changes, confirm that the target script exists at the expected path and inspect its current permission state using `ls -l`. This establishes a clear baseline and validates the scope of the change required.

```bash
ls -l /tmp/xfusioncorp.sh
```

**Expected output:**

```
---------- 1 root root 40 Feb 10 00:25 /tmp/xfusioncorp.sh
```

The permission string `----------` confirms:
- No execute bit is set for owner, group, or others
- The file is owned by `root:root`
- The file is 40 bytes and was created on Feb 10 at 00:25

The script cannot be run by any user in its current state.

> **Screenshot: Initial ls -l output confirming script exists with no permissions set**

![ls -l output on stapp01 showing /tmp/xfusioncorp.sh with ---------- permissions and root ownership](https://github.com/user-attachments/assets/1d8bfc47-a2ec-48bc-a517-d4e5ab26dc93)

---

### Step 4: Grant Execute Permission to All Users via sudo chmod

Apply execute permissions for all user classes (owner, group, and others) using `chmod` with the `a+x` symbolic mode, executed under `sudo` to satisfy the root ownership constraint. At this stage, `a+x` grants execute only. The subsequent requirement to ensure read access for all users is addressed in the jump host phase using `a+rx`.

```bash
sudo chmod +x /tmp/xfusioncorp.sh
```

**What this command does:**

- `sudo`: Elevates to root-level privileges, required because the file is owned by `root`
- `chmod +x`: Adds the execute bit for owner, group, and others (equivalent to `a+x`)
- `/tmp/xfusioncorp.sh`: The target file

When `sudo` is invoked for the first time in the session, the system displays the standard security lecture and prompts for `tony`'s password.

> **Screenshot: sudo chmod +x execution with sudo password prompt and security lecture**

![sudo chmod +x command on stapp01 with the standard sudo security lecture and password prompt](https://github.com/user-attachments/assets/4a8be63b-45eb-466d-a64f-10e1a18c19c4)

---

### Step 5: Verify Final Permission State

After exiting the `stapp01` session and returning to the jump host, an additional `sudo chmod a+rx` was applied from the jump host targeting the script path. This command failed with `No such file or directory` because `/tmp/xfusioncorp.sh` does not exist on the jump host filesystem. This was an environment-awareness error and is documented in the Errors and Resolutions section.

The correct approach was to re-SSH to `stapp01` and apply `sudo chmod a+rx` there. Run `ls -l` to confirm the final permission state.

```bash
# Re-enter stapp01
ssh tony@172.16.238.10

# Apply read and execute for all users
sudo chmod a+rx /tmp/xfusioncorp.sh

# Verify
ls -l /tmp/xfusioncorp.sh
```

**Expected final output:**

```
-r-xr-xr-x 1 root root 40 Feb 10 00:25 /tmp/xfusioncorp.sh
```

The permission string `-r-xr-xr-x` confirms:
- **Owner (root):** read and execute
- **Group (root):** read and execute
- **Others:** read and execute
- No write permission is granted to any class, which is correct and secure for a shared script

> **Screenshot: Final ls -l output confirming -r-xr-xr-x permissions on the script**

![Final ls -l on stapp01 showing /tmp/xfusioncorp.sh with -r-xr-xr-x permissions confirming successful remediation](https://github.com/user-attachments/assets/954d1bec-78b7-49ff-81fd-2a55a34d2499)

![Full terminal session view showing the complete chmod workflow from permission check through to final verification and session exit](https://github.com/user-attachments/assets/17ca42ed-05af-4a64-87d7-b86bdf560499)

---

## Errors and Resolutions

### Error: `chmod: cannot access '/tmp/xfusioncorp.sh': No such file or directory`

**Context:** After exiting `stapp01`, the command `sudo chmod a+rx /tmp/xfusioncorp.sh` was executed from the jump host (`thor@jumphost`).

**Root cause:** The script `/tmp/xfusioncorp.sh` exists on `stapp01`, not on the jump host. The `/tmp` directory is local to each host. Executing `chmod` on the jump host attempts to resolve the path on the jump host filesystem, where the file does not exist.

**Resolution:** Re-established an SSH session to `stapp01` and ran `sudo chmod a+rx /tmp/xfusioncorp.sh` from within the correct host context. The command completed successfully.

**Lesson:** Always verify the active shell context (hostname shown in the prompt) before executing administrative commands. The prompt prefix (`thor@jumphost` vs `tony@stapp01`) is the definitive indicator of which host's filesystem is in scope.

---

## Key Decisions

**Use of `a+rx` over `777`:** The `chmod a+rx` symbolic mode grants only read and execute permissions universally, without write access. Using `777` would have granted write access to all users, creating an unnecessary security risk for a shared executable. Minimal permission grants aligned with the principle of least privilege were applied.

**`sudo` over direct root login:** The `tony` account was used throughout with `sudo` escalation for privileged commands. This preserves an audit trail in the system logs (`/var/log/secure` or `/var/log/auth.log`) associating elevated actions with the `tony` identity, rather than using a shared `root` session with no individual accountability.

**Baseline verification before change:** Running `ls -l` before and after the `chmod` operation establishes a documented before/after state, which is essential for change control processes and incident tracing.

**Jump host as mandatory access gateway:** No direct connection to `stapp01` was attempted from an external network. All access was routed through the jump host, consistent with the Stratos Datacenter network security model.

---

## Lessons Learned

- **Shell prompt context is critical.** Before running any file system command, verify the active hostname in the terminal prompt. A single misread can result in commands targeting the wrong host.
- **`/tmp` is host-local.** Files placed in `/tmp` on one host are not shared or mounted across other hosts in the environment. Do not assume cross-host path visibility.
- **Symbolic chmod modes are safer than octal for targeted changes.** Using `a+rx` modifies only the specified bits without touching existing permission bits. Octal modes (e.g., `chmod 555`) replace the entire permission field, which can inadvertently remove bits not under review.
- **sudo sessions require password re-entry after timeout.** If a `sudo` session has expired between steps, the next `sudo` command will prompt for the account password again. This is expected behaviour and does not indicate an error.
- **Always validate with `ls -l` after permission changes.** Confirming the resulting permission string eliminates ambiguity and provides evidence of a correctly completed change.

---

## Result

| Check | Status |
|---|---|
| Script exists at `/tmp/xfusioncorp.sh` on stapp01 | Confirmed |
| Initial permissions were fully restrictive (`----------`) | Confirmed |
| Execute permission applied for owner, group, and others | Confirmed |
| Final permission string is `-r-xr-xr-x` | Confirmed |
| Task completed with no residual permission issues | Confirmed |

The script is now executable by all users on App Server 1. The change was applied using least-privilege access (`sudo`), validated before and after, and documented with full terminal output.

---

## Tags

`linux` `chmod` `file-permissions` `bash` `ssh` `jump-host` `system-administration` `devops` `stratos-datacenter` `xfusioncorp` `security-hardening`


























# Script Execution Permissions – Linux Administration Lab

## Lab Overview
- This lab demonstrates how to grant executable permissions to a bash script on a remote Linux server while ensuring **all users** can execute the script.  
- The task was performed within the **Stratos Datacenter** environment using a **jump host** for secure access.

---

## Environment Details

| Component        | Value |
|------------------|------|
| Jump Host        | jumphost.stratos.xfusioncorp.com |
| Target Server    | App Server 1 (stapp01) |
| Script Name      | xfusioncorp.sh |
| Script Location  | /tmp/xfusioncorp.sh |
| Required Access  | Execute permission for all users |

---

## Objective

- Grant executable permissions to `/tmp/xfusioncorp.sh`
- Ensure **owner, group, and others** can execute the script
- Validate permissions successfully

---

## High-Level Logic

- CONNECT to jump host
- AUTHENTICATE successfully

- SSH into App Server 1
- VERIFY script exists at target path

- CHECK current permissions
- IF script is not executable:
  -  APPLY execute permissions for all users

- RE-VERIFY permissions
- CONFIRM success

## Step-by-Step Implementation

## Step 1: Connect to Jump Host
- ssh thor@jump_host.stratos.xfusioncorp.com

📸 Screenshot:
<img width="1026" height="556" alt="image" src="https://github.com/user-attachments/assets/400833d8-d281-4262-9834-bfab5a99dafd" />

## Step 2: SSH into App Server 1
- ssh tony@stapp01.stratos.xfusioncorp.com

📸 Screenshot:
<img width="1040" height="531" alt="image" src="https://github.com/user-attachments/assets/a752e1df-151a-476c-adc9-e82b0b8d609a" />
<img width="716" height="513" alt="image" src="https://github.com/user-attachments/assets/06281ee9-e830-4eb5-8fae-832ac18e9f8d" />
<img width="1009" height="802" alt="image" src="https://github.com/user-attachments/assets/b531f87c-1e4b-43c9-bb7c-7f08407f8feb" />

## Step 3: Verify Script Existence
- ls -l /tmp/xfusioncorp.sh

Expected:

- Script exists

- Missing execute (x) permissions

📸 Screenshot:
<img width="958" height="816" alt="image" src="https://github.com/user-attachments/assets/4a8be63b-45eb-466d-a64f-10e1a18c19c4" />

## Step 4: Grant Execute Permission to All Users
- sudo chmod a+x /tmp/xfusioncorp.sh
- a+x ensures user, group, and others can execute the script.

📸 Screenshot:
<img width="976" height="874" alt="image" src="https://github.com/user-attachments/assets/954d1bec-78b7-49ff-81fd-2a55a34d2499" />

## Step 5: Verify Final Permissions
- ls -l /tmp/xfusioncorp.sh

- Expected Output:

-r-xr-xr-x 1 root root ...

📸 Screenshot:
<img width="1043" height="872" alt="image" src="https://github.com/user-attachments/assets/1d8bfc47-a2ec-48bc-a517-d4e5ab26dc93" />
<img width="965" height="877" alt="image" src="https://github.com/user-attachments/assets/17ca42ed-05af-4a64-87d7-b86bdf560499" />

## Result

- Script is executable
- All users have execution permission
- Task completed successfully

## Key Linux Concepts Demonstrated
- Linux file permission model (rwx)

- chmod symbolic mode (a+x)

- Secure SSH access via jump host

- Permission verification using ls -l

##  Tags
`linux` `chmod` `permissions` `devops` `bash` `system-administration`
