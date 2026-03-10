# Git Linear Rebase Workflow: Feature Branch Synchronization Without Merge Commits

![Git](https://img.shields.io/badge/Git-2.39%2B-F05032?style=for-the-badge&logo=git&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-CentOS-262577?style=for-the-badge&logo=centos&logoColor=white)
![Status](https://img.shields.io/badge/Status-Resolved-28a745?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-git--version--control-blue?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment](#environment)
- [Root Cause Analysis](#root-cause-analysis)
- [Prerequisites](#prerequisites)
- [Resolution](#resolution)
- [Verification](#verification)
- [Outcome](#outcome)
- [Key Takeaways](#key-takeaways)
- [Repository Structure](#repository-structure)

---

## Overview

This document details the end-to-end resolution of a Git branch synchronization task in a production-adjacent Stratos DC environment. A developer's `feature` branch had diverged from `master` due to new commits pushed upstream. The requirement was to rebase the `feature` branch onto `master` to produce a clean, linear commit history without introducing merge commits, and push the result to the remote repository.

***Screenshot: Task description panel showing repository path and rebase requirements***
> `![Task Overview](screenshots/01-task-overview.png)`

---

## Problem Statement

### Context

The Nautilus application development team maintains a project repository at `/opt/news.git` on the Stratos DC Storage Server. The working clone resides at `/usr/src/kodekloudrepos/news`. A developer was actively working on the `feature` branch when new commits were merged into `master`, causing the two branches to diverge from a common ancestor.

### Requirement

| Requirement | Detail |
|---|---|
| Operation | Rebase `feature` branch onto `master` |
| Constraint | No merge commits permitted |
| Constraint | No data loss from `feature` branch |
| Final Step | Push rebased branch to remote |

### Branch State Before Resolution

```
* 7d71c67 (HEAD -> feature, origin/feature)  Add new feature
| * 5d54451 (origin/master, master)           Update info.txt
|/
* fa7cbd6                                     initial commit
```

The diverged graph above confirms `feature` and `master` split from `fa7cbd6`. A standard `git merge` at this point would produce an unwanted merge commit, violating the team requirement.

***Screenshot: Git log graph showing diverged branch state before rebase***
> `![Pre-Rebase Graph](screenshots/02-pre-rebase-graph.png)`

---

## Environment

| Component | Value |
|---|---|
| Jump Host | `jump_host.stratos.xfusioncorp.com` |
| Storage Server | `ststor01.stratos.xfusioncorp.com` |
| Storage Server IP | `10.244.234.219` |
| Operating User | `natasha` |
| Working Repository | `/usr/src/kodekloudrepos/news` |
| Bare Remote Repository | `/opt/news.git` |
| Active Branch | `feature` |
| Rebase Target | `master` |

---

## Root Cause Analysis

Three blockers existed in the environment prior to executing the rebase. Each had to be resolved in sequence before Git operations could succeed.

### Blocker 1: Repository Owned by Root

The working repository at `/usr/src/kodekloudrepos/news` was owned by `root`. The operating user `natasha` lacked write access to the `.git` directory, causing the rebase to fail with:

```
error: could not create temporary .git/rebase-merge: Permission denied
```

**Resolution:** Transfer ownership of the working repository to `natasha`. The bare remote `/opt/news.git` must remain under `root` ownership to preserve remote integrity for the checker and other users.

### Blocker 2: Git Safe Directory Restriction

Git 2.35.2+ enforces ownership checks and refuses to operate on repositories owned by a different user. Even after the `chown` fix, Git blocked access to both paths with:

```
fatal: detected dubious ownership in repository
```

**Resolution:** Register both paths as trusted via `safe.directory` in the global Git config.

### Blocker 3: Missing Committer Identity

Git requires a committer identity to rewrite commits during a rebase. With no `user.email` or `user.name` configured in the environment, the rebase failed with:

```
fatal: unable to auto-detect email address
```

**Resolution:** Set global Git identity before executing the rebase.

---

## Prerequisites

* SSH access to `jump_host.stratos.xfusioncorp.com` as `thor`
* SSH access to `ststor01.stratos.xfusioncorp.com` as `natasha`
* `sudo` privileges for `natasha` on the storage server
* Git 2.35.2 or higher installed on the storage server

---

## Resolution

### Step 1: Access the Storage Server via Jump Host

Connect from the jump host to the storage server:

```bash
ssh natasha@ststor01.stratos.xfusioncorp.com
```

Verify identity and hostname immediately:

```bash
whoami && hostname
# Expected output:
# natasha
# ststor01
```

***Screenshot: Successful SSH connection and identity verification***
> `![SSH Connection](screenshots/03-ssh-connection.png)`

---

### Step 2: Fix Repository Ownership

Transfer ownership of the **working repository only** to `natasha`. The bare remote `/opt/news.git` must not be touched.

```bash
sudo chown -R natasha:natasha /usr/src/kodekloudrepos/news
```

> **Critical:** Never run `chown` on `/opt/news.git`. That bare repository is owned by `root` intentionally. Changing its ownership corrupts access for the validation checker and other system processes.

---

### Step 3: Register Safe Directories

Register both the working repository and bare remote as trusted in the global Git configuration:

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/news
git config --global --add safe.directory /opt/news.git
```

---

### Step 4: Configure Git Identity

Set the committer identity required for rebase commit rewriting:

```bash
git config --global user.email "natasha@ststor01.stratos.xfusioncorp.com"
git config --global user.name "natasha"
```

---

### Step 5: Navigate to Repository and Verify State

```bash
cd /usr/src/kodekloudrepos/news
```

Confirm clean working tree on the `feature` branch:

```bash
git status
# On branch feature
# nothing to commit, working tree clean
```

Inspect the full branch graph:

```bash
git log --oneline --all --graph
```

Confirm active branch:

```bash
git branch
# * feature
#   master
```

***Screenshot: git status, git log graph, and git branch output confirming clean state***
> `![Pre-Rebase Verification](screenshots/04-pre-rebase-verification.png)`

---

### Step 6: Execute the Rebase

Rebase the `feature` branch onto `master`:

```bash
git rebase master
```

Expected output:

```
Successfully rebased and updated refs/heads/feature.
```

This command replays all commits from `feature` on top of the latest commit in `master`, producing a linear history with new SHA identifiers. No merge commit is created.

***Screenshot: git rebase master showing successful output***
> `![Rebase Success](screenshots/05-rebase-success.png)`

---

### Step 7: Verify Linear History

Confirm the commit graph is now a straight line with no forks:

```bash
git log --oneline --all --graph
```

Expected output:

```
* 1dd1d90 (HEAD -> feature)          Add new feature
* 5d54451 (origin/master, master)    Update info.txt
| * 7d71c67 (origin/feature)         Add new feature
|/
* fa7cbd6                            initial commit
```

The local `feature` branch (`1dd1d90`) now sits directly on top of `master` (`5d54451`). The `origin/feature` reference still points to the old SHA until the push is completed.

***Screenshot: git log graph showing linear history after rebase***
> `![Post-Rebase Graph](screenshots/06-post-rebase-graph.png)`

---

### Step 8: Push the Rebased Branch

Because rebase rewrites commit history (new SHA hashes), a force push is required. `--force-with-lease` is used instead of `--force` as it is the safer option. It will abort if another user has pushed to the branch since the last fetch. `sudo` is required because Git internally accesses the bare repository at `/opt/news.git` which is owned by `root`.

```bash
sudo git push origin feature --force-with-lease
```

Expected output:

```
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 328 bytes | 328.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/news.git
 + 7d71c67...1dd1d90 feature -> feature (forced update)
```

***Screenshot: sudo git push output confirming forced update to /opt/news.git***
> `![Push Success](screenshots/07-push-success.png)`

---

## Verification

The following table maps each requirement to its verified outcome:

| Requirement | Verification Command | Result |
|---|---|---|
| Feature rebased onto master | `git log --oneline --all --graph` | `1dd1d90` sits on top of `5d54451` |
| No merge commit introduced | `git log --oneline --all --graph` | Straight line, no fork node |
| No feature data lost | `git log --oneline feature` | "Add new feature" commit preserved |
| Remote updated | Push output line `feature -> feature` | Forced update confirmed |

---

## Outcome

### Branch State After Resolution

```
* 1dd1d90 (HEAD -> feature)          Add new feature   <-- rebased
* 5d54451 (origin/master, master)    Update info.txt   <-- master tip
* fa7cbd6                            initial commit
```

The `feature` branch now has a perfectly linear history. All feature work is preserved. The remote at `/opt/news.git` reflects the updated state. No merge commits exist anywhere in the history.

***Screenshot: Final terminal session showing complete flow from SSH to push***
> `![Final State](screenshots/08-final-complete-session.png)`

---

## Key Takeaways

### Do This

| Action | Reason |
|---|---|
| `sudo chown -R natasha:natasha /usr/src/kodekloudrepos/news` | Fix only the working repo, not the bare remote |
| `git config --global --add safe.directory` for both paths | Satisfies Git 2.35.2+ ownership security check |
| `git rebase master` from `feature` branch | Replays commits linearly, no merge commit created |
| `sudo git push origin feature --force-with-lease` | `sudo` required for bare repo access; lease flag prevents overwriting unknown remote changes |

### Never Do This

| Action | Consequence |
|---|---|
| `sudo chown -R natasha:natasha /opt/news.git` | Breaks bare repo permissions, fails validation checker |
| `git merge master` from `feature` | Creates an unwanted merge commit, violates the linear history requirement |
| `git push --force` without `--lease` | Unsafe, can overwrite concurrent remote pushes silently |

---

<img width="1034" height="407" alt="image" src="https://github.com/user-attachments/assets/1d3dfb68-982e-4224-9291-56d92ff45ddf" />
<img width="1027" height="518" alt="image" src="https://github.com/user-attachments/assets/ca81d002-d5f9-40e0-939d-49faa02f36d8" />
<img width="1035" height="612" alt="image" src="https://github.com/user-attachments/assets/2d496932-22bb-4533-b960-1491716a777f" />
<img width="1031" height="575" alt="image" src="https://github.com/user-attachments/assets/d78db3f6-3d63-4919-bcd5-1596871b327e" />
<img width="1032" height="690" alt="image" src="https://github.com/user-attachments/assets/8c97b774-d02e-420d-8a60-4cf47c086324" />
<img width="1032" height="728" alt="image" src="https://github.com/user-attachments/assets/bca0a5b5-0fad-45a3-8e2b-dcb6399190c4" />
<img width="1033" height="826" alt="image" src="https://github.com/user-attachments/assets/7382bd91-b49e-4d9a-8d78-f4c898cff25b" />
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/5b7e40c1-e9c7-4dd9-aa55-1b36fa200d4f" />
