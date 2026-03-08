# Git Commit History Reset and Force Push on Remote Storage Server

![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Shell](https://img.shields.io/badge/Shell_Script-121011?style=for-the-badge&logo=gnu-bash&logoColor=white)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment and Infrastructure](#environment-and-infrastructure)
- [Prerequisites](#prerequisites)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Phase 1: Access the Storage Server](#phase-1-access-the-storage-server)
  - [Phase 2: Navigate to the Repository](#phase-2-navigate-to-the-repository)
  - [Phase 3: Resolve Safe Directory Error](#phase-3-resolve-safe-directory-error)
  - [Phase 4: Identify the Target Commit](#phase-4-identify-the-target-commit)
  - [Phase 5: Escalate to Root](#phase-5-escalate-to-root)
  - [Phase 6: Reset Commit History](#phase-6-reset-commit-history)
  - [Phase 7: Force Push to Remote](#phase-7-force-push-to-remote)
  - [Phase 8: Verify Final State](#phase-8-verify-final-state)
- [Root Cause Analysis](#root-cause-analysis)
- [Key Concepts](#key-concepts)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This document details the end-to-end resolution of a git repository cleanup task performed on a remote storage server within the Stratos Datacenter. The objective was to truncate an inflated commit history down to exactly two commits (`initial commit` and `add data.txt file`) and synchronize the change to the remote origin via a force push.

---

## Problem Statement

The development team had been using the git repository at `/usr/src/kodekloudrepos/media` on the Nautilus Storage Server (`ststor01`) for test purposes. Multiple test commits were pushed, polluting the commit history. The repository needed to be cleaned so that:

1. Only two commits exist in the history: `initial commit` and `add data.txt file`
2. Both `HEAD` and the `master` branch point to the `add data.txt file` commit
3. The remote origin reflects the rewritten history

**Repository path:** `/usr/src/kodekloudrepos/media`
**Target server:** `ststor01` (Nautilus Storage Server)
**Access entry point:** `jump_host` (Thor user)

---

## Environment and Infrastructure

| Server Name | Hostname | User | Purpose |
|-------------|----------|------|---------|
| jump_host | jump_host.stratos.xfusioncorp.com | thor | Jump Server to Access Stratos DC |
| ststor01 | ststor01.stratos.xfusioncorp.com | natasha | Nautilus Storage Server |

---

## Prerequisites

- SSH access from `jump_host` to `ststor01`
- User `natasha` with `sudo` privileges on `ststor01`
- Git installed on `ststor01`
- Knowledge of the target commit hash or message

---

## Resolution Walkthrough

### Phase 1: Access the Storage Server

From the jump host, initiate an SSH session to the storage server using the `natasha` credentials.

```bash
thor@jumphost ~$ ssh natasha@ststor01.stratos.xfusioncorp.com
```

Accept the host fingerprint on first connection:

```
The authenticity of host 'ststor01.stratos.xfusioncorp.com (10.244.195.214)' can't be established.
ED25519 key fingerprint is SHA256:yEyN8qvzhNxfcKVE+H05zwQPmQMKCXj4JyGWuOP1HIg.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'ststor01.stratos.xfusioncorp.com' (ED25519) to the list of known hosts.
```

Verify identity and hostname:

```bash
[natasha@ststor01 ~]$ whoami
natasha

[natasha@ststor01 ~]$ hostname
ststor01
```

> **Screenshot**
<img width="1040" height="475" alt="image" src="https://github.com/user-attachments/assets/7548944f-aecc-40fb-b9f3-9089955662cf" />

> *Caption: Successful SSH session established from jump_host to ststor01 as user natasha*

---

### Phase 2: Navigate to the Repository

Change into the target git repository directory and confirm its contents.

```bash
[natasha@ststor01 ~]$ cd /usr/src/kodekloudrepos/media
[natasha@ststor01 media]$ pwd
/usr/src/kodekloudrepos/media

[natasha@ststor01 media]$ ls -la
```

**Expected output:**

```
total 20
drwxr-xr-x 3 root root 4096 Mar  8 07:30 .
drwxr-xr-x 3 root root 4096 Mar  8 07:30 ..
drwxr-xr-x 7 root root 4096 Mar  8 07:30 .git
-rw-r--r-- 1 root root   10 Mar  8 07:30 info.txt
-rw-r--r-- 1 root root   32 Mar  8 07:30 media.txt
```

> **Note:** The `.git` directory and all repository files are owned by `root`. This ownership mismatch is the source of the errors encountered in subsequent steps.

> **Screenshot Placeholder**
> ## Repository Directory Listing
> *Caption: ls -la output confirming .git directory and files are owned by root*

---

### Phase 3: Resolve Safe Directory Error

Attempting `git log` fails due to Git's security policy flagging the ownership mismatch between the current user (`natasha`) and the repository owner (`root`).

**Error encountered:**

```
fatal: detected dubious ownership in repository at '/usr/src/kodekloudrepos/media'
To add an exception for this directory, call:
        git config --global --add safe.directory /usr/src/kodekloudrepos/media
```

**Resolution:** Register the directory as a safe exception in the global git config for the current user.

```bash
[natasha@ststor01 media]$ git config --global --add safe.directory /usr/src/kodekloudrepos/media
```

> **Screenshot Placeholder**
> ## Safe Directory Error and Fix
> *Caption: Git dubious ownership fatal error followed by the safe.directory config command resolving it*

---

### Phase 4: Identify the Target Commit

With the safe directory exception in place, view the full commit log to locate the hash of the `add data.txt file` commit.

```bash
[natasha@ststor01 media]$ git log --oneline
```

**Output:**

```
75487f6 (HEAD -> master, origin/master) Test Commit10
8cdd55e Test Commit9
3f8e3a3 Test Commit8
2acf402 Test Commit7
e446a93 Test Commit6
4676dbc Test Commit5
b0f1a8a Test Commit4
8010be4 Test Commit3
29269ff Test Commit2
e15d612 Test Commit1
a2300e8 add data.txt file      <-- TARGET COMMIT
ce4570a initial commit
```

**Target commit hash identified:** `a2300e8`

> **Screenshot Placeholder**
> ### Full Commit History Before Reset
> *Caption: git log --oneline showing 12 commits including the 10 test commits to be removed*

---

### Phase 5: Escalate to Root

Attempting the reset as `natasha` fails because the `.git` directory is owned by `root` and `natasha` lacks write permissions to create the required lock file.

**Error encountered:**

```
fatal: Unable to create '/usr/src/kodekloudrepos/media/.git/index.lock': Permission denied
```

**Resolution:** Escalate to the root user via `sudo su -`.

```bash
[natasha@ststor01 media]$ sudo su -
[root@ststor01 ~]#
```

Navigate back to the repository and register the safe directory exception for the root user's git config:

```bash
[root@ststor01 ~]# cd /usr/src/kodekloudrepos/media
[root@ststor01 media]# git config --global --add safe.directory /usr/src/kodekloudrepos/media
```

> **Note:** The `safe.directory` exception is stored per-user in `~/.gitconfig`. It must be registered separately for each Unix user that needs to interact with the repository. Since `natasha` and `root` have separate home directories, both require their own exception entry.

> **Screenshot Placeholder**
> ### Permission Denied Error and Root Escalation
> *Caption: index.lock permission denied error as natasha, then sudo su - escalation to root*

---

### Phase 6: Reset Commit History

With root privileges, perform a hard reset to the target commit. This moves `HEAD` and the `master` branch pointer back to `a2300e8`, permanently discarding all commits made after it.

```bash
[root@ststor01 media]# git reset --hard a2300e8
```

**Output:**

```
HEAD is now at a2300e8 add data.txt file
```

> **What `--hard` does:** Resets the index and the working tree simultaneously. Any uncommitted changes and all commits after the target are permanently discarded. This operation cannot be undone via standard git commands once the reflog expires.

> **Screenshot Placeholder**
> ### Successful Hard Reset Output
> *Caption: git reset --hard confirming HEAD is now at a2300e8 add data.txt file*

---

### Phase 7: Force Push to Remote

Because the local history has been rewritten to a point behind the remote, a standard `git push` would be rejected with a non-fast-forward error. A force push is required to overwrite the remote branch history.

```bash
[root@ststor01 media]# git push -f origin master
```

**Output:**

```
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To /opt/media.git
 + 75487f6...a2300e8 master -> master (forced update)
```

The `+` symbol and `(forced update)` label confirm the remote branch was successfully overwritten.

> **Warning:** Force pushing rewrites public history. This is intentional in this cleanup scenario but must be coordinated with all collaborators on shared branches in production environments, as their local copies will diverge.

> **Screenshot Placeholder**
> ### Force Push Confirmation
> *Caption: git push -f output confirming forced update from 75487f6 to a2300e8 on origin/master*

---

### Phase 8: Verify Final State

Confirm that the repository history now contains exactly two commits and that the local and remote refs are fully aligned.

```bash
[root@ststor01 media]# git log --oneline
```

**Final output:**

```
a2300e8 (HEAD -> master, origin/master) add data.txt file
ce4570a initial commit
```

**Verification checklist:**

- [x] Exactly 2 commits in history
- [x] `HEAD` points to `add data.txt file`
- [x] `master` branch points to `add data.txt file`
- [x] `origin/master` is in sync with local `master`
- [x] All 10 test commits have been permanently removed

> **Screenshot Placeholder**
> ### Final Git Log Verification
> *Caption: git log --oneline showing only 2 commits with HEAD, master, and origin/master all aligned at a2300e8*

---

## Root Cause Analysis

| Issue | Root Cause | Resolution |
|-------|------------|------------|
| `fatal: detected dubious ownership` | Git CVE-2022-24765 security policy blocks operations when the directory owner differs from the current user | Added `safe.directory` exception via `git config --global` for each user |
| `Permission denied: .git/index.lock` | The `.git` directory is owned by `root`; user `natasha` has no write access to create the lock file | Escalated privileges to `root` via `sudo su -` |
| Force push required | Local history was rewritten to behind the remote tip; standard push rejected as non-fast-forward | Used `git push -f origin master` to overwrite remote history |

---

## Key Concepts

**`git reset --hard <commit>`**
Moves `HEAD` and the current branch pointer to the specified commit. Discards all subsequent commits, staged changes, and working directory modifications. This is a destructive, irreversible operation on published history and should only be used when history rewriting is explicitly required.

**`git push -f` (Force Push)**
Overwrites the remote branch history with the local branch state regardless of divergence. Necessary when local history has been rewritten via `reset`, `rebase`, or `commit --amend` and the remote must be updated to match. Use with caution on branches shared with other contributors.

**Git Safe Directory (CVE-2022-24765)**
Introduced in Git 2.35.2 as a security hardening measure. Git refuses to operate on repositories whose owning user differs from the invoking user unless an explicit `safe.directory` exception is configured in `~/.gitconfig`. The exception is scoped per Unix user and must be set independently for each user that needs access.

---

## Lessons Learned

* Always inspect repository ownership with `ls -la` before running git operations on shared or system-owned directories to anticipate permission issues early.
* The `safe.directory` configuration is stored per-user. In multi-user scenarios, the exception must be registered separately for each Unix account that interacts with the repository.
* Hard resets on shared repositories must be communicated to all collaborators in advance. Any contributor with a local clone will need to reset their local copy or re-clone after a force push.
* Performing destructive git operations as the repository owner (here `root`) avoids partial failure states caused by mid-operation permission errors such as the `index.lock` issue encountered above.
* When escalating to root via `sudo su -`, always re-register the `safe.directory` exception for the root context before running git commands, even if it was already set for the original user.

---

## References

* [Git Documentation: git-reset](https://git-scm.com/docs/git-reset)
* [Git Documentation: git-push](https://git-scm.com/docs/git-push)
* [CVE-2022-24765: Git Safe Directory Security Fix](https://github.blog/2022-04-12-git-security-vulnerability-announced/)
* [KodeKloud Engineer Labs](https://kodekloud.com/courses/kodekloud-engineer/)

---

<img width="1038" height="493" alt="image" src="https://github.com/user-attachments/assets/1a8079d4-1840-468b-8e3c-30aee80f6651" />

<img width="1033" height="448" alt="image" src="https://github.com/user-attachments/assets/2a992946-dd58-4556-b60b-6a8867886aea" />
<img width="1030" height="474" alt="image" src="https://github.com/user-attachments/assets/95a6d1df-a825-4656-83f1-da7c1493551b" />
<img width="1031" height="604" alt="image" src="https://github.com/user-attachments/assets/0d1c2d55-ad16-4dcd-a878-520655cc2720" />
<img width="1039" height="591" alt="image" src="https://github.com/user-attachments/assets/3552a6f8-0133-4ba5-96af-6ebe6e7fb53a" />
<img width="1014" height="525" alt="image" src="https://github.com/user-attachments/assets/4e803f53-8078-4abe-aabf-58fcf83c23a3" />
<img width="1039" height="832" alt="image" src="https://github.com/user-attachments/assets/46d7a52b-2431-4a70-a4d9-468d2c8b78c5" />
<img width="1035" height="864" alt="image" src="https://github.com/user-attachments/assets/ef2e0af1-932b-46aa-b193-fa276d49524e" />
<img width="1025" height="861" alt="image" src="https://github.com/user-attachments/assets/55e88d3a-c231-4d92-973a-8d6321b2896f" />
<img width="1036" height="544" alt="image" src="https://github.com/user-attachments/assets/a5bf5eec-2326-49fc-8fdb-986d994096cb" />
<img width="1031" height="576" alt="image" src="https://github.com/user-attachments/assets/ed766cba-ea66-404f-b263-f20102f716aa" />



