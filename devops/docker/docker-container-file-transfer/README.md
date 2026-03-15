
# Docker Container File Transfer: Encrypted File Copy Operation

[![Docker](https://img.shields.io/badge/Docker-Container_Operations-2496ED?style=flat-square&logo=docker)](https://www.docker.com/)
[![Platform](https://img.shields.io/badge/Platform-Linux-FCC624?style=flat-square&logo=linux)](https://www.linux.org/)
[![Status](https://img.shields.io/badge/Status-Resolved-28a745?style=flat-square)]()
[![Infrastructure](https://img.shields.io/badge/Environment-Stratos_Datacenter-0A66C2?style=flat-square)]()

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Infrastructure Overview](#infrastructure-overview)
- [Prerequisites](#prerequisites)
- [Solution Architecture](#solution-architecture)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Phase 1: Establish SSH Connection to App Server 2](#phase-1-establish-ssh-connection-to-app-server-2)
  - [Phase 2: Verify Source File Integrity](#phase-2-verify-source-file-integrity)
  - [Phase 3: Confirm Container Runtime Status](#phase-3-confirm-container-runtime-status)
  - [Phase 4: Execute File Transfer to Container](#phase-4-execute-file-transfer-to-container)
  - [Phase 5: Post-Transfer Integrity Verification](#phase-5-post-transfer-integrity-verification)
- [Verification Results](#verification-results)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting Reference](#troubleshooting-reference)

---

## Problem Statement

The Nautilus DevOps team manages confidential encrypted data on **Application Server 2** within the **Stratos Datacenter**. A Docker container named `ubuntu_latest` runs on the same host.

**Objective:** Securely copy the encrypted file `/tmp/nautilus.txt.gpg` from the Docker host filesystem into the `ubuntu_latest` container at `/tmp/`, guaranteeing zero file modification throughout the entire operation.

**Key Constraint:** The file must arrive inside the container byte-for-byte identical to the source. Any modification invalidates the GPG encryption integrity.

---

## Infrastructure Overview

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| Jump Host Server | `jump-host` | `thor` | Secure gateway into Stratos DC |
| Application Server 2 | `stapp02` | `steve` | Hosts `ubuntu_latest` Docker container |

**Access Path:**
```
Local Workstation --> thor@jump-host --> steve@stapp02 --> ubuntu_latest container
```

---

## Prerequisites

Before executing this runbook, confirm all of the following are in place:

- [ ] SSH access to `jump-host` as user `thor`
- [ ] Valid credentials for `steve@stapp02`
- [ ] Docker daemon running on `stapp02`
- [ ] Source file `/tmp/nautilus.txt.gpg` present on `stapp02`
- [ ] Container `ubuntu_latest` in `Up` status
- [ ] `sudo` privileges for `steve` on `stapp02`

---

## Solution Architecture

```
+------------------+        SSH          +------------------+       docker cp      +----------------------+
|  thor@jump-host  |  ---------------->  |  steve@stapp02   |  -----------------> |  ubuntu_latest:/tmp/ |
|  (Gateway)       |                     |  (Docker Host)   |                      |  (Container)         |
+------------------+                     +------------------+                      +----------------------+
                                               |
                                         /tmp/nautilus.txt.gpg
                                         (Source: Docker Host FS)
```

The `docker cp` command operates at the filesystem layer, bypassing container networking entirely. It reads directly from the host filesystem and writes into the container's overlay filesystem without executing any process inside the container, ensuring the GPG-encrypted file is never decrypted, modified, or processed during transit.

---

## Step-by-Step Implementation

### Phase 1: Establish SSH Connection to App Server 2

From the jump host, initiate an SSH session to Application Server 2.

**Command:**
```bash
ssh steve@stapp02
```

**When prompted for host authenticity, type `yes` to accept and permanently add the fingerprint:**
```
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

**Enter the password when prompted:**
```
steve@stapp02's password: Am3ric@
```

**Expected Output:**
```
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
[steve@stapp02 ~]$
```

> **NOTE:** The ED25519 fingerprint warning is expected on first connection. This is a one-time event; subsequent connections will not prompt again.

---

**SCREENSHOT PLACEHOLDER**
> *Screenshot 1: Terminal showing successful SSH login from `thor@jump-host` to `steve@stapp02`, including the host authenticity prompt and password entry.*

---

### Phase 2: Verify Source File Integrity

Before any transfer, confirm the source file exists and capture its cryptographic checksum as a baseline for post-transfer comparison.

**Step 2a: Confirm file existence and metadata:**
```bash
ls -lh /tmp/nautilus.txt.gpg
```

**Expected Output:**
```
-rw-r--r-- 1 root root 105 Mar 15 03:55 /tmp/nautilus.txt.gpg
```

**Step 2b: Capture MD5 checksum of the source file:**
```bash
md5sum /tmp/nautilus.txt.gpg
```

**Expected Output:**
```
fb841ddf1d76475f2849828daf7708fb  /tmp/nautilus.txt.gpg
```

> **CRITICAL:** Record the MD5 hash value. This is your integrity baseline. The identical hash **must** appear after the file is copied into the container.

---

**SCREENSHOT PLACEHOLDER**
> *Screenshot 2: Terminal output showing `ls -lh` confirming file size of 105 bytes and `md5sum` output displaying hash `fb841ddf1d76475f2849828daf7708fb`.*

---

### Phase 3: Confirm Container Runtime Status

Verify the `ubuntu_latest` container is active and running before attempting the copy operation.

**Command:**
```bash
sudo docker ps
```

**Enter sudo password when prompted:**
```
[sudo] password for steve: Am3ric@
```

**Expected Output:**
```
CONTAINER ID   IMAGE    COMMAND       CREATED          STATUS          PORTS     NAMES
b197803ab7e2   ubuntu   "/bin/bash"   11 minutes ago   Up 11 minutes             ubuntu_latest
```

**Key fields to verify:**

| Field | Expected Value | Significance |
|---|---|---|
| `IMAGE` | `ubuntu` | Confirms correct base image |
| `STATUS` | `Up X minutes` | Container is running and healthy |
| `NAMES` | `ubuntu_latest` | Target container confirmed |

> **WARNING:** If `ubuntu_latest` is not listed or its STATUS shows `Exited`, the container must be started before proceeding. Do **not** attempt `docker cp` against a stopped container if the intent is to verify live state.

---

**SCREENSHOT PLACEHOLDER**
> *Screenshot 3: Terminal output of `sudo docker ps` showing container `ubuntu_latest` with status `Up 11 minutes` and container ID `b197803ab7e2`.*

---

### Phase 4: Execute File Transfer to Container

Use the `docker cp` command to copy the encrypted file from the host filesystem into the container. This command preserves file content, permissions, and timestamps.

**Command:**
```bash
sudo docker cp /tmp/nautilus.txt.gpg ubuntu_latest:/tmp/
```

**Expected Output:**
```
Successfully copied 2.05kB to ubuntu_latest:/tmp/
```

**Command Syntax Breakdown:**

| Component | Value | Description |
|---|---|---|
| `sudo docker cp` | Command | Executes Docker copy with elevated privileges |
| `/tmp/nautilus.txt.gpg` | Source path | Absolute path on the Docker host filesystem |
| `ubuntu_latest:/tmp/` | Destination | Container name followed by target path inside container |

> **NOTE:** The trailing `/` in `ubuntu_latest:/tmp/` explicitly targets a directory, preventing accidental filename overwrites and ensuring the original filename is preserved.

---

**SCREENSHOT PLACEHOLDER**
> *Screenshot 4: Terminal showing the `docker cp` command and the `Successfully copied 2.05kB to ubuntu_latest:/tmp/` confirmation message.*

---

### Phase 5: Post-Transfer Integrity Verification

Verify the file was transferred correctly by confirming its presence, metadata, and cryptographic checksum inside the container. The hash **must** match the value captured in Phase 2.

**Step 5a: Confirm file exists inside the container:**
```bash
sudo docker exec ubuntu_latest ls -lh /tmp/nautilus.txt.gpg
```

**Expected Output:**
```
-rw-r--r-- 1 root root 105 Mar 15 03:55 /tmp/nautilus.txt.gpg
```

**Step 5b: Verify MD5 checksum inside the container:**
```bash
sudo docker exec ubuntu_latest md5sum /tmp/nautilus.txt.gpg
```

**Expected Output:**
```
fb841ddf1d76475f2849828daf7708fb  /tmp/nautilus.txt.gpg
```

> **SUCCESS CRITERIA:** The MD5 hash `fb841ddf1d76475f2849828daf7708fb` inside the container is identical to the host source hash. The file was not modified during the operation.

---

**SCREENSHOT PLACEHOLDER**
> *Screenshot 5: Terminal output showing both `docker exec` commands confirming file size of 105 bytes, timestamp `Mar 15 03:55`, and matching MD5 hash `fb841ddf1d76475f2849828daf7708fb` inside the `ubuntu_latest` container.*

---

## Verification Results

| Verification Check | Host (Source) | Container (Destination) | Match |
|---|---|---|---|
| **File Path** | `/tmp/nautilus.txt.gpg` | `/tmp/nautilus.txt.gpg` | PASS |
| **File Size** | `105 bytes` | `105 bytes` | PASS |
| **Permissions** | `-rw-r--r--` | `-rw-r--r--` | PASS |
| **Owner** | `root root` | `root root` | PASS |
| **Timestamp** | `Mar 15 03:55` | `Mar 15 03:55` | PASS |
| **MD5 Hash** | `fb841ddf1d76475f2849828daf7708fb` | `fb841ddf1d76475f2849828daf7708fb` | **PASS** |

**Overall Status: ALL CHECKS PASSED. File integrity confirmed.**

---

## Best Practices

### Security

- **Never hardcode passwords** in scripts or shell history. Use SSH key-based authentication for production environments to eliminate password exposure.
- **Validate GPG file integrity** using checksums before and after every transfer, not just for audits, but as a standard operating procedure.
- **Apply the principle of least privilege.** The `steve` user uses `sudo` only for Docker commands, not as the root user directly.
- **Audit container access.** Use `docker events` or centralized logging (e.g., Splunk, ELK) to track all `docker cp` and `docker exec` operations in production.

### Operational

- **Always verify container status** with `docker ps` before executing `docker cp`. Copying to a stopped container does work at the filesystem level but confirms nothing about runtime state.
- **Use absolute paths** for both source and destination in `docker cp`. Relative paths introduce ambiguity and are a leading cause of misdirected file transfers.
- **Capture checksums before and after** every sensitive file operation. This two-point verification is the only reliable way to confirm zero modification.
- **Use `docker exec` for in-container verification** rather than mounting volumes or using `docker inspect`, which is more invasive and introduces additional operational risk.
- **Prefer `md5sum` or `sha256sum`** over `ls` alone. File size matching is necessary but not sufficient; only a hash confirms content integrity.

### Docker Specific

- **Use named containers** (like `ubuntu_latest`) rather than container IDs in operational scripts. IDs change on every container recreation; names are stable.
- **`docker cp` does not require a running container** for filesystem access, but always verify runtime state before executing operational tasks.
- **Avoid `docker exec` with interactive shells** (`-it`) in automated pipelines. Use non-interactive single-command execution as demonstrated in this runbook.

---

## Lessons Learned

**1. `docker cp` preserves file integrity by design.**
The command copies at the block level without interpreting file content. GPG-encrypted files, binary blobs, and sensitive archives can be transferred safely without risk of decryption, encoding conversion, or content alteration.

**2. MD5 checksum verification is non-negotiable for encrypted files.**
A file can arrive at the destination with the correct size and permissions but with corrupted content due to filesystem errors or interrupted transfers. Only a cryptographic hash confirms true byte-for-byte fidelity.

**3. The jump host architecture adds an SSH hop but does not affect Docker operations.**
All Docker commands execute on the application server directly. The jump host is purely a network gateway. Understanding this separation prevents confusion about where to run commands.

**4. `sudo` privilege escalation requires password on first use per session.**
The first `sudo` command in a session triggers the password prompt. Subsequent `sudo` commands within the default timeout window (typically 5 minutes) do not. Plan command sequences accordingly in time-sensitive operations.

**5. Trailing slash in destination paths matters.**
`ubuntu_latest:/tmp/` (with slash) copies the file **into** the `/tmp/` directory, preserving the original filename. `ubuntu_latest:/tmp` (without slash) could be interpreted as a rename target in some edge cases. Always use the trailing slash when targeting a directory.

**6. Host key verification on first SSH connection is expected, not an error.**
The ED25519 fingerprint warning is a standard SSH security prompt, not an indication of a man-in-the-middle attack in a known controlled datacenter environment. After accepting, the fingerprint is permanently stored in `~/.ssh/known_hosts`.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `No such container: ubuntu_latest` | Container is stopped or does not exist | Run `sudo docker ps -a` to check all containers; start with `sudo docker start ubuntu_latest` |
| `No such file or directory: /tmp/nautilus.txt.gpg` | Source file missing on host | Confirm file location with `find /tmp -name "*.gpg"` |
| `Permission denied` on `docker cp` | Missing sudo | Prepend `sudo` to the `docker cp` command |
| MD5 hash mismatch after copy | Filesystem error or interrupted transfer | Re-run `docker cp` and re-verify; check disk health with `df -h` |
| SSH connection refused to `stapp02` | Network issue or SSH service down | Verify connectivity with `ping stapp02`; check SSH service on target |
| `sudo: command not found` | Minimal container image lacks sudo | Use `docker exec` as root: `sudo docker exec -u root ubuntu_latest md5sum /tmp/nautilus.txt.gpg` |

---

## Full Command Reference

```bash
# Phase 1: Connect to App Server 2 from jump host
ssh steve@stapp02
# Password: Am3ric@

# Phase 2: Verify source file on Docker host
ls -lh /tmp/nautilus.txt.gpg
md5sum /tmp/nautilus.txt.gpg

# Phase 3: Confirm container is running
sudo docker ps
# sudo password: 

# Phase 4: Copy encrypted file into container
sudo docker cp /tmp/nautilus.txt.gpg ubuntu_latest:/tmp/

# Phase 5: Verify file inside container
sudo docker exec ubuntu_latest ls -lh /tmp/nautilus.txt.gpg
sudo docker exec ubuntu_latest md5sum /tmp/nautilus.txt.gpg
```

---

<img width="1032" height="362" alt="Image" src="https://github.com/user-attachments/assets/7b968d3d-9bdb-4ec1-999d-d2029a3e1b54" />

<img width="1032" height="398" alt="Image" src="https://github.com/user-attachments/assets/c216b362-0efa-452d-a69b-e75d804f5a6d" />

<img width="1029" height="584" alt="Image" src="https://github.com/user-attachments/assets/760ebf8c-a461-4cc2-8676-24db7e38003b" />

<img width="1032" height="655" alt="Image" src="https://github.com/user-attachments/assets/d64771fd-a6ab-49f0-beac-764fb918fc97" />

<img width="1029" height="708" alt="Image" src="https://github.com/user-attachments/assets/8458c65a-c519-4a5a-adea-289dc6ccd98f" />

