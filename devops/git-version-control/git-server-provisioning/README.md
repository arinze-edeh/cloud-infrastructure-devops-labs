# Centralized Git Server Provisioning on a Shared Storage Node

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment Context](#environment-context)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1: Access the Jump Host](#phase-1-access-the-jump-host)
  - [Phase 2: Connect to the Storage Server](#phase-2-connect-to-the-storage-server)
  - [Phase 3: Install Git via yum](#phase-3-install-git-via-yum)
  - [Phase 4: Initialize the Bare Git Repository](#phase-4-initialize-the-bare-git-repository)
  - [Phase 5: Verify Repository Structure](#phase-5-verify-repository-structure)
- [Validation Checklist](#validation-checklist)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)

---

## Overview

This document details the end-to-end provisioning of a centralized Git server on the Nautilus Storage Node (`ststor01`) within the Stratos Datacenter. The implementation covers SSH-based access through a jump host, Git installation via the native package manager, and initialization of a bare repository to serve as a shared remote for distributed development teams.

---

## Problem Statement

Development teams operating across the Stratos Datacenter require a centralized, server-side Git repository for source code collaboration. The storage server `ststor01` is designated as the Git hosting node due to its network accessibility and persistent storage configuration. The objective is to provision Git on this node and initialize a bare repository at `/opt/demo.git` that is ready to accept remote push and pull operations without requiring a working tree.

---

## Environment Context

| Attribute | Value |
|---|---|
| Datacenter | Stratos DC |
| Access Pattern | Jump Host to Storage Server |
| Jump Host | `thor@jumphost` |
| Target Node | `ststor01` |
| Target User | `natasha` |
| Target IP | `172.16.238.15` |
| OS Family | CentOS Stream 9 |
| Package Manager | `yum` |
| Repository Type | Bare Git Repository |
| Repository Path | `/opt/demo.git` |
| Git Version Installed | `2.52.0-1.el9` |

---

## Prerequisites

- SSH access to the Jump Host as `thor`
- SSH access to `ststor01` as `natasha` (password-based)
- `sudo` privileges on `ststor01`
- Network reachability from Jump Host to `172.16.238.15`
- Internet access from `ststor01` to CentOS Stream 9 and EPEL repositories

---

## Implementation

### Phase 1: Access the Jump Host

All operations in this project are initiated from the designated jump host. This access pattern enforces network segmentation, ensuring that storage nodes are not directly exposed to external connections.

```bash
ssh thor@jumphost
```

**Screenshot: Authenticated session on the Jump Host**

![Jump Host Login](https://github.com/user-attachments/assets/10e208a6-fdb4-4157-9132-4799bbda5c47)

---

### Phase 2: Connect to the Storage Server

From the jump host, initiate an SSH session to `ststor01` using the `natasha` user account. On first connection, SSH presents the host's ED25519 key fingerprint for verification. Accept and persist the key to the `known_hosts` file to prevent repeated prompts on subsequent connections.

```bash
ssh natasha@172.16.238.15
```

**Expected behavior on first connection:**

- SSH displays the host key fingerprint and requests confirmation
- Respond `yes` to permanently add the host to `~/.ssh/known_hosts`
- Enter the password for `natasha` when prompted
- A successful login lands at the `[natasha@ststor01 ~]$` shell prompt

**Screenshot: SSH host key verification and successful authentication to ststor01**

![SSH Connection to ststor01](https://github.com/user-attachments/assets/10e208a6-fdb4-4157-9132-4799bbda5c47)

> **Security Note:** In production environments, password-based SSH authentication should be replaced with ED25519 key-pair authentication. Accepting unknown host fingerprints without out-of-band verification is a risk in sensitive environments. Always validate fingerprints against a trusted source before confirming.

---

### Phase 3: Install Git via yum

With an active session on `ststor01`, install Git using the `yum` package manager. The `-y` flag suppresses interactive confirmation prompts, enabling unattended installation suitable for automation pipelines.

```bash
sudo yum install -y git
```

`yum` performs the following sequence automatically:

1. Refreshes metadata from configured repositories (CentOS Stream 9 BaseOS, AppStream, Extras, EPEL)
2. Resolves the full dependency tree for `git`
3. Downloads and installs all required packages
4. Verifies package integrity against checksums

**Screenshot: yum resolving repositories and beginning dependency installation**

![yum Install - Dependency Resolution](https://github.com/user-attachments/assets/1f55ebae-8ec2-44ca-9876-e9b5285985d6)

**Screenshot: Git installation completed with all 67 packages verified**

![yum Install - Complete](https://github.com/user-attachments/assets/ca30e46d-72f4-42a6-81e5-66b041ba54a2)

**Packages installed include:**

- `git-2.52.0-1.el9.x86_64` (primary binary)
- `git-core` and `git-core-doc` (core libraries and documentation)
- `perl-*` runtime dependencies (Git uses Perl for several utilities)
- `less`, `groff-base`, and additional supporting packages

> **Operational Note:** The EPEL repository is enabled on this node. Git and its Perl dependency chain pull from both BaseOS and AppStream. Ensure EPEL is consistently configured across all nodes in environments where reproducible package states are required.

---

### Phase 4: Initialize the Bare Git Repository

Initialize an empty bare Git repository at `/opt/demo.git`. A **bare repository** contains only the Git object store and metadata; it has no working tree. This is the correct architecture for a shared, centralized remote that multiple developers push to and pull from.

```bash
sudo git init --bare /opt/demo.git
```

`sudo` is required because `/opt` is root-owned by default. The `--bare` flag instructs Git to omit the working tree and store all repository data directly in the target directory rather than in a `.git` subdirectory.

**Screenshot: Bare repository successfully initialized at /opt/demo.git**

![git init --bare](https://github.com/user-attachments/assets/a2b2bed4-13dc-4da4-ba3c-e8959aa4bce8)

> **Key Design Decision:** Bare repositories are the standard pattern for Git remotes. Unlike standard repositories, they do not maintain a checked-out branch, making them unsuitable for direct development but ideal as authoritative push targets. This prevents the common error of pushing to a non-bare repository with an active checked-out branch.

---

### Phase 5: Verify Repository Structure

Confirm that the bare repository was created correctly by listing its contents. A valid bare repository must contain a specific set of directories and files representing the Git internal object model.

```bash
ls -l /opt/demo.git
```

**Expected output:**

```
total 28
-rw-r--r-- 1 root root   23 Feb 27 01:46 HEAD
-rw-r--r-- 1 root root   66 Feb 27 01:46 config
-rw-r--r-- 1 root root   73 Feb 27 01:46 description
drwxr-xr-x 2 root root 4096 Feb 27 01:46 hooks
drwxr-xr-x 2 root root 4096 Feb 27 01:46 info
drwxr-xr-x 4 root root 4096 Feb 27 01:46 objects
drwxr-xr-x 4 root root 4096 Feb 27 01:46 refs
```

**Screenshot: Bare repository directory listing confirming correct structure**

![Repository Structure](https://github.com/user-attachments/assets/1c8b0329-7172-4862-a405-df1d08569f60)

**Component breakdown:**

| Entry | Purpose |
|---|---|
| `HEAD` | Points to the current default branch reference (initially `master`) |
| `config` | Repository-level Git configuration |
| `description` | Human-readable description used by GitWeb and other tools |
| `hooks/` | Directory for server-side hook scripts (pre-receive, post-receive, update) |
| `info/` | Contains the `exclude` file for repository-wide gitignore patterns |
| `objects/` | Git object store containing all commits, trees, and blobs |
| `refs/` | Reference store for branches, tags, and remotes |

---

## Validation Checklist

- [x] Jump Host SSH session established as `thor`
- [x] Storage Server SSH session established as `natasha` on `172.16.238.15`
- [x] Host key fingerprint accepted and persisted to `known_hosts`
- [x] Git `2.52.0-1.el9` installed successfully via `yum`
- [x] All 67 packages installed and verified by `yum`
- [x] Bare repository initialized at `/opt/demo.git`
- [x] Repository contains all required bare repository components: `HEAD`, `config`, `description`, `hooks/`, `info/`, `objects/`, `refs/`

---

## Key Decisions

**Bare repository over standard repository:** A bare repository is the only appropriate structure for a centralized remote. Pushing to a standard repository with an active checkout can corrupt the working tree and is rejected by Git in many configurations.

**`/opt` as the repository location:** The `/opt` path is conventionally used for optional, add-on software and shared infrastructure components in Red Hat-family distributions. It is accessible to all users with appropriate permissions and is outside user home directories, making it suitable for shared service resources.

**`sudo` for repository initialization:** Since `/opt` is root-owned, `sudo` is required for the `git init` operation. In production, consider using a dedicated `git` service account that owns `/opt/demo.git` to reduce the need for root-level access during routine operations.

**`yum` with `-y` flag:** Suppressing interactive prompts ensures the installation can be scripted or run non-interactively in automation contexts such as Ansible playbooks or CI/CD pipelines.

---

## Errors and Resolutions

**SSH host key warning on first connection**

- **Symptom:** `The authenticity of host '172.16.238.15' can't be established.`
- **Cause:** The host has not been previously connected to from this jump host, so no entry exists in `~/.ssh/known_hosts`.
- **Resolution:** Type `yes` at the prompt. SSH adds the fingerprint to `known_hosts` and does not prompt again on subsequent connections.
- **Production consideration:** Automate host key distribution using `ssh-keyscan` and pre-populate `known_hosts` during provisioning to avoid interactive prompts in automated workflows.

**Default branch name hint from Git**

- **Symptom:** Git outputs multiple `hint:` lines noting that `master` is the default branch name and that this will change to `main` in Git 3.0.
- **Cause:** This is an informational message, not an error. Git is configured to use `master` as the initial branch name by default on this installation.
- **Resolution:** No action required for immediate functionality. To suppress the hint globally and set a preferred default branch name, run:
  ```bash
  git config --global init.defaultBranch main
  ```

---

## Lessons Learned

**Bare repositories require explicit permissions planning.** Since the repository is owned by `root`, developers pushing to `git+ssh://natasha@172.16.238.15/opt/demo.git` must have write access to `/opt/demo.git`. In team environments, configure a shared `git` group, set group ownership on the repository directory, and enable the `setgid` bit to ensure new files inherit group ownership automatically.

**Package manager metadata freshness matters.** On long-running nodes, `yum` may use stale cached metadata. If package resolution fails or returns unexpected versions, run `sudo yum clean all && sudo yum makecache` before retrying installation.

**Bare repository hooks are the correct extension point.** Server-side hooks (`pre-receive`, `post-receive`, `update`) placed in `hooks/` enable enforcement of commit policies, integration with CI systems, and deployment automation without requiring client-side configuration changes.

**Default branch name consistency.** Teams should standardize the default branch name (typically `main`) across all repositories. Configure `init.defaultBranch` globally on the Git server before initializing repositories to ensure consistency and avoid branch naming discrepancies between the server and developer workstations.

---

## Outcome

A fully operational bare Git repository has been provisioned on the Nautilus Storage Node and is immediately ready to serve as a centralized remote for source control operations:

- **Remote URL pattern:** `ssh://natasha@172.16.238.15/opt/demo.git`
- **Supported operations:** `git clone`, `git push`, `git fetch`, `git pull`
- **Server-side hooks:** Available in `/opt/demo.git/hooks/` for policy enforcement and automation
- **Working tree:** None (by design for a bare repository)



