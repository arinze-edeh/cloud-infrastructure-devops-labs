# Docker Container-to-Image Conversion

> **Scope:** DevOps | Docker | Container Image Engineering
> **Difficulty:** Foundational
> **Infrastructure:** Nautilus Project | Stork DC

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Infrastructure Context](#infrastructure-context)
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Phase 1: SSH into Application Server 2](#phase-1-ssh-into-application-server-2)
  - [Phase 2: Verify the Running Container](#phase-2-verify-the-running-container)
  - [Phase 3: Commit Container to Image](#phase-3-commit-container-to-image)
  - [Phase 4: Verify the Created Image](#phase-4-verify-the-created-image)
  - [Phase 5: Exit and Return to Jump Host](#phase-5-exit-and-return-to-jump-host)
- [Validation Checklist](#validation-checklist)
- [Error Reference and Resolutions](#error-reference-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Command Reference Summary](#command-reference-summary)

---

## Problem Statement

A Nautilus developer made iterative changes inside a running container and needed those changes preserved as a reusable image. A formal request was raised to the DevOps team to capture the current state of the container and produce a new Docker image from it.

**Requirement:**
- Create image **`beta:xfusion`** on **Application Server 2** (`stapp02`)
- Source: container named **`ubuntu_latest`** running on the same server
- Entry point: **Jump Host Server** (`jump-host`) via SSH tunnel

---

## Infrastructure Context

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| Jump Host Server | `jump-host` | `thor` | Secure access gateway to Stork DC |
| Application Server 2 | `stapp02` | `steve` | Target server hosting `ubuntu_latest` container |

> **Note:** All server credentials are managed by the infrastructure team. Refer to the Nautilus Infrastructure Details document for the full server inventory.

---

## Prerequisites

Before executing this task, confirm the following:

- [ ] You have SSH access to `jump-host` as user `thor`
- [ ] Docker is installed and the daemon is running on `stapp02`
- [ ] The container `ubuntu_latest` is in a **running** state on `stapp02`
- [ ] Your user (`steve`) has `sudo` privileges on `stapp02`
- [ ] No naming conflict exists for the image `beta:xfusion` on `stapp02`

---

## Architecture Overview

```
[Local Machine]
      |
      | SSH
      v
[jump-host] (thor)
      |
      | SSH --> stapp02 (steve)
      v
[Application Server 2]
      |
      | docker commit
      v
[Container: ubuntu_latest] --> [Image: beta:xfusion]
```

The workflow follows a **Jump Host pattern**: all access to internal servers routes through the designated jump host, enforcing network segmentation and access control within Stork DC.

---

## Step-by-Step Implementation

### Phase 1: SSH into Application Server 2

From `jump-host`, initiate an SSH session to Application Server 2 using the `steve` account.

```bash
ssh steve@stapp02
```

When the host authenticity prompt appears (first-time connection):

```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

Enter password when prompted:

```
Am3ric@
```

**Verify correct server identity immediately after login:**

```bash
whoami
```

Expected output:

```
steve
```

```bash
hostname
```

Expected output:

```
stapp02
```

> **Why this matters:** Confirming identity before executing any Docker commands prevents accidental operations on the wrong server, a critical discipline in multi-server environments.

**Screenshot Placeholder:**

```
[ SCREENSHOT 1: Terminal showing successful SSH login, whoami output "steve",
  hostname output "stapp02" ]
```

---

### Phase 2: Verify the Running Container

Before committing, confirm the container `ubuntu_latest` is active and in a running state.

```bash
sudo docker ps
```

Enter sudo password when prompted:

```
Am3ric@
```

**Expected output columns to verify:**

| Column | Expected Value |
|---|---|
| `IMAGE` | `ubuntu` |
| `STATUS` | `Up X minutes` |
| `NAMES` | `ubuntu_latest` |

**Full expected output format:**

```
CONTAINER ID   IMAGE    COMMAND       CREATED          STATUS           PORTS   NAMES
3d3317424c70   ubuntu   "/bin/bash"   12 minutes ago   Up 12 minutes            ubuntu_latest
```

> **If the container shows `Exited` status**, run `sudo docker start ubuntu_latest` before proceeding.
> **If the container does not appear at all**, run `sudo docker ps -a` to check all containers including stopped ones.

**Screenshot Placeholder:**

```
[ SCREENSHOT 2: Terminal showing sudo docker ps output with ubuntu_latest
  container in "Up" status ]
```

---

### Phase 3: Commit Container to Image

With the container confirmed running, execute `docker commit` to convert the container's current filesystem state into a new image tagged `beta:xfusion`.

```bash
sudo docker commit ubuntu_latest beta:xfusion
```

**Expected output:**

```
sha256:cc043da0e517bcaf66af1d42e80bfcb40d7797013aab72af354f1cbcba3f191f
```

> The SHA256 digest is the content-addressable identifier for the newly created image layer. Its presence confirms the image was written successfully to the local Docker image store.

> **If no output is returned or an error appears**, do not proceed to Phase 4. Revisit Phase 2 to confirm the container state.

**Screenshot Placeholder:**

```
[ SCREENSHOT 3: Terminal showing the docker commit command and the returned
  SHA256 digest ]
```

---

### Phase 4: Verify the Created Image

Confirm the image `beta:xfusion` was created correctly with the right repository name, tag, and a valid image ID.

```bash
sudo docker images beta:xfusion
```

**Expected output:**

```
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
beta         xfusion   cc043da0e517   15 seconds ago   138MB
```

**Verification checklist for this output:**

- ✅ `REPOSITORY` is exactly `beta` (case-sensitive)
- ✅ `TAG` is exactly `xfusion` (case-sensitive)
- ✅ `IMAGE ID` is a valid 12-character hex string
- ✅ `CREATED` shows seconds or minutes ago (not a stale timestamp)
- ✅ `SIZE` is a non-zero value

> **If the output returns only headers with no rows**, the commit in Phase 3 did not succeed. Re-run Phase 3.

**Screenshot Placeholder:**

```
[ SCREENSHOT 4: Terminal showing sudo docker images beta:xfusion output
  with REPOSITORY "beta", TAG "xfusion", size 138MB ]
```

---

### Phase 5: Exit and Return to Jump Host

Once image creation is verified, cleanly terminate the SSH session and return to the jump host.

```bash
exit
```

**Expected output on the terminal:**

```
logout
Connection to stapp02 closed.
```

**Confirm you are back on the jump host:**

```bash
whoami
```

Expected:

```
thor
```

```bash
hostname
```

Expected:

```
jump-host
```

**Screenshot Placeholder:**

```
[ SCREENSHOT 5: Terminal showing exit logout message, connection closed,
  then whoami "thor" and hostname "jump-host" ]
```

---

## Validation Checklist

Use this checklist as your final gate before marking the task complete.

| Step | Action | Expected Result | Status |
|---|---|---|---|
| 1 | SSH to `stapp02` as `steve` | Shell prompt shows `[steve@stapp02 ~]$` | ✅ |
| 2 | `whoami` on `stapp02` | `steve` | ✅ |
| 3 | `hostname` on `stapp02` | `stapp02` | ✅ |
| 4 | `sudo docker ps` | `ubuntu_latest` shows `Up` status | ✅ |
| 5 | `sudo docker commit ubuntu_latest beta:xfusion` | SHA256 digest returned | ✅ |
| 6 | `sudo docker images beta:xfusion` | `beta xfusion` row visible with valid size | ✅ |
| 7 | `exit` from `stapp02` | `Connection to stapp02 closed` | ✅ |
| 8 | `whoami` on jump host | `thor` | ✅ |
| 9 | `hostname` on jump host | `jump-host` | ✅ |

---

## Error Reference and Resolutions

| Error Message | Root Cause | Resolution |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker service not running | `sudo systemctl start docker` then retry |
| `No such container: ubuntu_latest` | Container name mismatch or container not created | Run `sudo docker ps -a` to find actual container name |
| `permission denied` on docker commands | Running without elevated privileges | Prefix all Docker commands with `sudo` |
| `invalid reference format` | Incorrect image name syntax | Ensure format is `repository:tag` with no spaces; verify it is `beta:xfusion` |
| `ssh: connect to host stapp02 port 22: Connection refused` | Wrong hostname or SSH service down | Confirm target hostname is `stapp02`; verify SSH service with `sudo systemctl status sshd` |
| `sudo: command not found` | Minimal container shell inherited | Ensure you are on `stapp02` host, not inside the container shell |
| Empty output from `docker images beta:xfusion` | Commit failed silently | Re-run `sudo docker commit ubuntu_latest beta:xfusion` and check for errors |

---

## Best Practices

**Access Control**

- Always route into production servers via the designated jump host. Never attempt direct access bypassing the jump host gateway.
- Verify `whoami` and `hostname` immediately after every SSH login before running any commands.

**Container State Management**

- Always confirm the container is in `Up` (running) state before committing. Committing an `Exited` container captures a potentially incomplete or inconsistent state.
- Use `docker inspect <container>` for deeper state validation on critical workloads before committing.

**Image Naming Conventions**

- Follow the `repository:tag` format strictly. Tags must be lowercase, alphanumeric, and use hyphens or dots as separators.
- Never commit to `latest` tag in production workflows. Always use explicit, descriptive tags to maintain traceability.
- Prefix image names with project or team identifiers (e.g., `nautilus/beta:xfusion`) in shared registries to avoid namespace collisions.

**Commit Hygiene**

- `docker commit` captures the entire writable layer of a container, including temporary files, logs, and cached package data. For production images, prefer `Dockerfile`-based builds for reproducibility and auditability.
- Document what changes were made inside the container before committing. Without documentation, a committed image becomes a black box.
- After committing, immediately verify with `docker images` to confirm the image metadata before exiting the server.

**Session Management**

- Always `exit` SSH sessions explicitly. Do not close terminal tabs without terminating the session to avoid leaving orphaned connections.
- Verify you have returned to the correct host (`jump-host`) after exiting to prevent subsequent commands from running on the wrong server.

---

## Lessons Learned

**1. Identity Verification is Non-Negotiable**

Running `whoami` and `hostname` immediately after SSH login caught any potential misrouting before executing any state-changing commands. In environments with multiple servers sharing similar naming conventions (`stapp01`, `stapp02`, `stapp03`), this discipline prevents costly mistakes.

**2. Container Status Must Be Confirmed Before Commit**

The `docker commit` command will succeed on a stopped container, but the captured state may not reflect the developer's intended changes if the container crashed or was stopped mid-operation. Checking `STATUS` in `docker ps` before committing is a mandatory gate, not an optional step.

**3. SHA256 Output Confirms Write Success**

The SHA256 digest returned by `docker commit` is not cosmetic. It is the content-addressable ID of the committed image layer. Absence of this output indicates the operation did not complete. Always treat a missing digest as a failure signal.

**4. Targeted Image Verification Over Broad Listing**

Using `sudo docker images beta:xfusion` instead of `sudo docker images` (which lists all images) provides a focused, unambiguous confirmation that the exact target image was created. In environments with many images, broad listing introduces risk of misreading results.

**5. Jump Host Architecture Enforces Discipline**

The two-hop SSH pattern (local to jump-host, jump-host to target server) reinforces the practice of intentional access. Each hop requires explicit confirmation, reducing the likelihood of accidental commands on wrong targets compared to direct server access.

**6. Sudo Password Reuse Awareness**

On systems configured with `sudo` requiring a password, the same user account password is used for both SSH login and sudo elevation. Confirm sudo works immediately after login (Phase 2) rather than discovering a sudo misconfiguration mid-task.

---

## Command Reference Summary

```bash
# Phase 1: SSH from jump host to Application Server 2
ssh steve@stapp02

# Phase 1: Verify identity
whoami
hostname

# Phase 2: Confirm running container
sudo docker ps

# Phase 3: Commit container state to new image
sudo docker commit ubuntu_latest beta:xfusion

# Phase 4: Verify image creation
sudo docker images beta:xfusion

# Phase 5: Exit back to jump host
exit

# Phase 5: Confirm return to jump host
whoami
hostname
```

---


<img width="1032" height="503" alt="image" src="https://github.com/user-attachments/assets/557c68d0-442f-4bdc-ac59-22e7e7157869" />
<img width="1028" height="598" alt="image" src="https://github.com/user-attachments/assets/ca20e638-b666-46da-800a-77f2e1e8e36f" />
<img width="1027" height="659" alt="image" src="https://github.com/user-attachments/assets/12cde6e9-4eec-4a59-b57c-273479e72b3f" />
<img width="1030" height="815" alt="image" src="https://github.com/user-attachments/assets/23eda709-20b9-49fd-a32e-65e276c159c4" />
