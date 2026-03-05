# Git Repository HEAD Revert: Resolving Unauthorized Commit Pollution in Production

![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![DevOps](https://img.shields.io/badge/DevOps-0A66C2?style=for-the-badge&logo=azuredevops&logoColor=white)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=for-the-badge)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Environment Details](#environment-details)
- [Root Cause Analysis](#root-cause-analysis)
- [Prerequisites](#prerequisites)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Phase 1: Remote Access](#phase-1-remote-access-to-storage-server)
  - [Phase 2: Repository Navigation](#phase-2-navigate-to-the-git-repository)
  - [Phase 3: Safe Directory Fix](#phase-3-resolve-dubious-ownership-error)
  - [Phase 4: Permission Investigation](#phase-4-investigate-write-permission-failure)
  - [Phase 5: Privileged Revert](#phase-5-execute-revert-with-elevated-privileges)
  - [Phase 6: Verification](#phase-6-verify-the-revert)
- [Command Reference](#command-reference)
- [Error Index](#error-index)
- [Key Takeaways](#key-takeaways)

---

## Problem Statement

The Nautilus application development team identified that recent commits pushed to the shared git repository at `/usr/src/kodekloudrepos/games` on the **Stratos DC Storage Server** were erroneous and needed to be rolled back. The DevOps team was tasked with safely reverting the repository HEAD to the previous known-good commit, identified by the `initial commit` message, without destroying commit history.

The revert commit was required to carry the exact message `revert games` (all lowercase).

---

## Environment Details

| Component | Details |
|---|---|
| **Jump Host** | `jump_host.stratos.xfusioncorp.com` (user: `thor`) |
| **Storage Server** | `ststor01.stratos.xfusioncorp.com` / `172.16.238.15` |
| **Storage Server User** | `natasha` |
| **Repository Path** | `/usr/src/kodekloudrepos/games` |
| **Repository Owner** | `root` |
| **Target Branch** | `master` |
| **Bad Commit** | `5b11221` — "add data.txt file" (HEAD) |
| **Target Commit** | `71b573d` — "initial commit" |
| **Required Revert Message** | `revert games` |

---

## Root Cause Analysis

Two distinct blockers were encountered during execution:

**Blocker 1: Git Dubious Ownership**
Git version 2.35.2+ introduced a security check that flags repositories owned by a different user than the one running the git command. Since the repository was owned by `root` but accessed by `natasha`, git refused all operations until the directory was explicitly marked as a safe exception.

**Blocker 2: Write Permission Denied**
Even after the safe directory exception was registered for the `natasha` user context, the repository's `.git/` directory was owned by `root`, making it impossible for `natasha` to create the `index.lock` file required by git write operations. All mutating git commands (revert, commit) required `sudo` elevation.

---

## Prerequisites

- SSH access to the jump host (`thor@jump_host`)
- `sudo` privileges for `natasha` on `ststor01`
- Git installed on the storage server
- Network connectivity between jump host and `172.16.238.15`

---

## Resolution Walkthrough

### Phase 1: Remote Access to Storage Server

From the jump host, initiate an SSH session to the storage server using `natasha`'s credentials.

```bash
ssh natasha@172.16.238.15
```

When prompted about host authenticity (first-time connection), type `yes` to accept and permanently add the host fingerprint.

***Screenshot 1: Successful SSH login to ststor01***

<img width="1041" height="411" alt="image" src="https://github.com/user-attachments/assets/672ff97b-c168-4420-b805-29ea3464fb6a" />

---

### Phase 2: Navigate to the Git Repository

Change into the target repository directory and confirm the working path.

```bash
cd /usr/src/kodekloudrepos/games
pwd
```

Expected output:

```
/usr/src/kodekloudrepos/games
```

***Screenshot:***
<img width="1041" height="581" alt="image" src="https://github.com/user-attachments/assets/5d189d06-8d55-4efd-886b-e5f279a887de" />

---

### Phase 3: Resolve Dubious Ownership Error

Attempting `git log` at this stage will fail with a fatal ownership warning.

***Screenshot 2: git log fatal ownership error***

> **[Insert screenshot: terminal showing the `detected dubious ownership` fatal error message after running `git log --oneline`]**

Register the directory as a safe exception in `natasha`'s global git config to allow read operations:

```bash
git config --global --add safe.directory /usr/src/kodekloudrepos/games
```

Confirm the commit history is now readable:

```bash
git log --oneline
```

Expected output:

```
5b11221 (HEAD -> master, origin/master) add data.txt file
71b573d initial commit
```

***Screenshot 3: git log showing full commit history***

<img width="1036" height="560" alt="image" src="https://github.com/user-attachments/assets/2bba76cb-80ae-449c-a917-d014a0d2a702" />

This confirms:

- HEAD is at `5b11221` ("add data.txt file") which is the commit to revert
- The previous commit `71b573d` carries the message `initial commit` as expected

---

### Phase 4: Investigate Write Permission Failure

Attempting `git revert HEAD --no-commit` as `natasha` will fail with a permission error on `.git/index.lock`.

```bash
git revert HEAD --no-commit
# fatal: Unable to create '/usr/src/kodekloudrepos/games/.git/index.lock': Permission denied
```

Inspect directory ownership to confirm root ownership:

```bash
ls -la /usr/src/kodekloudrepos/
```

Expected output:

```
drwxr-xr-x 3 root root 4096 Mar  5 07:39 games
```

***Screenshot 4: ls -la confirming root ownership***

> **[Insert screenshot: terminal showing `ls -la` output with `root root` ownership on the games directory]**

All write operations must use `sudo`. The safe directory exception must also be registered in root's git config context separately.

---

### Phase 5: Execute Revert with Elevated Privileges

**Step 5a:** Register the safe directory for the `root` (sudo) context:

```bash
sudo git config --global --add safe.directory /usr/src/kodekloudrepos/games
```

Enter `natasha`'s sudo password when prompted.

**Step 5b:** Stage the revert without auto-committing, so a custom commit message can be supplied:

```bash
sudo git revert HEAD --no-commit
```

This stages the inverse of the HEAD commit changes without creating a commit yet.

**Step 5c:** Commit the staged revert with the exact required message (all lowercase):

```bash
sudo git commit -m "revert games"
```

Expected output:

```
[master 862cb51] revert games
 1 file changed, 1 insertion(+)
 create mode 100644 info.txt
```

***Screenshot 5: Successful git commit confirmation***

> **[Insert screenshot: terminal showing commit confirmation output with hash `862cb51` and message `revert games`]**

---

### Phase 6: Verify the Revert

Confirm the new commit is at HEAD with the correct message and that the full history is intact:

```bash
sudo git log --oneline
```

Expected output:

```
862cb51 (HEAD -> master) revert games
5b11221 (origin/master) add data.txt file
71b573d initial commit
```

***Screenshot 6: Final git log confirming successful revert***

> **[Insert screenshot: terminal showing the three-commit log with `revert games` at HEAD on master]**

**Verification checklist:**

- HEAD points to the new `revert games` commit
- The previous bad commit (`5b11221`) is preserved in history (non-destructive revert)
- `origin/master` still shows `5b11221`, confirming the revert has not yet been pushed upstream

---

## Command Reference

Complete sequence of commands executed to resolve this issue:

```bash
# Phase 1: Connect to storage server from jump host
ssh natasha@172.16.238.15

# Phase 2: Navigate to repository
cd /usr/src/kodekloudrepos/games
pwd

# Phase 3: Fix dubious ownership for natasha user context
git config --global --add safe.directory /usr/src/kodekloudrepos/games
git log --oneline

# Phase 4: Fix dubious ownership for root (sudo) context
sudo git config --global --add safe.directory /usr/src/kodekloudrepos/games

# Phase 5: Execute the revert with exact required commit message
sudo git revert HEAD --no-commit
sudo git commit -m "revert games"

# Phase 6: Verify final state
sudo git log --oneline
```

---

## Error Index

| Error | Cause | Resolution |
|---|---|---|
| `fatal: detected dubious ownership` | Git 2.35.2+ security check blocks cross-user repository access | Run `git config --global --add safe.directory <path>` as the active user |
| `fatal: Unable to create index.lock: Permission denied` | Repository owned by `root`; write operations blocked for `natasha` | Prefix all mutating git commands with `sudo` |
| `sudo git` still triggers ownership error | `sudo` runs git as `root`, which has its own separate global git config | Run `sudo git config --global --add safe.directory <path>` as a separate step before any sudo git operations |

---

## Key Takeaways

**Use `git revert` over `git reset` for shared repositories.** `git revert` creates a new commit that undoes changes, preserving the full commit history. `git reset` rewrites history and forces divergence on shared branches, which is destructive in team environments.

**The `--no-commit` flag is critical when a custom message is required.** Without it, git auto-generates a commit message from the reverted commit's subject line. Using `--no-commit` stages the changes without committing, allowing a precise message to be supplied via `git commit -m`.

**Cross-user git operations in secure environments require two-step sudo configuration.** The safe directory must be registered in both the active user's config AND in root's config (via `sudo git config`) when elevated privileges are needed for write access. These are two separate global config files.

**Always verify ownership before git operations on shared infrastructure.** A quick `ls -la` on the parent directory prevents wasted time debugging permission errors that are actually ownership issues in disguise.

---



<img width="1034" height="542" alt="image" src="https://github.com/user-attachments/assets/05d63cff-b0be-455e-a500-69d33cb4d49c" />

<img width="1034" height="658" alt="image" src="https://github.com/user-attachments/assets/a3087417-136e-4b46-b525-3a357512fbe5" />
<img width="1033" height="790" alt="image" src="https://github.com/user-attachments/assets/04c2d457-4663-488a-80b6-2dd84a06786f" />
<img width="1036" height="781" alt="image" src="https://github.com/user-attachments/assets/5a8362c1-d758-479f-815e-fb1ced0e9185" />
<img width="1035" height="838" alt="image" src="https://github.com/user-attachments/assets/b62e6645-3462-4752-b215-b3d1982604fb" />
<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/2c4a87b1-3919-459a-9c2c-81cc20c2f8ee" />
