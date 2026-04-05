# Git Repository Provisioning on Nautilus Storage Server

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Summary](#solution-summary)
- [Environment Details](#environment-details)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: SSH into the Storage Server](#step-1-ssh-into-the-storage-server)
  - [Step 2: Verify the Source Bare Repository](#step-2-verify-the-source-bare-repository)
  - [Step 3: Verify the Target Clone Directory](#step-3-verify-the-target-clone-directory)
  - [Step 4: Navigate to the Target Directory](#step-4-navigate-to-the-target-directory)
  - [Step 5: Clone the Bare Repository](#step-5-clone-the-bare-repository)
  - [Step 6: Verify Clone Integrity and Remote Configuration](#step-6-verify-clone-integrity-and-remote-configuration)
- [Validation](#validation)
- [Key Decisions](#key-decisions)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Lessons Learned](#lessons-learned)

---

## Overview

This document details the provisioning of a Git working clone from a pre-existing bare repository (`/opt/apps.git`) residing on the Nautilus Storage Server (`ststor01`) within the Stratos Datacenter. The operation was executed under the `natasha` user account and required no modifications to the source repository or any existing directory structure.

---

## Problem Statement

The Nautilus team maintained a bare Git repository at `/opt/apps.git` on the storage server `ststor01.stratos.xfusioncorp.com`. While the repository structure was intact, no working clone had been provisioned for developer use. Without an active clone linked to the bare repository, the development team lacked a workspace from which to stage, commit, and push application source code.

**Root Cause:** A bare repository stores Git objects and references but provides no working tree. Developers cannot work directly inside a bare repo. A clone was required to establish a usable working directory linked to the bare repo as its remote `origin`.

---

## Solution Summary

Clone the bare repository at `/opt/apps.git` into the designated workspace at `/usr/src/kodekloudrepos` on the storage server, using `git clone` with a local path reference. This provisions a fully initialized working directory (`apps/`) with `origin` automatically set to `/opt/apps.git`, enabling the development team to begin committing and pushing immediately.

---

## Environment Details

| Parameter | Value |
|---|---|
| **Server** | `ststor01.stratos.xfusioncorp.com` |
| **Server IP** | `172.16.238.15` |
| **Operating User** | `natasha` |
| **Source Bare Repository** | `/opt/apps.git` |
| **Clone Target Directory** | `/usr/src/kodekloudrepos` |
| **Resulting Clone Path** | `/usr/src/kodekloudrepos/apps` |
| **Remote Origin** | `/opt/apps.git` |

---

## Prerequisites

- SSH access to `ststor01.stratos.xfusioncorp.com` with the `natasha` user credentials
- Git installed and available in the system `PATH` on `ststor01`
- The bare repository at `/opt/apps.git` must exist and be readable by `natasha`
- The target directory `/usr/src/kodekloudrepos` must exist and be writable by `natasha`
- No pre-existing `apps/` directory inside `/usr/src/kodekloudrepos` (prevents clone conflicts)

---

## Implementation

### Step 1: SSH into the Storage Server

Initiate an SSH session from the jump host (`jumphost`) into the storage server as user `natasha`. On first connection, the server's ED25519 host key is presented for fingerprint verification. Accept and proceed; the key is then permanently added to the local `known_hosts` file for all subsequent connections.

```bash
ssh natasha@ststor01.stratos.xfusioncorp.com
# Enter password when prompted
```

**Screenshot: Successful SSH authentication into ststor01 as natasha**

![SSH into Storage Server](https://github.com/user-attachments/assets/ef3ca39b-89aa-4681-8e42-a1e6300a8b32)

> **Operational Note:** The `Warning: Permanently added ... to the list of known hosts` message is expected on first connection. On subsequent logins, the fingerprint is matched silently. If this warning persists on repeat connections, investigate potential host key rotation or man-in-the-middle conditions.

---

### Step 2: Verify the Source Bare Repository

Before cloning, confirm that the source bare repository exists, is owned by `natasha`, and contains the expected Git internal objects. A bare repository does not have a working tree; its top-level contents are the raw Git object store.

```bash
ls -ld /opt/apps.git
ls /opt/apps.git
```

**Expected output:**

```
drwxr-xr-x 7 natasha natasha 4096 Feb 28 02:55 /opt/apps.git
HEAD  branches  config  description  hooks  info  objects  refs
```

**Screenshot: Source bare repository structure confirmed at /opt/apps.git**

![Verify Source Bare Repository](https://github.com/user-attachments/assets/107bd6af-454a-4155-b318-cdbf03e1d97e)

> **Key Indicators:** The presence of `HEAD`, `objects/`, and `refs/` confirms this is a valid Git bare repository. The `config` file will contain the `[core] bare = true` flag. The `natasha` ownership confirms the operating user has the required read access.

---

### Step 3: Verify the Target Clone Directory

Confirm that the designated clone workspace exists and is accessible. Do not create or modify this directory. Its prior existence was confirmed by the task specification.

```bash
ls -ld /usr/src/kodekloudrepos
```

**Expected output:**

```
drwxr-xr-x 2 natasha natasha 4096 Feb 28 02:55 /usr/src/kodekloudrepos
```

**Screenshot: Target directory /usr/src/kodekloudrepos confirmed as pre-existing**

![Verify Target Directory](https://github.com/user-attachments/assets/9a6f97cd-b7cd-4507-a1d4-ecf9059f5fe0)

> **Operational Constraint:** The task specification explicitly prohibits creating or modifying directories outside the repository. The directory already exists, owned by `natasha`, satisfying both the access requirement and the no-modification constraint.

---

### Step 4: Navigate to the Target Directory

Change the working directory to `/usr/src/kodekloudrepos` and confirm the current path before executing the clone. This prevents the clone from landing in an unintended location.

```bash
cd /usr/src/kodekloudrepos
pwd
```

**Expected output:**

```
/usr/src/kodekloudrepos
```

**Screenshot: Working directory confirmed as /usr/src/kodekloudrepos prior to clone**

![Navigate to Target Directory](https://github.com/user-attachments/assets/916c568a-c7e2-4a96-9c9f-8b3165d33278)

> **Best Practice:** Always validate the current working directory with `pwd` immediately before running `git clone`. This eliminates the risk of inadvertently cloning into the home directory or an incorrect path.

---

### Step 5: Clone the Bare Repository

Execute `git clone` using the local filesystem path to the bare repository. Git treats the bare repository as a remote origin automatically, eliminating the need for manual remote configuration.

```bash
git clone /opt/apps.git
```

**Expected output:**

```
Cloning into 'apps'...
warning: You appear to have cloned an empty repository.
done.
```

**Screenshot: Git clone completed; working directory apps/ created under /usr/src/kodekloudrepos**

![Clone the Bare Repository](https://github.com/user-attachments/assets/1d7ab386-bb1f-445b-9fc2-ba4d606d66db)

> **Expected Warning:** The message `You appear to have cloned an empty repository` is expected and non-critical. It indicates the bare repository contains no commits yet. Git still initializes the local clone correctly, setting up the remote `origin` and branch tracking configuration. The development team can now add files, commit, and push to populate the repository.

---

### Step 6: Verify Clone Integrity and Remote Configuration

Navigate into the cloned working directory and run `git status` and `git remote -v` to confirm the clone is properly initialized and that `origin` points correctly to the bare repository.

```bash
ls -l
cd apps
git status
git remote -v
```

**Expected output:**

```
# ls -l
total 4
drwxr-xr-x 3 natasha natasha 4096 Feb 28 03:02 apps

# git status
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)

# git remote -v
origin  /opt/apps.git (fetch)
origin  /opt/apps.git (push)
```

**Screenshot: Clone verified; git status shows master branch, git remote -v confirms origin at /opt/apps.git**

![Verify Clone Integrity](https://github.com/user-attachments/assets/e68d956a-7867-47fe-9f7e-c9fbb2aefd1c)

> **Verification Checklist:**
> - `On branch master` confirms the default branch is initialized
> - `No commits yet` is consistent with cloning from an empty bare repository
> - `origin /opt/apps.git (fetch)` and `origin /opt/apps.git (push)` confirm the remote is correctly set to the source bare repository
> - No manual `git remote add` was required; `git clone` handled this automatically

---

## Validation

The following conditions confirm successful and complete implementation:

- `ststor01` is accessible via SSH as `natasha` from the jump host
- `/opt/apps.git` exists with a valid bare repository structure (`HEAD`, `objects/`, `refs/`, `config`)
- `/usr/src/kodekloudrepos` existed prior to the operation and was not created or altered
- The working clone is located at `/usr/src/kodekloudrepos/apps`
- `git status` inside the clone reports `On branch master` and `No commits yet`
- `git remote -v` confirms `origin` is set to `/opt/apps.git` for both fetch and push
- No files were modified, deleted, or created outside the repository clone directory

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Used local path (`/opt/apps.git`) for `git clone` | Direct filesystem clone avoids network protocol overhead and eliminates the need for SSH key setup or daemon configuration for local-only operations |
| Did not use `git init` or `git remote add` manually | `git clone` handles remote origin configuration automatically, reducing the risk of misconfiguration |
| Verified source repository structure before cloning | Ensures the bare repository is valid and avoids ambiguous clone failures mid-operation |
| Confirmed working directory with `pwd` before cloning | Eliminates the risk of cloning into an unintended path |
| Did not initialize any files in the clone | Task scope explicitly required no modifications; the clone was provisioned in a clean, empty state for the development team to populate |

---

## Risks and Edge Cases

**Empty Repository Warning**
The `warning: You appear to have cloned an empty repository` output is benign in this context. If this warning appears unexpectedly when cloning a repository that should contain commits, investigate whether the correct bare repository path was targeted or whether the repository's `HEAD` reference is pointing to a branch with no commits.

**Permissions on /opt/apps.git**
If `natasha` does not have read permissions on `/opt/apps.git`, the clone will fail with `fatal: repository '/opt/apps.git' does not exist` or a permission denied error. Verify ownership and ACLs before executing.

**Pre-existing apps/ Directory**
If an `apps/` directory already exists at `/usr/src/kodekloudrepos/apps`, `git clone` will fail with `destination path 'apps' already exists and is not an empty directory`. Inspect and remove or rename the existing directory before reattempting.

**Branch Name Differences**
In newer Git versions, the default branch may initialize as `main` rather than `master`, depending on the `init.defaultBranch` configuration of the Git installation. Verify the branch name if downstream CI/CD pipelines assume a specific default branch name.

**Host Key Verification**
The first SSH login required accepting the ED25519 host key fingerprint. In automated or non-interactive environments, use `ssh-keyscan` to pre-populate `known_hosts` or configure `StrictHostKeyChecking=accept-new` to handle this without manual intervention.

---

## Lessons Learned

- **Bare repositories are not working directories.** A bare repo stores the Git object database and is suitable as a centralized remote, but developers must clone from it to get a working tree. Understanding this distinction is critical when setting up Git-based source control infrastructure.
- **Local path clones are valid Git remotes.** Git's transport layer supports filesystem paths as remote URLs. This is efficient for same-host operations and avoids unnecessary network stack overhead.
- **Always verify source before operating.** Confirming the bare repository's internal structure (`ls /opt/apps.git`) before cloning catches corruption or incorrect paths early, before a mid-operation failure.
- **`git clone` is idempotent in intent but not in execution.** If the operation fails partway through, a partial `apps/` directory may be left behind. Always check for residual state before retrying.
- **Confirming `git remote -v` post-clone is a mandatory validation step.** It provides definitive proof that the remote origin is correctly configured, which is the single most important attribute of a functional Git clone.
