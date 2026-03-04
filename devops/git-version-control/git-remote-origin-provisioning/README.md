# Git Remote Provisioning: xFusionCorp Nautilus Demo Repository

> **Domain:** DevOps | Git Version Control | Remote Repository Management
> **Environment:** Stratos DC | xFusionCorp Infrastructure
> **Complexity:** Intermediate
> **Status:** Resolved

---

## Table of Contents

- [Overview](#overview)
- [Infrastructure Reference](#infrastructure-reference)
- [Problem Statement](#problem-statement)
- [Root Causes Identified](#root-causes-identified)
- [Prerequisites](#prerequisites)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Phase 1: Access the Storage Server](#phase-1-access-the-storage-server)
  - [Phase 2: Verify Repository State](#phase-2-verify-repository-state)
  - [Phase 3: Add the New Git Remote](#phase-3-add-the-new-git-remote)
  - [Phase 4: Copy, Stage, and Commit](#phase-4-copy-stage-and-commit)
  - [Phase 5: Push to Remote Origin](#phase-5-push-to-remote-origin)
- [Errors Encountered and Fixes Applied](#errors-encountered-and-fixes-applied)
- [Verification](#verification)
- [Key Takeaways](#key-takeaways)


---

## Overview

This document details the end-to-end resolution of a Git remote provisioning task performed on the xFusionCorp Stratos DC storage server. The objective was to configure a new Git remote on an existing cloned repository, commit a tracked file, and push the master branch to the newly registered remote origin.

The task required navigating two non-trivial permission barriers that are common in shared enterprise Linux environments: a Git safe directory ownership conflict and a root-owned `.git/config` file that blocked standard user writes. Both are documented with their root causes and applied fixes.

---

## Infrastructure Reference

| Server Name | IP Address | Hostname | User | Purpose |
|---|---|---|---|---|
| jump_host | Dynamic | jump_host.stratos.xfusioncorp.com | thor | Jump Server to Access Stratos DC |
| ststor01 | 172.16.238.15 | ststor01.stratos.xfusioncorp.com | natasha | Nautilus Storage Server |

> **Note:** All Git operations were performed on `ststor01`. The jump host served only as the SSH entry point into the Stratos DC network.

---

## Problem Statement

The xFusionCorp development team maintains a project under the bare repository `/opt/demo.git`, cloned to `/usr/src/kodekloudrepos/demo` on the storage server. A new bare repository `/opt/xfusioncorp_demo.git` was provisioned on the same server by the DevOps team. The following three tasks were required:

1. Add a new Git remote named `dev_demo` pointing to `/opt/xfusioncorp_demo.git` inside the cloned repo at `/usr/src/kodekloudrepos/demo`
2. Copy `/tmp/index.html` into the repository, then stage and commit it to the `master` branch
3. Push the `master` branch to the new `dev_demo` remote origin

---

## Root Causes Identified

### Issue 1: Git Safe Directory Ownership Conflict

**Error:**
```
fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/demo'
```

**Cause:** The repository at `/usr/src/kodekloudrepos/demo` was cloned or owned by a different user (root). Git 2.35.2+ introduced a security check that prevents users from operating on repositories owned by another UID. Since `natasha` did not own the directory, Git refused all operations.

**Fix:** Register the directory as a safe exception in natasha's global Git config.

---

### Issue 2: Permission Denied on `.git/config`

**Error:**
```
error: could not lock config file .git/config: Permission denied
fatal: could not set 'remote.dev_demo.url' to '/opt/xfusioncorp_demo.git'
```

**Cause:** The `.git/config` file and the `.git/` directory itself were owned by root, making them unwritable by `natasha`. All Git commands that modify repository configuration (remote add, git add, git commit) required elevated privileges.

**Fix:** Prefix all Git write operations with `sudo` for the duration of this task.

---

## Prerequisites

- SSH access to `jump_host` with user `thor`
- SSH access from jump_host to `ststor01` with user `natasha` and sudo privileges
- The bare repository `/opt/xfusioncorp_demo.git` must already exist on `ststor01`
- The file `/tmp/index.html` must already exist on `ststor01`
- Git installed on `ststor01` (version 2.35.2 or later assumed)

---

## Resolution Walkthrough

### Phase 1: Access the Storage Server

SSH from the jump host into the storage server using natasha's credentials.

```bash
# From local machine: access the jump host
ssh thor@jump_host.stratos.xfusioncorp.com
# Password: mjolnir123
```

```bash
# From jump_host: access the storage server
ssh natasha@172.16.238.15

```

***Screenshot: Successful SSH login to ststor01 from jump_host***

<img width="1026" height="440" alt="image" src="https://github.com/user-attachments/assets/392bd8ef-c14c-4177-ae3a-d6b3a4b5a8c6" />

---

### Phase 2: Verify Repository State

Navigate into the cloned repository and verify the current remotes, active branch, and source file before making any changes.

```bash
# Navigate to the repository
cd /usr/src/kodekloudrepos/demo
```

**Attempt to list remotes (triggers ownership error):**

```bash
git remote -v
# fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/demo'
```

**Apply safe directory fix:**

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/demo
```

**Re-check remotes after fix:**

```bash
git remote -v
# origin  /opt/demo.git (fetch)
# origin  /opt/demo.git (push)
```

**Confirm active branch is master:**

```bash
git branch
# * master
```

**Confirm source file exists:**

```bash
ls -lh /tmp/index.html
# -rw-r--r-- 1 root root 120 Mar  4 04:37 /tmp/index.html
```

***Screenshot: Safe directory fix applied and pre-conditions verified***

> ![Pre-condition Verification](./screenshots/02-preconditions-verified.png)

---

### Phase 3: Add the New Git Remote

Attempt to add the remote without elevated privileges, observe the permission error, then apply the fix.

**Attempt without sudo (fails):**

```bash
git remote add dev_demo /opt/xfusioncorp_demo.git
# error: could not lock config file .git/config: Permission denied
# fatal: could not set 'remote.dev_demo.url' to '/opt/xfusioncorp_demo.git'
```

**Add remote with sudo:**

```bash
sudo git remote add dev_demo /opt/xfusioncorp_demo.git
# Password: Bl@kW
```

**Verify both remotes are registered:**

```bash
sudo git remote -v
# dev_demo        /opt/xfusioncorp_demo.git (fetch)
# dev_demo        /opt/xfusioncorp_demo.git (push)
# origin  /opt/demo.git (fetch)
# origin  /opt/demo.git (push)
```

***Screenshot: Permission error on git remote add, then successful resolution with sudo***

> ![Remote Add Permission Error and Fix](./screenshots/03-remote-add-permission-fix.png)

***Screenshot: Both remotes confirmed with git remote -v***

> ![Remotes Verified](./screenshots/04-remotes-verified.png)

---

### Phase 4: Copy, Stage, and Commit

Copy the source file into the repository, stage it, verify the staging area, then commit to master.

**Copy the file:**

```bash
sudo cp /tmp/index.html .
```

**Stage the file:**

```bash
sudo git add index.html
```

**Verify staging area:**

```bash
sudo git status
# On branch master
# Your branch is up to date with 'origin/master'.
#
# Changes to be committed:
#   (use "git restore --staged <file>..." to unstage)
#         new file:   index.html
```

***Screenshot: index.html staged and confirmed as new file***

> ![File Staged](./screenshots/05-file-staged.png)

**Commit to master:**

```bash
sudo git commit -m "Add index.html from /tmp for xFusionCorp demo update"
# [master d0c7875] Add index.html from /tmp for xFusionCorp demo update
#  1 file changed, 10 insertions(+)
#  create mode 100644 index.html
```

***Screenshot: Successful commit on master branch***

> ![Commit Successful](./screenshots/06-commit-successful.png)

---

### Phase 5: Push to Remote Origin

Push the master branch to the newly registered `dev_demo` remote.

```bash
sudo git push dev_demo master
```

**Expected output:**

```
Enumerating objects: 6, done.
Counting objects: 100% (6/6), done.
Delta compression using up to 16 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 615 bytes | 615.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/xfusioncorp_demo.git
 * [new branch]      master -> master
```

***Screenshot: Successful push to dev_demo remote with master branch confirmed***

> ![Push Successful](./screenshots/07-push-successful.png)

---

## Errors Encountered and Fixes Applied

| # | Error | Cause | Fix Applied |
|---|---|---|---|
| 1 | `fatal: detected dubious ownership` | Repository owned by a different UID than the active user | `git config --global --add safe.directory <path>` |
| 2 | `could not lock config file .git/config: Permission denied` | `.git/` directory and config owned by root | Prefix all Git write commands with `sudo` |

---

## Verification

After completing all phases, the following confirms successful task completion:

```bash
sudo git log --oneline -3
# d0c7875 Add index.html from /tmp for xFusionCorp demo update

sudo git remote -v
# dev_demo        /opt/xfusioncorp_demo.git (fetch)
# dev_demo        /opt/xfusioncorp_demo.git (push)
# origin  /opt/demo.git (fetch)
# origin  /opt/demo.git (push)
```

***Screenshot: Final verification showing commit log and remote configuration***

> ![Final Verification](./screenshots/08-final-verification.png)

---

## Key Takeaways

**Git safe directory policy (CVE-2022-24765):** Git 2.35.2 introduced ownership checks as a security hardening measure. In shared enterprise environments where repositories are initialized or cloned by root and accessed by service accounts, registering safe directories in the global config is the correct and intended resolution.

**sudo for root-owned Git repositories:** When `.git/config` is owned by root, all configuration writes including `git remote add`, `git add`, and `git commit` require `sudo`. This is expected behavior in tightly controlled lab and production server environments where the repository was provisioned by a privileged user.

**Verification at every step:** Confirming remotes, branch, and staging state before executing write operations prevents cascading errors and is standard practice for production Git workflows.

---



<img width="1034" height="488" alt="image" src="https://github.com/user-attachments/assets/d6503834-5336-46d2-9f69-fc029426617f" />
<img width="1026" height="496" alt="image" src="https://github.com/user-attachments/assets/ea39a1c2-bda5-45d5-a531-c7e0724f6df9" />
<img width="1032" height="600" alt="image" src="https://github.com/user-attachments/assets/3ab4ea2d-af6d-4e42-b953-f8da8543d5d6" />
<img width="1032" height="555" alt="image" src="https://github.com/user-attachments/assets/a16f276b-28a7-42e5-b0a3-bd5a7d9dd613" />
<img width="1026" height="647" alt="image" src="https://github.com/user-attachments/assets/6c926983-78cd-4f5c-8e92-bb404571a92d" />
<img width="1013" height="586" alt="image" src="https://github.com/user-attachments/assets/d373db81-f36a-47e9-8a08-103b70dad27e" />
<img width="1030" height="750" alt="image" src="https://github.com/user-attachments/assets/0398c09b-ef0e-472c-88dc-c372b62fcc18" />
<img width="1034" height="811" alt="image" src="https://github.com/user-attachments/assets/6cdb860b-cf6c-4949-8e2b-edd638d7383b" />
<img width="1023" height="817" alt="image" src="https://github.com/user-attachments/assets/e7c89aa0-747d-4292-a7a0-8bd1427da0f9" />
<img width="1027" height="856" alt="image" src="https://github.com/user-attachments/assets/1b51fe4e-c524-48fa-b144-7fed5626dd9a" />
<img width="1033" height="756" alt="image" src="https://github.com/user-attachments/assets/e9cfeba5-c080-4995-be58-a2a2e3d9b36d" />
<img width="1030" height="685" alt="image" src="https://github.com/user-attachments/assets/3ffc9631-45b1-4a7c-a1cc-31d66505d05c" />
<img width="1035" height="600" alt="image" src="https://github.com/user-attachments/assets/29a20588-f94e-41b4-a7a2-13ca3e90f7bb" />


