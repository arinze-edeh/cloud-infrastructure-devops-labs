# Git Branch Management on Remote Storage Server
> **Enterprise DevOps Runbook** | Stratos DC | Nautilus Project

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment](#environment)
- [Prerequisites](#prerequisites)
- [Resolution Workflow](#resolution-workflow)
  - [Phase 1: Remote Access](#phase-1-remote-access)
  - [Phase 2: Repository Initialization](#phase-2-repository-initialization)
  - [Phase 3: Branch Management](#phase-3-branch-management)
  - [Phase 4: Commit and Merge](#phase-4-commit-and-merge)
  - [Phase 5: Push to Origin](#phase-5-push-to-origin)
- [Issues Encountered and Fixes](#issues-encountered-and-fixes)
- [Verification](#verification)
- [Key Takeaways](#key-takeaways)
- [References](#references)

---

## Overview

This runbook documents the end-to-end process of managing a Git repository hosted on a remote storage server within the Stratos Datacenter. The task involved creating a feature branch, committing a file, merging back to the default branch, and pushing both branches to the remote origin, all performed through a jump host topology.

---

## Problem Statement

The Nautilus application development team required the following Git operations to be performed on the storage server (`ststor01`) against the repository cloned at `/usr/src/kodekloudrepos/blog`:

- Create a new branch named **`nautilus`** from **`master`**
- Copy `/tmp/index.html` (present on the storage server) into the repository
- Stage and commit the file on the `nautilus` branch
- Merge the `nautilus` branch back into `master`
- Push both `master` and `nautilus` branches to the remote origin at `/opt/blog.git`

---

## Environment

| Component | Details |
|---|---|
| Jump Host | `jump_host.stratos.xfusioncorp.com` |
| Jump Host User | `thor` |
| Storage Server | `ststor01` / `172.16.238.15` |
| Storage Server User | `natasha` |
| Repository Path | `/usr/src/kodekloudrepos/blog` |
| Remote Origin | `/opt/blog.git` |
| Source File | `/tmp/index.html` |
| Target Branch | `nautilus` |
| Base Branch | `master` |

---

## Prerequisites

- SSH access from the jump host to the storage server
- `sudo` privileges for user `natasha` on `ststor01`
- Git installed on the storage server
- Source file `/tmp/index.html` present on the storage server
- Remote origin `/opt/blog.git` reachable from the storage server

---

## Resolution Workflow

### Phase 1: Remote Access

SSH from the jump host into the storage server using the `natasha` credentials.

```bash
ssh natasha@172.16.238.15
```

Verify the correct host after login:

```bash
hostname
# Expected: ststor01.stratos.xfusioncorp.com
```

***Screenshot: Successful SSH login to ststor01 showing hostname confirmation***
<img width="1035" height="612" alt="image" src="https://github.com/user-attachments/assets/ca95d3d6-64c5-4858-be48-84c302fee568" />

---

### Phase 2: Repository Initialization

Navigate to the repository and resolve the Git safe directory ownership warning (see [Issues Encountered](#issues-encountered-and-fixes)).

```bash
cd /usr/src/kodekloudrepos/blog
pwd
git config --global --add safe.directory /usr/src/kodekloudrepos/blog
git status
git branch
```

**Expected output of `git status`:**
```
On branch master
Your branch is up to date with 'origin/master'.
nothing to commit, working tree clean
```

**Expected output of `git branch`:**
```
* master
```

***Screenshot: Terminal output showing git status and branch on master***
<img width="1031" height="614" alt="image" src="https://github.com/user-attachments/assets/c0013740-a589-4754-bb2e-12352920c1c4" />

Verify the source file is present before proceeding:

```bash
ls -la /tmp/index.html
# Expected: file owned by root, readable
```

***Screenshot Placeholder: ls output confirming /tmp/index.html exists***

---

### Phase 3: Branch Management

Because the `.git` directory is owned by `root`, all Git operations require `sudo`.

Create and switch to the `nautilus` branch:

```bash
sudo git checkout -b nautilus
```

**Expected output:**
```
Switched to a new branch 'nautilus'
```

***Screenshot Placeholder: sudo git checkout -b nautilus success output***

---

### Phase 4: Commit and Merge

**Copy the file into the repository:**

```bash
sudo cp /tmp/index.html /usr/src/kodekloudrepos/blog/
```

**Stage the file:**

```bash
sudo git add index.html
```

**Commit the file:**

```bash
sudo git commit -m "Add index.html to nautilus branch"
```

**Expected output:**
```
[nautilus cd0e1cc] Add index.html to nautilus branch
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
```

***Screenshot Placeholder: git commit success output on nautilus branch***

**Switch to master and merge:**

```bash
sudo git checkout master
sudo git merge nautilus
```

**Expected merge output:**
```
Updating 923d0e0..cd0e1cc
Fast-forward
 index.html | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 index.html
```

***Screenshot Placeholder: Fast-forward merge output from nautilus into master***

---

### Phase 5: Push to Origin

**Push master:**

```bash
sudo git push origin master
```

**Expected output:**
```
To /opt/blog.git
   923d0e0..cd0e1cc  master -> master
```

**Switch to nautilus and push:**

```bash
sudo git checkout nautilus
sudo git push origin nautilus
```

**Expected output:**
```
To /opt/blog.git
 * [new branch]      nautilus -> nautilus
```

***Screenshot Placeholder: Both git push commands showing successful push to /opt/blog.git***

---

## Issues Encountered and Fixes

### Issue 1: Git Dubious Ownership Warning

**Error:**
```
fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/blog'
```

**Root Cause:** The repository directory was owned by `root` while Git was being invoked as `natasha`, triggering Git's safe directory protection (introduced in Git 2.35.2).

**Fix:**
```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/blog
```

***Screenshot Placeholder: Terminal showing the dubious ownership error and the fix command***

---

### Issue 2: Permission Denied When Creating Branch

**Error:**
```
fatal: cannot lock ref 'refs/heads/nautilus': Unable to create
'/usr/src/kodekloudrepos/blog/.git/refs/heads/nautilus.lock': Permission denied
```

**Root Cause:** The `.git` directory and all its contents were owned by `root`. The `natasha` user lacked write permissions to create lock files required by Git for branch operations.

```bash
ls -la /usr/src/kodekloudrepos/blog/.git
# drwxr-xr-x 8 root root 4096 ...
```

**Fix:** Prefix all Git commands with `sudo` to execute them as root:

```bash
sudo git checkout -b nautilus
sudo git add index.html
sudo git commit -m "Add index.html to nautilus branch"
sudo git checkout master
sudo git merge nautilus
sudo git push origin master
sudo git push origin nautilus
```

***Screenshot Placeholder: Permission denied error followed by successful sudo git checkout***

---

## Verification

Run the following command to confirm both branches are in sync and pointing to the correct commit:

```bash
sudo git log --oneline --all --graph
```

**Expected output:**
```
* cd0e1cc (HEAD -> nautilus, origin/nautilus, origin/master, master) Add index.html to nautilus branch
* 923d0e0 initial commit
```

This confirms:
- Both `master` and `nautilus` local branches exist
- Both `origin/master` and `origin/nautilus` remote branches exist
- All four refs point to the same commit `cd0e1cc`
- The merge was a clean fast-forward with no divergence

***Screenshot Placeholder: git log --oneline --all --graph showing all four refs aligned***

---

## Key Takeaways

**1. Git Safe Directory Protection**
Git 2.35.2 and later enforce ownership checks. When a repository is owned by a different user than the one invoking Git, you must explicitly whitelist the directory using `git config --global --add safe.directory`.

**2. Root-Owned Repositories Require sudo**
When a repository's `.git` directory is owned by `root`, all write operations including branch creation, staging, committing, and pushing must be executed with `sudo`. Read operations such as `git status` and `git branch` may succeed without it depending on directory permissions.

**3. Fast-Forward Merges**
Since `master` had no new commits after `nautilus` branched off, the merge was a clean fast-forward. No merge commit was created, and the branch histories remained linear.

**4. Jump Host Topology**
All operations were performed from inside the storage server after SSH-ing through the jump host. No Git operations were performed from the jump host itself.

---

## References

- [Git Safe Directory Documentation](https://git-scm.com/docs/git-config#Documentation/git-config.txt-safedirectory)
- [Git CVE-2022-24765 Safe Directory Fix](https://github.blog/2022-04-12-git-security-vulnerability-announced/)
- [Git Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [KodeKloud Nautilus Project](https://kodekloud.com)

---

*Maintained by the Nautilus DevOps Team | Stratos Datacenter*



<img width="1036" height="715" alt="image" src="https://github.com/user-attachments/assets/ecd487bd-fae6-4381-9086-9c67784517ca" />

<img width="1026" height="674" alt="image" src="https://github.com/user-attachments/assets/70538549-314b-4873-89ce-19909e64ddb1" />
<img width="1034" height="711" alt="image" src="https://github.com/user-attachments/assets/16cf1c92-fec2-4e0b-8227-29b3214caa49" />
<img width="1034" height="435" alt="image" src="https://github.com/user-attachments/assets/da172961-3521-4965-ad59-be724914cf11" />
<img width="1038" height="423" alt="image" src="https://github.com/user-attachments/assets/3ceed183-b90e-4948-b788-c1fee7c9e5f3" />
<img width="1037" height="541" alt="image" src="https://github.com/user-attachments/assets/ad95570d-c7de-4a3c-a2e5-32b406cad342" />
<img width="1035" height="649" alt="image" src="https://github.com/user-attachments/assets/16d3fb9b-4fae-4273-8afa-d24eb2b95668" />
<img width="1035" height="649" alt="image" src="https://github.com/user-attachments/assets/6907c18c-e9aa-48be-8918-63533c188f2b" />
<img width="1044" height="428" alt="image" src="https://github.com/user-attachments/assets/f1584102-b598-46d7-bc44-f75a33d14e05" />
<img width="1036" height="321" alt="image" src="https://github.com/user-attachments/assets/cd794b51-7ff5-4acd-8ca4-03d1bf08d268" />
<img width="1029" height="374" alt="image" src="https://github.com/user-attachments/assets/3ae59fc3-ffe6-4f08-b74f-636886f4ac89" />
<img width="1038" height="580" alt="image" src="https://github.com/user-attachments/assets/b8aa7626-b989-435e-8784-3dfbf5a3e591" />
<img width="1029" height="647" alt="image" src="https://github.com/user-attachments/assets/d81e2fd6-449c-4277-b6c1-b462448b821b" />
<img width="1036" height="745" alt="image" src="https://github.com/user-attachments/assets/08083c1c-45c1-4d37-825c-0fbdf3078f1a" />

