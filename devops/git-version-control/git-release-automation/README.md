# Automated Release Governance via Native Git Hook Architecture in Enterprise Bare Repository Environments

> **Enterprise-style Git Hook Pipeline** | Automated release tagging on push to `master` via `post-update` hook on a bare remote repository hosted on a dedicated Storage Server.

---

## Table of Contents

- [Overview](#overview)
- [Infrastructure](#infrastructure)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1: Server Access](#phase-1-server-access)
  - [Phase 2: Repository Verification](#phase-2-repository-verification)
  - [Phase 3: Branch Merge](#phase-3-branch-merge)
  - [Phase 4: Hook Creation](#phase-4-hook-creation)
  - [Phase 5: Hook Testing](#phase-5-hook-testing)
  - [Phase 6: Push and Finalize](#phase-6-push-and-finalize)
- [Verification](#verification)
- [Hook Reference](#hook-reference)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)

---

## Overview

This repository documents the end-to-end setup of an automated Git release tagging system for the **Nautilus Application Development Team** operating within the **Stratos DC** infrastructure. The solution leverages a native Git `post-update` hook to automatically generate a date-stamped release tag every time changes are pushed to the `master` branch, eliminating manual tagging overhead and enforcing consistent release naming conventions across the team.

| Property | Value |
|---|---|
| **Hook Type** | `post-update` |
| **Trigger** | Push to `master` branch |
| **Tag Format** | `release-YYYY-MM-DD` |
| **Remote Repository** | `/opt/ecommerce.git` |
| **Working Clone** | `/usr/src/kodekloudrepos/ecommerce` |
| **Execution User** | `natasha` |

---

## Infrastructure

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| `jump_host` | `jump_host.stratos.xfusioncorp.com` | `thor` | Jump Server / Entry Point |
| `ststor01` | `ststor01.stratos.xfusioncorp.com` | `natasha` | Nautilus Storage Server |

> **Note:** All operations are performed as user `natasha` on `ststor01`. Do not escalate privileges or alter existing directory permissions.

---

## Problem Statement

### Context

The Nautilus application team maintains a Git repository (`/opt/ecommerce.git`) cloned under `/usr/src/kodekloudrepos` on the Storage Server. Development work is carried out on the `feature` branch and must be merged into `master` through a controlled pipeline.

### Requirements

1. Merge the `feature` branch into `master` before pushing.
2. Implement a `post-update` hook that auto-generates a release tag in the format `release-YYYY-MM-DD` on every push to `master`.
3. Test the hook at least once and confirm tag creation.
4. Push all changes and tags to the remote.
5. Preserve all existing repository and directory permissions.

### Root Cause of Manual Overhead

Without automation, release tagging requires a developer to manually run `git tag` after every merge and push cycle. This is error-prone, inconsistent, and does not scale across multiple contributors or deployment cycles.

---

## Architecture

```
thor@jumphost
      |
      |  SSH
      v
natasha@ststor01
      |
      |  cd /usr/src/kodekloudrepos/ecommerce
      v
 [Working Clone]
      |
      |  git push origin master
      v
 /opt/ecommerce.git  (bare remote)
      |
      |  triggers
      v
 .git/hooks/post-update
      |
      |  git tag release-YYYY-MM-DD
      v
 Release Tag Created and Pushed
```

---

## Prerequisites

- SSH access from `jump_host` to `ststor01` as user `natasha`
- Git installed on `ststor01`
- Repository already cloned at `/usr/src/kodekloudrepos/ecommerce`
- Both `feature` and `master` branches exist in the repository
- Write access to `.git/hooks/` directory

---

## Implementation

### Phase 1: Server Access

SSH into the Storage Server from the Jump Host using the `natasha` user account.

```bash
ssh natasha@ststor01.stratos.xfusioncorp.com
```

On first connection, accept the host fingerprint when prompted:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

**Screenshot: SSH Connection to Storage Server**

<img width="1029" height="394" alt="image" src="https://github.com/user-attachments/assets/05573b00-ba6d-43b2-95a2-de195e482ae7" />

> *Successful SSH login as `natasha` into `ststor01.stratos.xfusioncorp.com`*

---

### Phase 2: Repository Verification

Navigate to the repository base directory and confirm the `ecommerce` clone exists, then enter the repository.

```bash
cd /usr/src/kodekloudrepos
ls -la
```

**Expected output:**

```
drwxr-xr-x 3 natasha natasha 4096 Mar 12 02:12 ecommerce
```

> **Important:** Running `git status` in `/usr/src/kodekloudrepos` will fail with `fatal: not a git repository`. Always `cd` into the `ecommerce` subdirectory first.

```bash
cd ecommerce
git status
git branch
```

**Expected output:**

```
On branch feature
nothing to commit, working tree clean

* feature
  master
```

**Screenshot: Repository Verification**

<img width="1029" height="579" alt="image" src="https://github.com/user-attachments/assets/f1f2b3af-ca13-4319-ab71-8fbf12c67257" />

> *Confirming `ecommerce` repo exists with both `feature` and `master` branches*

---

### Phase 3: Branch Merge

Switch to `master` and merge the `feature` branch using a fast-forward merge.

```bash
git checkout master
git merge feature
```

**Expected output:**

```
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
Updating 38db7a4..5edef8d
Fast-forward
 feature.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 feature.txt
```

Confirm the merge commit log:

```bash
git log --oneline -5
```

**Expected output:**

```
5edef8d (HEAD -> master, origin/feature, feature) Add feature
38db7a4 (origin/master) initial commit
```

**Screenshot: Branch Merge**

<img width="1027" height="760" alt="image" src="https://github.com/user-attachments/assets/90c9f4e9-2849-4888-b92a-0e36071650ac" />

> *Fast-forward merge of `feature` into `master` confirmed via `git log`*

---

### Phase 4: Hook Creation

Navigate into the hooks directory and create the `post-update` hook script.

```bash
cd .git/hooks
```

Create the hook file using a heredoc to preserve exact formatting:

```bash
cat > post-update << 'EOF'
#!/bin/bash
branch=$(git rev-parse --abbrev-ref HEAD)
if [ "$branch" = "master" ]; then
    DATE=$(date +%Y-%m-%d)
    TAG="release-${DATE}"
    git tag "$TAG"
    echo "Release tag created: $TAG"
fi
EOF
```

Make the hook executable:

```bash
chmod +x post-update
```

Verify content and permissions:

```bash
cat post-update
ls -la post-update
```

**Expected permission output:**

```
-rwxr-xr-x 1 natasha natasha 202 Mar 12 02:32 post-update
```

**Screenshot: Hook Creation and Permissions**

<img width="1035" height="840" alt="image" src="https://github.com/user-attachments/assets/2ba14881-2d15-49ff-a99c-57593a297e62" />

> *`post-update` hook written, verified, and marked executable with `-rwxr-xr-x`*

---

### Phase 5: Hook Testing

Return to the repository root and manually trigger the hook to validate it executes correctly before relying on it in the push pipeline.

```bash
cd /usr/src/kodekloudrepos/ecommerce
.git/hooks/post-update
```

**Expected output:**

```
Release tag created: release-2026-03-12
```

Confirm the tag exists in the local repository:

```bash
git tag
```

**Expected output:**

```
release-2026-03-12
```

**Screenshot: Hook Test and Tag Verification**

<img width="1036" height="819" alt="image" src="https://github.com/user-attachments/assets/f7bd5a82-3aa1-4e18-aa9d-68de76364f77" />

> *Hook fires successfully and `release-2026-03-12` tag confirmed via `git tag`*

---

### Phase 6: Push and Finalize

Push the merged `master` branch to the remote, then push all tags.

```bash
git push origin master
git push origin --tags
```

**Expected output:**

```
To /opt/ecommerce.git
   38db7a4..5edef8d  master -> master

To /opt/ecommerce.git
 * [new tag]         release-2026-03-12 -> release-2026-03-12
```

Run a final log and tag check to confirm the complete state:

```bash
git log --oneline -5
git tag
```

**Expected final log output:**

```
5edef8d (HEAD -> master, tag: release-2026-03-12, origin/master, origin/feature, feature) Add feature
38db7a4 initial commit
```

**Screenshot: Final Push and Verification**

<img width="1034" height="422" alt="image" src="https://github.com/user-attachments/assets/124c32e2-a3cb-41f3-a497-e8b296170a91" />

> *`master` and `release-2026-03-12` tag successfully pushed to `/opt/ecommerce.git`*

---

## Verification

The task is fully complete when all of the following conditions are confirmed:

| Checkpoint | Command | Expected Result |
|---|---|---|
| On `master` branch | `git branch` | `* master` |
| Feature merged | `git log --oneline` | `HEAD -> master` includes feature commit |
| Hook exists | `ls -la .git/hooks/post-update` | `-rwxr-xr-x` |
| Hook fires correctly | `.git/hooks/post-update` | `Release tag created: release-YYYY-MM-DD` |
| Tag created locally | `git tag` | `release-YYYY-MM-DD` listed |
| Master pushed | `git push origin master` | `master -> master` confirmed |
| Tag pushed to remote | `git push origin --tags` | `[new tag]` confirmed |
| Final log state | `git log --oneline -5` | `tag: release-YYYY-MM-DD` visible on HEAD |

---

## Hook Reference

### post-update Hook Script

```bash
#!/bin/bash

# post-update hook
# Triggered automatically after a successful push to the remote repository.
# Creates a date-stamped release tag when changes land on the master branch.

branch=$(git rev-parse --abbrev-ref HEAD)

if [ "$branch" = "master" ]; then
    DATE=$(date +%Y-%m-%d)
    TAG="release-${DATE}"
    git tag "$TAG"
    echo "Release tag created: $TAG"
fi
```

### Tag Naming Convention

| Component | Value | Example |
|---|---|---|
| Prefix | `release-` | `release-` |
| Date Format | `YYYY-MM-DD` | `2026-03-12` |
| Full Tag | `release-YYYY-MM-DD` | `release-2026-03-12` |

### Hook Location

```
/usr/src/kodekloudrepos/ecommerce/.git/hooks/post-update
```

---

## Troubleshooting

### `fatal: not a git repository`

**Cause:** Running `git` commands from `/usr/src/kodekloudrepos` instead of the `ecommerce` subdirectory.

**Resolution:**
```bash
cd /usr/src/kodekloudrepos/ecommerce
git status
```

---

### Hook does not fire on push

**Cause:** Hook file is not executable.

**Resolution:**
```bash
chmod +x /usr/src/kodekloudrepos/ecommerce/.git/hooks/post-update
ls -la /usr/src/kodekloudrepos/ecommerce/.git/hooks/post-update
# Must show: -rwxr-xr-x
```

---

### Tag already exists error

**Cause:** Hook was already triggered today and the tag `release-YYYY-MM-DD` already exists locally.

**Resolution:**
```bash
git tag -d release-2026-03-12
.git/hooks/post-update
```

---

### Push rejected

**Cause:** Local branch is behind the remote.

**Resolution:**
```bash
git pull origin master
git push origin master
```

---

## Security Considerations

- All operations are scoped to user `natasha` with no privilege escalation required.
- Directory permissions on `/usr/src/kodekloudrepos` and its contents are not modified beyond adding execute permission to the hook file.
- The hook script does not accept external input and operates only on the current branch ref, mitigating injection risk.
- SSH host key fingerprint is verified and permanently added to known hosts on first connection.

---
