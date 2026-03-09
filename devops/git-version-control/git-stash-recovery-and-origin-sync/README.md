# Git Stash Recovery and Remote Push on Restricted Storage Server

![Task Status](https://img.shields.io/badge/Status-Resolved-brightgreen)
![Environment](https://img.shields.io/badge/Environment-Stratos%20DC-blue)
![Server](https://img.shields.io/badge/Server-ststor01-orange)
![Git](https://img.shields.io/badge/Git-Stash%20Recovery-red)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure](#infrastructure)
- [Root Cause Analysis](#root-cause-analysis)
- [Resolution Steps](#resolution-steps)
- [Screenshots](#screenshots)
- [Commands Reference](#commands-reference)
- [Lessons Learned](#lessons-learned)
- [Prevention and Best Practices](#prevention-and-best-practices)
- [Contributing](#contributing)

---

## Overview

This document details the identification and resolution of a git stash recovery issue encountered on the Nautilus application development infrastructure within Stratos DC. A developer had previously stashed in-progress changes in the `/usr/src/kodekloudrepos/demo` repository on the storage server (`ststor01`). The objective was to restore a specific stash entry (`stash@{1}`), commit the recovered changes, and push them to the remote origin.

---

## Problem Statement

### Context

The Nautilus application development team maintains a git repository at `/usr/src/kodekloudrepos/demo` on the `ststor01` (Nautilus Storage Server). A developer stashed work-in-progress changes and the team required restoration of the stash identified by `stash@{1}`.

### Symptoms Observed

| Symptom | Error Message | Severity |
|---|---|---|
| Dubious ownership detection | `fatal: detected dubious ownership in repository` | High |
| Index lock permission failure | `error: Unable to create '.git/index.lock': Permission denied` | High |
| Stash inaccessible without privilege escalation | `error: could not write index` | Medium |

### Objective

1. List available stash entries in `/usr/src/kodekloudrepos/demo`
2. Apply `stash@{1}` to the working tree
3. Stage, commit, and push the restored changes to `origin/master`

---

## Infrastructure

### Target Environment

| Property | Value |
|---|---|
| Datacenter | Stratos DC |
| Repository Path | `/usr/src/kodekloudrepos/demo` |
| Remote Origin | `/opt/demo.git` |
| Target Branch | `master` |
| Access Method | SSH via Jump Host |

### Server Details

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| `jump_host` | `jump_host.stratos.xfusioncorp.com` | `thor` | Jump Server (Stratos DC entry point) |
| `ststor01` | `ststor01.stratos.xfusioncorp.com` | `natasha` | Nautilus Storage Server (target) |

---

## Root Cause Analysis

Two distinct issues were identified during this resolution:

### Issue 1: Git Safe Directory Violation

**Root Cause:** The repository at `/usr/src/kodekloudrepos/demo` was owned by a different user (likely `root` or another system user) than the logged-in user (`natasha`). Git 2.35.2+ introduced a security check that rejects repository operations when the directory owner does not match the current user, raising a `dubious ownership` fatal error.

**Impact:** Prevented all git operations including `git stash list`.

### Issue 2: Insufficient Write Permissions on `.git/index`

**Root Cause:** Even after resolving the safe directory issue, the `.git` directory had restrictive permissions that blocked `natasha` from creating the `.git/index.lock` file required by git during index write operations.

**Impact:** Prevented `git stash apply` from completing without elevated privileges (`sudo`).

---

## Resolution Steps

### Step 1: Connect to Jump Host

Initiate SSH session from your local machine to the Stratos DC jump host.

```bash
ssh thor@jump_host.stratos.xfusioncorp.com
```

> **Note:** Use the credentials for user `thor` on `jump_host`.

---

### Step 2: SSH into the Storage Server

From the jump host, connect to the Nautilus Storage Server.

```bash
ssh natasha@ststor01
```

Accept the ED25519 host key fingerprint on first connection when prompted.

```
The authenticity of host 'ststor01 (10.244.234.207)' can't be established.
ED25519 key fingerprint is SHA256:yEyN8qvzhNxfcKVE+H05zwQPmQMKCXj4JyGWuOP1HIg.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

**Screenshot**

<img width="1036" height="482" alt="image" src="https://github.com/user-attachments/assets/98726301-adcc-4fa1-911f-c029f7b3ccc3" />

---

### Step 3: Navigate to the Repository

```bash
cd /usr/src/kodekloudrepos/demo
```

---

### Step 4: Resolve Git Safe Directory Error

Attempting `git stash list` initially fails with a fatal ownership error.

```bash
git stash list
# fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/demo'
```

Register the repository as a safe directory in the global git configuration:

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/demo
```

**Screenshot**

<img width="1031" height="470" alt="image" src="https://github.com/user-attachments/assets/b57c7447-dd95-4db1-b33e-612502dd4225" />

Verify stash entries are now visible:

```bash
git stash list
```

Expected output:

```
stash@{0}: WIP on master: 40efb4b initial commit
stash@{1}: WIP on master: 40efb4b initial commit
```

**Screenshot**

<img width="1033" height="528" alt="image" src="https://github.com/user-attachments/assets/c40ab99f-6375-465a-8b27-dcd89de21bd1" />

---

### Step 5: Apply the Target Stash Entry

Apply `stash@{1}` using `sudo` to overcome the `.git/index` permission restriction:

```bash
sudo git stash apply stash@{1}
```

Expected output confirms the stash was applied and a new file is staged:

```
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   welcome.txt
```

**Screenshot**

> **### Screenshot 4: sudo git stash apply stash@{1} output showing welcome.txt staged ###**

---

### Step 6: Stage All Changes

```bash
sudo git add .
```

---

### Step 7: Commit the Restored Changes

```bash
sudo git commit -m "Restore stashed changes from stash@{1}"
```

Expected output:

```
[master fd170ad] Restore stashed changes from stash@{1}
 1 file changed, 1 insertion(+)
 create mode 100644 welcome.txt
```

**Screenshot Placeholder**

> ***** Screenshot 5: git commit output confirming welcome.txt committed on master *****

---

### Step 8: Push to Remote Origin

```bash
sudo git push origin master
```

Expected output:

```
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 323 bytes | 323.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/demo.git
   40efb4b..fd170ad  master -> master
```

**Screenshot Placeholder**

> ****** Screenshot 6: Successful git push origin master with commit hash 40efb4b to fd170ad ******

---

### Step 9: Exit the Storage Server

```bash
exit
```

```
logout
Connection to ststor01 closed.
```

---

## Screenshots

| # | Description | Location |
|---|---|---|
| 1 | SSH connection from jump_host to ststor01 | `[Insert Screenshot]` |
| 2 | git stash list error and safe.directory resolution | `[Insert Screenshot]` |
| 3 | git stash list output with stash@{0} and stash@{1} | `[Insert Screenshot]` |
| 4 | sudo git stash apply stash@{1} showing welcome.txt staged | `[Insert Screenshot]` |
| 5 | git commit confirming welcome.txt on master | `[Insert Screenshot]` |
| 6 | git push origin master with ref update | `[Insert Screenshot]` |

---

## Commands Reference

Complete ordered command sequence for reproducibility:

```bash
# From local machine
ssh thor@jump_host.stratos.xfusioncorp.com

# From jump_host
ssh natasha@ststor01

# From ststor01
cd /usr/src/kodekloudrepos/demo
git stash list
git config --global --add safe.directory /usr/src/kodekloudrepos/demo
git stash list
sudo git stash apply stash@{1}
sudo git add .
sudo git commit -m "Restore stashed changes from stash@{1}"
sudo git push origin master
exit
```

---

## Lessons Learned

### 1. Git Ownership Security Model (CVE-2022-24765)

Git 2.35.2 introduced `safe.directory` enforcement to prevent privilege escalation attacks via git hooks in shared directories. On servers where repositories are owned by `root` or a service account but accessed by regular users, this check must be explicitly bypassed using `git config --global --add safe.directory`.

### 2. Shared Repository Permission Management

When multiple users or processes interact with a single git repository, write permissions on `.git/` contents (particularly `index` and `index.lock`) must align with the operational workflow. Repositories managed by root on shared servers will require `sudo` for all write operations.

### 3. Stash Index Awareness

Git stash indices are zero-based and ordered newest-first. `stash@{0}` is always the most recent stash. Confirm the correct stash identifier via `git stash list` and, when in doubt, inspect contents with `git stash show stash@{N}` before applying.

---

## Prevention and Best Practices

### Repository Ownership Alignment

Ensure the repository directory owner matches the user who will perform git operations:

```bash
sudo chown -R natasha:natasha /usr/src/kodekloudrepos/demo
```

This eliminates the need for both the `safe.directory` workaround and `sudo` escalation for routine git operations.

### Stash Hygiene

Document stash entries with descriptive messages to avoid ambiguity:

```bash
git stash push -m "Feature: add welcome page content"
```

This makes `git stash list` output self-documenting and reduces the risk of applying the wrong stash.

### Audit Trail

Always use descriptive commit messages when restoring stashed work to maintain a clear audit trail:

```bash
git commit -m "Restore: welcome.txt from stash@{1} - WIP initial commit"
```

---

## Contributing

For issues related to this runbook or the Nautilus Storage infrastructure, raise a ticket via the internal engineering portal or contact the Stratos DC platform team.

When submitting improvements to this document:

1. Fork the repository
2. Create a feature branch: `git checkout -b docs/update-stash-runbook`
3. Commit changes: `git commit -m "Docs: improve stash recovery runbook"`
4. Push and open a pull request

---

*Maintained by the Nautilus Platform Engineering Team | Stratos DC*


<img width="1032" height="412" alt="image" src="https://github.com/user-attachments/assets/206a8a36-1672-4f3b-81d8-e4af865ee217" />
<img width="1032" height="446" alt="image" src="https://github.com/user-attachments/assets/76db174f-4844-4fa6-84b8-960327d83ec3" />


<img width="1037" height="588" alt="image" src="https://github.com/user-attachments/assets/87f21c9d-db43-4825-a964-cb7d64115a2c" />
<img width="1035" height="641" alt="image" src="https://github.com/user-attachments/assets/a068ca46-53cc-451f-8897-39dcaef12d98" />
<img width="1037" height="414" alt="image" src="https://github.com/user-attachments/assets/b42dae69-d9fa-47a8-a891-cf6eb08edc11" />
<img width="1038" height="477" alt="image" src="https://github.com/user-attachments/assets/a2700f81-d14c-46f9-8a14-187aace1e975" />
<img width="1026" height="499" alt="image" src="https://github.com/user-attachments/assets/2cdf26f2-a72f-460b-8510-991dc5593162" />



