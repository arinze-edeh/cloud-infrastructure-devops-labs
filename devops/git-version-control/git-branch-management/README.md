# Git Branch Management - Feature Branch Creation from Master

> **Domain:** DevOps | Git Version Control
> **Difficulty:** Intermediate
> **Environment:** Stratos DC — Nautilus Infrastructure
> **Status:** ✅ Resolved

---

## Table of Contents

- [Business Context](#business-context)
- [Infrastructure Overview](#infrastructure-overview)
- [Problem Statement](#problem-statement)
- [Root Cause Analysis](#root-cause-analysis)
- [Resolution](#resolution)
- [Verification](#verification)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Business Context

The Nautilus development team required a dedicated feature branch to isolate new application changes from the stable `master` codebase. As part of standard GitFlow practices, a new branch `xfusioncorp_apps` was provisioned from `master` in the centralized repository hosted on the Stratos DC Storage Server.

**Requirement Specification:**

| Parameter | Value |
|-----------|-------|
| Target Server | Storage Server (`ststor01`) |
| Repository Path | `/usr/src/kodekloudrepos/apps` |
| New Branch | `xfusioncorp_apps` |
| Source Branch | `master` |
| Code Changes Permitted | ❌ No |

---

## Infrastructure Overview

| Server | IP Address | User | Purpose |
|--------|-----------|------|---------|
| `jump_host` | Dynamic | `thor` | Stratos DC Entry Point |
| `ststor01` | `172.16.238.15` | `natasha` | Nautilus Storage Server (Target) |

---

## Problem Statement

Upon connecting to the Storage Server and attempting to create the feature branch, **two sequential blockers** were encountered that prevented standard branch creation.

---

### Problem 1 — Git Dubious Ownership Error

**Trigger:** Running `git branch` as user `natasha`

**Error Output:**
```
fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/apps'
To add an exception for this directory, call:
        git config --global --add safe.directory /usr/src/kodekloudrepos/apps
```

***Screenshot — Dubious Ownership Error***

![Dubious Ownership Error](_screenshots/01-dubious-ownership-error.png)
> *`git branch` fails immediately due to Git's CVE-2022-24765 ownership security check*

---

**Root Cause:**

Git introduced a security policy (CVE-2022-24765) where it refuses to operate on repositories owned by a different user than the one executing the command. The repository at `/usr/src/kodekloudrepos/apps` was owned by `root`, while `natasha` was the executing user.

```
drwxr-xr-x 8 root root 4096 Mar  2 21:01 .git   ← owned by root
                                                     executed by natasha ← mismatch ✗
```

---

### Problem 2 — Permission Denied on Branch Creation

**Trigger:** After resolving the ownership exception, running `git checkout -b xfusioncorp_apps`

**Error Output:**
```
fatal: cannot lock ref 'refs/heads/xfusioncorp_apps': Unable to create
'/usr/src/kodekloudrepos/apps/.git/refs/heads/xfusioncorp_apps.lock': Permission denied
```

***Screenshot — Permission Denied on Branch Creation***

![Permission Denied Error](_screenshots/02-permission-denied-branch.png)
> *Branch creation fails because natasha lacks write access to the root-owned `.git` directory*

**Root Cause:**

The entire repository including `.git/refs/heads/` is owned by `root` with `drwxr-xr-x` permissions. Git must write a `.lock` file inside `.git/refs/heads/` when creating a branch. User `natasha` has only read and execute permissions (`r-x`), not write (`w`). The operation is blocked at the filesystem level.

```
drwxr-xr-x  root root  .git/
                  ↑
                  natasha = r-x only (no write access)
                  git branch creation = requires write to .git/refs/heads/
                  Result: Permission Denied ✗
```

***Screenshot — Ownership Confirmation via ls -la***

![ls -la showing root ownership](_screenshots/03-ls-la-root-ownership.png)
> *`ls -la` confirms all repository files and `.git` directory are owned by `root:root`*

---

## Resolution

Since the repository is root-owned and `natasha` holds sudo privileges, the resolution requires privilege escalation to `root` to perform the branch operation safely.

---

### Step 1 — Connect to Jump Host

```bash
# Authenticated as thor on jump_host
thor@jumphost ~$
```

---

### Step 2 — SSH into the Storage Server

```bash
ssh natasha@172.16.238.15
# Accept host fingerprint on first connection: yes
# Enter password when prompted
```

***Screenshot — Successful SSH Connection to ststor01***

<img width="1033" height="534" alt="image" src="https://github.com/user-attachments/assets/cc7902ae-b03e-4362-9a39-3fb380d36e6d" />

> *Successful SSH from jump_host to ststor01; host fingerprint accepted and permanently added*

---

### Step 3 — Navigate to Repository and Inspect Ownership

```bash
cd /usr/src/kodekloudrepos/apps
pwd
ls -la
```

***Screenshot — Repository Directory Listing***

<img width="1032" height="658" alt="image" src="https://github.com/user-attachments/assets/c1f0b2e3-3813-45ae-af55-8dd1895995b5" />

> *Directory listing confirms `.git` and all files are owned by `root:root` — write blocked for natasha*

---

### Step 4 — Register Safe Directory Exception as natasha

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/apps
```

> Resolves Problem 1 for the `natasha` user context. Write permission remains blocked — escalation required.

---

### Step 5 — Escalate Privileges to Root

```bash
sudo su -
# Enter natasha's sudo password when prompted
```

***Screenshot — Privilege Escalation to Root***

![sudo su - escalation](_screenshots/06-sudo-escalation.png)
> *`sudo su -` grants full root shell; note the standard sudo security acknowledgement*

---

### Step 6 — Register Safe Directory Exception as Root

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/apps
```

> **Critical:** Git's `~/.gitconfig` is per-user. Root's global config at `/root/.gitconfig` requires its own safe directory registration independent of natasha's.

---

### Step 7 — Navigate to Repository and Checkout master

```bash
cd /usr/src/kodekloudrepos/apps
git checkout master
```

**Expected Output:**
```
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
```

***Screenshot — Checkout master Branch***

![git checkout master](_screenshots/07-checkout-master.png)
> *Explicitly checked out master before branching — ensures the new branch originates from the correct source*

---

### Step 8 — Create the Feature Branch

```bash
git checkout -b xfusioncorp_apps
```

**Expected Output:**
```
Switched to a new branch 'xfusioncorp_apps'
```

***Screenshot — Feature Branch Creation Success***

![git checkout -b xfusioncorp_apps](_screenshots/08-branch-creation-success.png)
> *`xfusioncorp_apps` created and checked out in a single command; no code changes made*

---

### Step 9 — Verify Branch Exists

```bash
git branch
```

**Expected Output:**
```
  kodekloud_apps
  master
* xfusioncorp_apps
```

***Screenshot — Branch Verification***

![git branch verification](_screenshots/09-branch-verification.png)
> *Active branch (`*`) confirms `xfusioncorp_apps` was successfully created from master*

---

### Step 10 — Exit Root Shell

```bash
exit
# Returns to natasha@ststor01
```

***Screenshot — Complete Terminal Session Overview***

![Full terminal session](_screenshots/10-full-session-overview.png)
> *End-to-end terminal session: SSH → ownership diagnosis → escalation → branch creation → verification → exit*

---

## Verification

| Check | Command | Expected Result | Status |
|-------|---------|----------------|--------|
| Correct server | `hostname` | `ststor01` | ✅ |
| Correct path | `pwd` | `/usr/src/kodekloudrepos/apps` | ✅ |
| Branched from master | `git checkout master` → `-b` | Clean switch confirmed | ✅ |
| Branch exists | `git branch` | `* xfusioncorp_apps` listed | ✅ |
| No code changes | `git status` | `nothing to commit` | ✅ |

---

## Lessons Learned

### 1. Git CVE-2022-24765 Safe Directory Policy
Git 2.35.2+ enforces ownership checks as a security measure against privilege escalation via malicious repositories. When a repo is owned by a different user, Git requires an explicit safe directory exception registered **per user** in their respective `~/.gitconfig`. Running as both `natasha` and `root` requires the exception to be set twice.

### 2. Root-Owned Repositories in Shared Environments
In enterprise lab and shared server environments, repositories are frequently initialized as `root`. Always verify ownership with `ls -la` **before** attempting git operations as a non-root user to anticipate permission failures early.

### 3. sudo su - vs sudo git
Using `sudo su -` (full root shell) is more reliable than `sudo git` in this scenario. A bare `sudo git` command would use root's binary but still reference natasha's environment, which may lack the safe directory exception in `/root/.gitconfig`.

### 4. Always Checkout Source Branch Explicitly
Before creating a feature branch, explicitly checkout the intended source (`git checkout master`). In this case, `kodekloud_apps` was the active branch — branching without switching would have created `xfusioncorp_apps` from the wrong base, violating the task requirement.

---

## Key Commands Reference

```bash
# Resolve dubious ownership (run as each user that needs access)
git config --global --add safe.directory /path/to/repo

# Escalate to root
sudo su -

# Explicit source branch checkout before branching
git checkout master

# Create and switch to new branch in one command
git checkout -b xfusioncorp_apps

# Confirm branch creation
git branch

# Exit root shell
exit
```

---

## References

- [Git CVE-2022-24765 Security Advisory — GitHub Blog](https://github.blog/2022-04-12-git-security-vulnerability-announced/)
- [Git Documentation — git-branch](https://git-scm.com/docs/git-branch)
- [Git Documentation — git-checkout](https://git-scm.com/docs/git-checkout)
- [Linux File Permissions Reference](https://man7.org/linux/man-pages/man1/chmod.1.html)

---

*Documented as part of the Nautilus DevOps Infrastructure Series — Stratos Datacenter Operations*




<img width="1035" height="642" alt="image" src="https://github.com/user-attachments/assets/3c479b20-e809-4b9a-a596-6addeb8a9496" />
<img width="1033" height="686" alt="image" src="https://github.com/user-attachments/assets/772cf1e0-3649-45d0-b6b4-af12df426fe2" />
<img width="1031" height="738" alt="image" src="https://github.com/user-attachments/assets/4b514d98-58e0-46ac-87d9-028aa9b35faf" />
<img width="1033" height="825" alt="image" src="https://github.com/user-attachments/assets/d492d10e-d7fc-4b7d-ac4e-942e8462ab70" />
<img width="1036" height="870" alt="image" src="https://github.com/user-attachments/assets/581e801d-0b76-48b6-ad3b-b1d30c9d58fc" />
<img width="1037" height="867" alt="image" src="https://github.com/user-attachments/assets/20085c4a-903b-48a7-80c9-feeab2410809" />
<img width="1034" height="559" alt="image" src="https://github.com/user-attachments/assets/1ba8873d-c34d-44bb-860c-4e556e493daa" />
<img width="1036" height="869" alt="image" src="https://github.com/user-attachments/assets/a4531482-ec4b-4650-860f-00c8e1a1f5b2" />


