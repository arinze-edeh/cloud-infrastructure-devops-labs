# Git Linear Rebase Workflow: Feature Branch Synchronization Without Merge Commits



![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![SSH](https://img.shields.io/badge/SSH-4D4D4D?style=for-the-badge&logo=openssh&logoColor=white)
![DevOps](https://img.shields.io/badge/DevOps-0A66C2?style=for-the-badge&logo=azuredevops&logoColor=white)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure](#infrastructure)
- [Prerequisites](#prerequisites)
- [Solution Walkthrough](#solution-walkthrough)
- [Screenshots](#screenshots)
- [Key Concepts](#key-concepts)
- [Outcome](#outcome)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This document details the resolution of a Git branch synchronization task carried out on the **Nautilus Application Development** infrastructure in **Stratos DC**. The objective was to rebase a `feature` branch on top of the `master` branch in a bare remote repository, preserving all feature branch commits while avoiding merge commits, then force-pushing the rebased history to the remote.

---

## Problem Statement

### Context

The Nautilus application development team maintains a Git repository at `/opt/news.git` (bare remote) on the **Stratos Storage Server** (`ststor01`). The working clone resides at `/usr/src/kodekloudrepos/news`.

### Requirements

| Requirement | Detail |
|---|---|
| Branch to update | `feature` |
| Base branch | `master` |
| Strategy | Rebase (not merge) |
| Constraint | No merge commits to be introduced |
| Final step | Push rebased `feature` branch to remote |

### The Challenge

A developer working on the `feature` branch had their branch diverge from `master` after new commits were pushed to `master`. The developer required the `feature` branch to be updated to reflect the latest `master` commits while retaining all work done on `feature`, and critically, **without introducing a merge commit** that a standard `git merge` would produce.

---

## Infrastructure

### Environment Details

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| ststor01 | ststor01.stratos.xfusioncorp.com | natasha | Nautilus Storage Server |
| jump_host | jump_host.stratos.xfusioncorp.com | thor | Jump Server (Stratos DC entry point) |

### Repository Layout

```
Remote (bare):   /opt/news.git                          (on ststor01)
Working clone:   /usr/src/kodekloudrepos/news            (on ststor01)
```

---

## Prerequisites

- SSH access to the jump host (`thor@jumphost`)
- Valid credentials for `natasha` on `ststor01`
- Git installed on the storage server
- Write permissions to the repository working directory

---

## Solution Walkthrough

### Step 1: Connect to the Storage Server via Jump Host

From the jump host, establish an SSH session to the storage server using the `natasha` account.

```bash
thor@jumphost ~$ ssh natasha@ststor01.stratos.xfusioncorp.com
```

Accept the host key fingerprint on first connection, then authenticate with the password.

**Verify identity and hostname after login:**

```bash
[natasha@ststor01 ~]$ whoami && hostname
natasha
ststor01
```

> **Screenshot**
> ### SSH Login and Identity Verification

<img width="1027" height="518" alt="image" src="https://github.com/user-attachments/assets/ca81d002-d5f9-40e0-939d-49faa02f36d8" />

---

### Step 2: Fix Directory Ownership

The working clone directory must be owned by `natasha` to allow Git operations without privilege errors.

```bash
[natasha@ststor01 ~]$ sudo chown -R natasha:natasha /usr/src/kodekloudrepos/news
```

---

### Step 3: Configure Git Safe Directories

Git requires directories to be explicitly marked safe when they are not owned by the invoking user or when running under `sudo`. Register both the working clone and the bare remote as safe:

```bash
[natasha@ststor01 ~]$ git config --global --add safe.directory /usr/src/kodekloudrepos/news
[natasha@ststor01 ~]$ git config --global --add safe.directory /opt/news.git
```

---

### Step 4: Set Git User Identity

Configure the global Git identity for commit metadata and push attribution:

```bash
[natasha@ststor01 ~]$ git config --global user.email "natasha@ststor01.stratos.xfusioncorp.com"
[natasha@ststor01 ~]$ git config --global user.name "natasha"
```

---

### Step 5: Inspect Repository State

Navigate to the working clone and verify the current branch and commit graph before making any changes:

```bash
[natasha@ststor01 ~]$ cd /usr/src/kodekloudrepos/news
[natasha@ststor01 news]$ git status
On branch feature
nothing to commit, working tree clean

[natasha@ststor01 news]$ git log --oneline --all --graph
* 7d71c67 (HEAD -> feature, origin/feature) Add new feature
| * 5d54451 (origin/master, master) Update info.txt
|/
* fa7cbd6 initial commit
```

**Observed Divergence:**

```
            fa7cbd6  (initial commit)
           /        \
  feature: 7d71c67   master: 5d54451
```

The `feature` branch had diverged from `master` after `5d54451` was committed to `master`. A standard `git merge` would add an unwanted merge commit at the tip of `feature`.

> **Screenshot**
> ### Pre-Rebase Git Graph
<img width="1032" height="690" alt="image" src="https://github.com/user-attachments/assets/8c97b774-d02e-420d-8a60-4cf47c086324" />

---

### Step 6: Perform the Rebase

Rebase the `feature` branch on top of `master`. This replays the `feature` commits on top of the latest `master` commit, producing a linear history with no merge commit:

```bash
[natasha@ststor01 news]$ git rebase master
Successfully rebased and updated refs/heads/feature.
```

---

### Step 7: Verify the Rebased History

Confirm the linear commit graph after the rebase:

```bash
[natasha@ststor01 news]$ git log --oneline --all --graph
* 1dd1d90 (HEAD -> feature) Add new feature
* 5d54451 (origin/master, master) Update info.txt
| * 7d71c67 (origin/feature) Add new feature
|/
* fa7cbd6 initial commit
```

**Post-Rebase State:**

```
fa7cbd6 -> 5d54451 (master) -> 1dd1d90 (feature HEAD)
```

The local `feature` branch now sits cleanly on top of `master`. The old remote `feature` tip (`7d71c67`) still exists on the remote and will be overwritten in the next step.

> **Screenshot Placeholder**
> ### Post-Rebase Git Graph
> *(Insert screenshot of `git log --oneline --all --graph` confirming linear history after rebase)*

---

### Step 8: Force Push to Remote

Because the rebase rewrites commit history (new SHA `1dd1d90` replaces `7d71c67`), a regular `git push` will be rejected. Use `--force-with-lease` to safely overwrite the remote `feature` branch. This flag is safer than `--force` as it verifies no one else has pushed to the remote since your last fetch:

```bash
[natasha@ststor01 news]$ sudo git push origin feature --force-with-lease
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 16 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 328 bytes | 328.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/news.git
 + 7d71c67...1dd1d90 feature -> feature (forced update)
```

The `+` prefix in the push output confirms a forced update was applied. The remote `feature` branch now points to the rebased commit `1dd1d90`.

> **Screenshot Placeholder**
> ### Successful Force Push Output
> *(Insert screenshot of the `git push --force-with-lease` terminal output)*

---

## Screenshots

> ### Full Terminal Session Overview
> *(Insert full-session screenshot showing all commands executed from SSH login through push)*

---

## Key Concepts

### `git rebase` vs `git merge`

| Aspect | `git merge` | `git rebase` |
|---|---|---|
| History shape | Non-linear (merge commit added) | Linear (no merge commit) |
| Commit SHAs | Preserved | Rewritten |
| Remote push | Standard push | Requires force push |
| Use case | Preserving exact branch history | Clean, linear history |

### `--force-with-lease` vs `--force`

| Flag | Behavior |
|---|---|
| `--force` | Unconditionally overwrites the remote branch |
| `--force-with-lease` | Fails if the remote branch has been updated by another push since your last fetch, preventing accidental data loss |

### Why Ownership and Safe Directory Matter

Git 2.35.2+ introduced ownership checks to prevent privilege escalation attacks. When the working directory owner differs from the invoking user, Git refuses to operate unless the path is explicitly added to `safe.directory`. This is a common friction point in shared or containerized DevOps environments.

---

## Outcome

| Checkpoint | Result |
|---|---|
| SSH access established | Confirmed |
| Working directory ownership fixed | Confirmed |
| Safe directory configuration applied | Confirmed |
| Pre-rebase divergence verified | Confirmed |
| Rebase completed without merge commit | Confirmed |
| Linear history verified post-rebase | Confirmed |
| Remote `feature` branch updated via force push | Confirmed |

The `feature` branch was successfully rebased onto `master` and pushed to the remote bare repository at `/opt/news.git`. No merge commits were introduced. All original feature branch changes were preserved under the new commit SHA `1dd1d90`.

---

## Lessons Learned

**Always use `--force-with-lease` over `--force` when pushing rebased branches.** It provides a safety net against accidentally overwriting commits pushed by teammates between your fetch and push.

**Perform a `git log --all --graph` before and after a rebase.** Visual confirmation of the commit topology prevents surprises and makes the rebase result immediately verifiable.

**Set `safe.directory` proactively in shared environments.** In environments where multiple users or automation accounts interact with the same repository, pre-configuring safe directories in global Git config avoids unexpected permission failures.

**Coordinate rebase operations with the team.** Because rebasing rewrites SHAs, any collaborator with a local copy of the old `feature` branch will need to reset to the new remote tip after a force push.

---

## References

- [Git Rebase Documentation](https://git-scm.com/docs/git-rebase)
- [Git Push --force-with-lease](https://git-scm.com/docs/git-push#Documentation/git-push.txt---force-with-leaseltrefnamegt)
- [Git Safe Directory Configuration](https://git-scm.com/docs/git-config#Documentation/git-config.txt-safedirectory)
- [Atlassian: Merging vs Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)

---

Environment: Stratos DC, Nautilus Storage Server | Scope: Git Branch Management*

<img width="1034" height="407" alt="image" src="https://github.com/user-attachments/assets/1d3dfb68-982e-4224-9291-56d92ff45ddf" />

<img width="1035" height="612" alt="image" src="https://github.com/user-attachments/assets/2d496932-22bb-4533-b960-1491716a777f" />
<img width="1031" height="575" alt="image" src="https://github.com/user-attachments/assets/d78db3f6-3d63-4919-bcd5-1596871b327e" />

<img width="1032" height="728" alt="image" src="https://github.com/user-attachments/assets/bca0a5b5-0fad-45a3-8e2b-dcb6399190c4" />
<img width="1033" height="826" alt="image" src="https://github.com/user-attachments/assets/7382bd91-b49e-4d9a-8d78-f4c898cff25b" />
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/5b7e40c1-e9c7-4dd9-aa55-1b36fa200d4f" />
