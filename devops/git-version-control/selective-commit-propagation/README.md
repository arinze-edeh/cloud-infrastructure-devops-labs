# Git Selective Commit Propagation: Cross-Branch Cherry-Pick on Root-Owned Repository

> **Category:** `devops/git-version-control/selective-commit-propagation`
> **Environment:** Stratos DC | Storage Server `ststor01` | Nautilus Project
> **Complexity:** Intermediate
> **Status:** Resolved

---

## Table of Contents

* [Overview](#overview)
* [Environment Details](#environment-details)
* [Problem Statement](#problem-statement)
* [Root Cause Analysis](#root-cause-analysis)
* [Prerequisites](#prerequisites)
* [Resolution Walkthrough](#resolution-walkthrough)
* [Verification](#verification)
* [Lessons Learned](#lessons-learned)
* [Quick Reference Command Sheet](#quick-reference-command-sheet)

---

## Overview

This document details the end-to-end resolution of a Git cherry-pick task performed on a remote storage server in an enterprise multi-server DevOps environment. The task required selectively propagating a single commit (`Update info.txt`) from the `feature` branch to the `master` branch of the Nautilus project repository, followed by a push to the bare remote at `/opt/official.git`.

The operation surfaced three compounding infrastructure problems that are common in shared enterprise environments: a missing working clone, a Git ownership security block, and a filesystem permission barrier. Each is documented with its exact error, root cause, and resolution.

---

## Environment Details

| Property | Value |
|---|---|
| Jump Host | `thor@jumphost` |
| Target Server | `ststor01.stratos.xfusioncorp.com` |
| Server IP | `172.16.238.15` |
| Login User | `natasha` |
| Working Directory | `/usr/src/kodekloudrepos/official` |
| Bare Remote | `/opt/official.git` |
| Source Branch | `feature` |
| Target Branch | `master` |
| Target Commit Message | `Update info.txt` |
| Target Commit Hash | `d1a149c` |

---

## Problem Statement

The task required cherry-picking the commit with message `Update info.txt` from the `feature` branch into `master` and pushing the result to the bare remote. Upon connecting to the storage server and navigating to the expected repository path, the operation could not proceed due to three sequential blocking issues.

### Screenshot: Initial SSH Connection and Hostname Verification
<img width="1031" height="629" alt="image" src="https://github.com/user-attachments/assets/c5c47667-ded1-4990-827a-f796e9f8a2b3" />

>#### Caption: Successful SSH connection from thor@jumphost to natasha@ststor01
>#### Command shown: ssh natasha@ststor01 && hostname
>#### Expected output: ststor01.stratos.xfusioncorp.com



## Root Cause Analysis

### Problem 1: Working Directory Is Not a Git Repository

**Error Encountered:**
```
fatal: not a git repository (or any of the parent directories): .git
```

**Location:** `/usr/src/kodekloudrepos`

**Root Cause:**
The task specification stated the repo was cloned at `/usr/src/kodekloudrepos`, however the actual cloned repository existed one level deeper at `/usr/src/kodekloudrepos/official`. The parent directory contained no `.git` folder and therefore could not respond to any Git commands.

**Discovery Command:**
```bash
ls -la /usr/src/kodekloudrepos
```

**Output:**
```
drwxr-xr-x 3 root root 4096 Mar  6 05:29 official
```

### Screenshot: Directory Listing Revealing Subdirectory

<img width="1037" height="538" alt="image" src="https://github.com/user-attachments/assets/ab0f421f-0c9c-46b8-9a3a-0aee816a5e2e" />

>#### Caption: ls -la output showing the 'official' subdirectory inside /usr/src/kodekloudrepos
>#### Key detail: All entries owned by root:root

---

### Problem 2: Git Dubious Ownership Security Block

**Error Encountered:**
```
fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/official'
To add an exception for this directory, call:
        git config --global --add safe.directory /usr/src/kodekloudrepos/official
```

**Root Cause:**
Git version 2.35.2 introduced CVE-2022-24765 protection. When the directory owner (`root`) differs from the current user (`natasha`), Git refuses all operations as a security measure to prevent privilege escalation via malicious `.git/config` files in shared directories.

**Resolution:**
```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/official
```

This registers the path as explicitly trusted for the current user's global Git configuration, allowing Git commands to proceed.

### Screenshot: Dubious Ownership Error and Safe Directory Fix

<img width="1032" height="655" alt="image" src="https://github.com/user-attachments/assets/e8282d7d-6e13-4420-9855-afbbd119ad59" />

>#### Caption: Git security block on ownership mismatch and the safe.directory config fix
>#### Commands shown: git status (error), git config --global --add safe.directory ..., git status (success)

---

### Problem 3: Filesystem Permission Denied on Branch Switch

**Error Encountered:**
```
fatal: Unable to create '/usr/src/kodekloudrepos/official/.git/index.lock': Permission denied
```

**Root Cause:**
Although Git now recognised the repository (Problem 2 resolved), the `.git` directory and all its contents were owned by `root`. Git requires write access to `.git/index.lock` to safely perform branch switches and commits. Since `natasha` is not the owner, the write was rejected by the kernel.

**Resolution:**
All Git write operations were prefixed with `sudo`, which elevated privileges to `root` for each command:

```bash
sudo git checkout master
sudo git cherry-pick d1a149c
sudo git push origin master
```

### Screenshot: Permission Denied Error on Checkout Without sudo

<img width="1035" height="507" alt="image" src="https://github.com/user-attachments/assets/32730dd0-7be4-4334-8c3f-54a90b6d0159" />

>#### Caption: git checkout master failing with Permission Denied on .git/index.lock
>#### Key detail: Contrast with the sudo version succeeding immediately after

---

## Prerequisites

* SSH access from `thor@jumphost` to `natasha@ststor01` (password: available from infrastructure details)
* `sudo` privileges for `natasha` on `ststor01`
* Repository already present at `/usr/src/kodekloudrepos/official`
* Bare remote accessible at `/opt/official.git`

---

## Resolution Walkthrough

### Phase 1: Connect to the Storage Server

**Step 1.1** SSH from the jump host to the storage server:

```bash
ssh natasha@ststor01
```

Enter password when prompted. Accept the host fingerprint on first connection by typing `yes`.

**Step 1.2** Verify you are on the correct host:

```bash
hostname
```

Expected output: `ststor01.stratos.xfusioncorp.com`

---

### Phase 2: Locate the Repository

**Step 2.1** Navigate to the documented path:

```bash
cd /usr/src/kodekloudrepos
git status
```

If the error `fatal: not a git repository` appears, the actual repo is in a subdirectory.

**Step 2.2** List the directory contents:

```bash
ls -la /usr/src/kodekloudrepos
```

**Step 2.3** Navigate into the actual repository:

```bash
cd /usr/src/kodekloudrepos/official
```

---

### Phase 3: Resolve Git Ownership Block

**Step 3.1** Attempt `git status` to surface the ownership error:

```bash
git status
```

**Step 3.2** Register the directory as safe for the current user:

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/official
```

**Step 3.3** Confirm Git now responds correctly:

```bash
git status
```

Expected output: `On branch feature` with a clean working tree.

### Screenshot: Git Status After Safe Directory Registration

<img width="1034" height="763" alt="image" src="https://github.com/user-attachments/assets/85380687-9201-49b2-ba02-8a14a1c47244" />

>#### Caption: git status returning 'On branch feature, nothing to commit, working tree clean'
>#### This confirms the ownership block is resolved

---

### Phase 4: Identify the Target Commit

**Step 4.1** View the commit log (currently on the `feature` branch):

```bash
git log --oneline
```

**Output observed:**
```
062203a (HEAD -> feature, origin/feature) Update welcome.txt
d1a149c Update info.txt
34cf452 (origin/master, master) Add welcome.txt
3d326be initial commit
```

**Step 4.2** Record the hash of the target commit:

* Target: `d1a149c` (`Update info.txt`)
* This is the only commit that must be brought into `master`
* `062203a` (`Update welcome.txt`) must remain on `feature` only

### Screenshot: Git Log Showing Branch State and Target Commit

<img width="1038" height="399" alt="image" src="https://github.com/user-attachments/assets/23ed235d-0816-4950-baf6-12e44e3eab81" />

>#### Caption: git log --oneline output showing all four commits
>#### Key detail: d1a149c is the cherry-pick target; origin/master is two commits behind HEAD

---

### Phase 5: Cherry-Pick onto Master

**Step 5.1** Switch to the `master` branch using `sudo` (required due to root ownership):

```bash
sudo git checkout master
```

Expected output:
```
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
```

**Step 5.2** Apply the target commit to `master`:

```bash
sudo git cherry-pick d1a149c
```

Expected output:
```
[master b8d9e80] Update info.txt
 Date: Fri Mar 6 05:29:03 2026 +0000
 1 file changed, 1 insertion(+), 1 deletion(-)
```

Note that a new commit hash (`b8d9e80`) is generated. This is expected behaviour: cherry-pick creates a new commit object on the target branch. The commit message and diff are preserved, but the parent hash changes.

### Screenshot: Successful Cherry-Pick Output

<img width="1038" height="347" alt="image" src="https://github.com/user-attachments/assets/6a97427e-44e5-4e1a-885f-2edd5f68422e" />

>#### Caption: sudo git cherry-pick d1a149c succeeding with new commit hash b8d9e80
>#### Key detail: '1 file changed, 1 insertion(+), 1 deletion(-)' confirms correct file was modified

---

### Phase 6: Verify the Commit on Master

**Step 6.1** Inspect the master branch log:

```bash
sudo git log --oneline
```

Expected output:
```
b8d9e80 (HEAD -> master) Update info.txt
34cf452 (origin/master) Add welcome.txt
3d326be initial commit
```

This confirms `Update info.txt` is now the latest commit on the local `master` branch and that `origin/master` is one commit behind, ready to receive the push.

### Screenshot: Git Log Confirming Cherry-Pick on Master

<img width="1038" height="399" alt="image" src="https://github.com/user-attachments/assets/3e3f990f-2617-4bf9-b674-774bb2bbad9d" />

### Caption: sudo git log --oneline showing b8d9e80 at HEAD on master
### Key detail: origin/master still at 34cf452, confirming the push has not happened yet

---

### Phase 7: Push to the Bare Remote

**Step 7.1** Push the updated `master` branch to `origin`:

```bash
sudo git push origin master
```

Expected output:
```
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 316 bytes | 316.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/official.git
   34cf452..b8d9e80  master -> master
```

The ref update line `34cf452..b8d9e80 master -> master` confirms the remote `master` has been fast-forwarded to include the cherry-picked commit.

### Screenshot: Push Output Confirming Remote Update

<img width="1032" height="432" alt="image" src="https://github.com/user-attachments/assets/01a3893e-7c51-40f9-a8a8-b7485022ea0e" />

>#### Caption: sudo git push origin master output showing ref update 34cf452..b8d9e80
>#### Key detail: 'To /opt/official.git' confirms push target is the correct bare remote

---

## Verification

Run the following to confirm end state:

```bash
sudo git log origin/master --oneline
```

Expected:
```
b8d9e80 (HEAD -> master, origin/master) Update info.txt
34cf452 Add welcome.txt
3d326be initial commit
```

Both `HEAD -> master` and `origin/master` pointing to `b8d9e80` confirms the local and remote branches are fully synchronised with the cherry-picked commit applied.

### Screenshot: Final Verification of Remote Master State

```
### [SCREENSHOT PLACEHOLDER]
### Caption: git log showing both master and origin/master pointing to b8d9e80
### This is the terminal success state for the task
```

---

## Lessons Learned

### 1. Always Verify the Actual Repository Path Before Running Git Commands

The specification said `/usr/src/kodekloudrepos` but the repo was at `/usr/src/kodekloudrepos/official`. Run `ls -la` on the documented path before assuming it is a Git root.

### 2. Git Ownership Checks Are a Security Feature, Not a Bug

The `safe.directory` block exists to prevent privilege escalation attacks in shared environments. The correct resolution is to register the path explicitly, not to disable the check globally or to change directory ownership without team approval.

### 3. `sudo git` Is Sometimes the Only Option for Root-Owned Repos

When a repository has been initialised or cloned as root in a shared server path, non-root users will always hit permission errors on any write operation. Using `sudo git` is the appropriate short-term solution. The long-term fix is to set correct ownership on the repo with `chown -R natasha:natasha /usr/src/kodekloudrepos/official`.

### 4. Cherry-Pick Creates a New Commit Hash by Design

The original commit `d1a149c` became `b8d9e80` on master. This is not an error. Cherry-pick reapplies the diff as a new commit whose parent is the tip of the target branch. The commit message and file changes are identical.

### 5. Only Cherry-Pick the Exact Required Commit

The `feature` branch had two commits ahead of `master`. Only `d1a149c` (`Update info.txt`) was required. Cherry-picking `062203a` (`Update welcome.txt`) would have polluted `master` with unreviewed feature work.

---

## Quick Reference Command Sheet

```bash
# 1. Connect to storage server
ssh natasha@ststor01
# Password: Bl@kW

# 2. Navigate to repository
cd /usr/src/kodekloudrepos/official

# 3. Resolve ownership block
git config --global --add safe.directory /usr/src/kodekloudrepos/official

# 4. Confirm current branch and view commits
git status
git log --oneline

# 5. Switch to master (requires sudo)
sudo git checkout master

# 6. Cherry-pick the target commit
sudo git cherry-pick d1a149c

# 7. Verify
sudo git log --oneline

# 8. Push to remote
sudo git push origin master
```

---


<img width="1034" height="441" alt="image" src="https://github.com/user-attachments/assets/af65ed23-82b7-43e4-a316-2d34cb0d29a9" />

<img width="1023" height="676" alt="image" src="https://github.com/user-attachments/assets/bd8b2587-38e2-48e1-a891-d989456cfd89" />
<img width="1034" height="617" alt="image" src="https://github.com/user-attachments/assets/6057fdc1-596e-4bbb-99a1-367b9adf1e6c" />

<img width="999" height="350" alt="image" src="https://github.com/user-attachments/assets/4e29e0e1-7b48-4541-b15c-008c40eb5b15" />
<img width="1032" height="646" alt="image" src="https://github.com/user-attachments/assets/725bd7d0-a0e8-4679-bbaa-ec80fc247849" />


<img width="1035" height="781" alt="image" src="https://github.com/user-attachments/assets/751b0def-fabb-4a42-8f9b-18bab06ed0e1" />




