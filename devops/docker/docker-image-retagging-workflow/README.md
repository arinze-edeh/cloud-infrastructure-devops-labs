# Docker Image Management: Pull and Re-tag on Remote Application Server

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Shell Script](https://img.shields.io/badge/Shell_Script-121011?style=for-the-badge&logo=gnu-bash&logoColor=white)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure Context](#infrastructure-context)
- [Prerequisites](#prerequisites)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Phase 1: Establish SSH Connection to Application Server 3](#phase-1-establish-ssh-connection-to-application-server-3)
  - [Phase 2: Verify Docker Service Health](#phase-2-verify-docker-service-health)
  - [Phase 3: Pull the Target Docker Image](#phase-3-pull-the-target-docker-image)
  - [Phase 4: Re-tag the Image](#phase-4-re-tag-the-image)
  - [Phase 5: Validate the Operation](#phase-5-validate-the-operation)
- [Verification and Acceptance Criteria](#verification-and-acceptance-criteria)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Reference Commands](#reference-commands)

---

## Overview

This document details the end-to-end operational procedure for pulling a containerized image from Docker Hub onto a remote application server within the **Nautilus DC** environment and applying a new tag for downstream use. The task was performed as part of the Nautilus project initiative to validate containerized environment application features in collaboration with the DevOps team.

**Scope:** Stratos DC, Application Server 3 (`stapp03`)
**Access Model:** Bastion/Jump Host architecture via `jump-host`
**Outcome:** `busybox:musl` pulled and re-tagged as `busybox:blog` successfully

---

## Problem Statement

The Nautilus project development team required a specific Docker image (`busybox:musl`) to be available on **Application Server 3** in Stratos DC, with a new custom tag (`busybox:blog`) created to support containerized feature testing workflows. Direct access to application servers is restricted by network policy; all operations must transit through the designated Jump Host.

**Task Requirements:**

* Pull `busybox:musl` from Docker Hub on Application Server 3
* Re-tag (create a new tag) the pulled image as `busybox:blog`
* Confirm both tags resolve to the same image digest

---

## Infrastructure Context

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| Jump Host Server | `jump-host` | `thor` | Secure bastion access into Stratos DC |
| Application Server 3 | `stapp03` | `banner` | Target host for containerized workloads |

> **Security Note:** Credentials referenced in this document are lab-environment credentials. In production environments, use SSH key-based authentication and retrieve secrets from a secrets manager such as HashiCorp Vault or AWS Secrets Manager. Never hardcode credentials in scripts or documentation.

---

## Prerequisites

Before executing this procedure, confirm the following:

* You are authenticated on the Jump Host as user `thor`
* Network connectivity exists between `jump-host` and `stapp03`
* User `banner` has `sudo` privileges on `stapp03`
* Outbound internet access is available on `stapp03` to reach Docker Hub (`registry-1.docker.io`)
* Docker Engine is installed on `stapp03`

---

## Resolution Walkthrough

### Phase 1: Establish SSH Connection to Application Server 3

From the Jump Host, initiate an SSH session to Application Server 3 using its hostname.

```bash
ssh banner@stapp03
```

When prompted about host authenticity on first-time connection, type `yes` to accept and permanently add the host fingerprint to known hosts.

```
The authenticity of host 'stapp03 (10.244.195.4)' can't be established.
ED25519 key fingerprint is SHA256:M3WlMiXpydvnTbg7grAulqXGrDgXmV0fQ0HuwintO7w.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
```

Enter the password for user `banner` when prompted.

> **SCREENSHOT**

<img width="1032" height="499" alt="Image" src="https://github.com/user-attachments/assets/deb54cf1-db63-440a-8989-55b9bc3c3aad" />

> *Jump Host terminal showing successful SSH connection to `stapp03`, host key acceptance prompt, and password prompt.*

Confirm you are on the correct host and logged in as the correct user:

```bash
hostname
```

**Expected output:**
```
stapp03
```

```bash
whoami
```

**Expected output:**
```
banner
```

> **SCREENSHOT**

<img width="1034" height="443" alt="Image" src="https://github.com/user-attachments/assets/39ab8510-8f2b-4ea2-82cd-8f2b2c04a5d3" />

> *Terminal output confirming hostname as `stapp03` and current user as `banner`.*



---

### Phase 2: Verify Docker Service Health

Before performing any Docker operations, confirm the Docker Engine is active and running. This step prevents wasted pull attempts against a stopped or degraded daemon.

```bash
sudo systemctl status docker
```

Enter the `sudo` password for `banner` when prompted.

**Expected output (key indicators):**
```
docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
   Active: active (running) since Mon 2026-03-16 21:22:34 UTC; 36min ago
 Main PID: 1381 (dockerd)
      Tasks: 18
     Memory: 29.5M (peak: 30.9M)
        CPU: 373ms
Mar 16 21:22:34 stapp03 systemd[1]: Started Docker Application Container Engine.
```

Verify the following before proceeding:

* `Active:` shows **`active (running)`** in green
* `Main PID` is populated with a valid process ID
* The final log entry confirms `Started Docker Application Container Engine`

> **[SCREENSHOT-03]**
> *Full `systemctl status` output for `docker.service` showing `active (running)` state, PID `1381`, and systemd startup log lines confirming daemon initialization.*

If Docker is not running, start it with:

```bash
sudo systemctl start docker
```

---

### Phase 3: Pull the Target Docker Image

Pull `busybox:musl` from the official Docker Hub registry:

```bash
sudo docker pull busybox:musl
```

Docker resolves the image manifest, downloads the required layers, and verifies the digest automatically.

**Expected output:**
```
musl: Pulling from library/busybox
5bfa213ad291: Pull complete
Digest: sha256:19b646668802469d968a05342a601e78da4322a414a7c09b1c9ee25165042138
Status: Downloaded newer image for busybox:musl
docker.io/library/busybox:musl
```

Key indicators of a successful pull:

* Each layer shows `Pull complete`
* A `Digest` SHA256 hash is printed
* `Status: Downloaded newer image for busybox:musl` is confirmed
* The fully qualified image reference `docker.io/library/busybox:musl` is displayed

> **[SCREENSHOT-04]**
> *Terminal output of `sudo docker pull busybox:musl` showing layer download, SHA256 digest `sha256:19b6466...`, and `Status: Downloaded newer image` confirmation.*

---

### Phase 4: Re-tag the Image

Create the new tag `busybox:blog` pointing to the same image as `busybox:musl`. This operation does not copy or duplicate image data. It creates a new tag reference to the same underlying image layers.

```bash
sudo docker tag busybox:musl busybox:blog
```

A successful tag operation produces no terminal output. The absence of an error message confirms the tag was created successfully.

> **[SCREENSHOT-05]**
> *Terminal showing the `docker tag` command executed with no error output, confirming silent success behavior.*

---

### Phase 5: Validate the Operation

List all local `busybox` images to confirm both tags exist and share an identical Image ID, which proves they reference the same underlying image.

```bash
sudo docker images busybox
```

**Expected output:**
```
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
busybox      blog      0188a8de47ca   17 months ago   1.51MB
busybox      musl      0188a8de47ca   17 months ago   1.51MB
```

**Acceptance criteria checklist:**

* [ ] `busybox:musl` is present in the local image store
* [ ] `busybox:blog` is present in the local image store
* [ ] Both tags display the **same IMAGE ID** (`0188a8de47ca`)
* [ ] Both tags reflect the **same SIZE** (`1.51MB`)

> **[SCREENSHOT-06]**
> *Terminal output of `sudo docker images busybox` showing both `busybox:blog` and `busybox:musl` entries with identical IMAGE ID `0188a8de47ca` and SIZE `1.51MB`, confirming a successful re-tag.*

---

## Verification and Acceptance Criteria

The following table summarizes the final verification state upon task completion:

| Verification Check | Expected Value | Observed Value | Status |
|---|---|---|---|
| Target hostname | `stapp03` | `stapp03` | PASS |
| Logged-in user | `banner` | `banner` | PASS |
| Docker service state | `active (running)` | `active (running)` | PASS |
| Image `busybox:musl` present | Yes | Yes | PASS |
| Image `busybox:blog` present | Yes | Yes | PASS |
| IMAGE ID match across tags | Identical hash | `0188a8de47ca` | PASS |
| Pull digest | SHA256 present | `sha256:19b6466...042138` | PASS |

---

## Best Practices

### SSH and Host Key Management

* Always verify host fingerprints on first connection. Do not use `StrictHostKeyChecking=no` in production as it bypasses man-in-the-middle protection.
* Use SSH config files (`~/.ssh/config`) to manage multi-hop bastion access cleanly rather than chaining manual SSH commands.
* Prefer SSH key-pair authentication over password authentication for all server access, particularly in automated pipelines.

### Docker Service Verification

* Always verify Docker daemon health before running image operations. A stopped or degraded daemon will produce misleading errors or fail silently.
* Use `sudo systemctl is-active docker` in scripts for a clean programmatic check rather than parsing verbose `status` output.

### Image Tagging Strategy

* Treat image tags as lightweight pointers, not copies. Multiple tags sharing the same IMAGE ID consume no additional disk space.
* Follow a consistent tagging convention across environments (for example `:<environment>`, `:<semver>`, `:<git-sha>`) to support traceability and safe rollback.
* Avoid using `latest` as a production tag. Explicit version or purpose-based tags such as `busybox:blog` provide operational clarity in multi-environment deployments.

### Digest Pinning for Security

* When pulling images for production use, pin to the SHA256 digest rather than a floating tag to guarantee immutability:
  ```bash
  sudo docker pull busybox@sha256:19b646668802469d968a05342a601e78da4322a414a7c09b1c9ee25165042138
  ```
* Floating tags (for example `musl`, `latest`) can be silently updated by the upstream publisher, introducing unverified changes into your environment.

### Least Privilege Enforcement

* Limit `sudo docker` access to only the users and service accounts that require it. Membership in the `docker` group grants effective root access on the host.
* In production environments, evaluate rootless Docker or Podman to eliminate the privileged daemon attack surface entirely.

### Audit and Logging

* Record the image digest for every pull operation in your change log or ticketing system. This provides a cryptographic audit trail for compliance and incident response.
* Enable Docker daemon logging with a centralized log driver such as `journald`, `fluentd`, or `awslogs` to capture all image operations outside the host lifecycle.

---

## Lessons Learned

**1. Host Key Verification Must Be Handled Proactively**
The first SSH connection to `stapp03` triggered an interactive host authenticity prompt. In automated pipelines, unhandled interactive prompts cause silent failures. Pre-populate `~/.ssh/known_hosts` using `ssh-keyscan` during environment provisioning to eliminate this friction point before pipelines run.

**2. Pre-flight Service Health Checks Prevent Wasted Effort**
Confirming Docker was in an `active (running)` state before executing pull operations eliminated an entire failure class from the outset. Incorporate daemon health checks as a mandatory first step in all container-related runbooks and automation scripts.

**3. Silent Success Is Valid Unix Behavior in Tagging Operations**
The `docker tag` command produces no output on success. Teams unfamiliar with Unix conventions may interpret silence as an error. The correct validation pattern is always to follow a silent command with an explicit verification step such as `docker images` to confirm the expected state.

**4. IMAGE ID Equality Is the Definitive Confirmation for Re-tag Operations**
Verifying that both `busybox:musl` and `busybox:blog` shared the identical IMAGE ID (`0188a8de47ca`) was the definitive confirmation that the re-tag was correct and not a second independent image pull. Always use IMAGE ID equality as the acceptance criterion for re-tag operations, not just tag name presence.

**5. Hostname and User Validation After Every SSH Hop Is Non-Negotiable**
In multi-hop infrastructure, it is operationally easy to execute commands on the wrong host. Running `hostname` and `whoami` immediately after each SSH connection is a simple, zero-cost guardrail that should be standardized in every infrastructure runbook.

**6. Bastion Architecture Enforces Defense-in-Depth**
The requirement to route through `jump-host` to reach `stapp03` reflects a layered security architecture. Application servers are not directly internet-exposed, which reduces the attack surface and centralizes access logging. The Jump Host itself must be hardened, access-logged, and subject to the same patching cadence as the servers it protects.

---

## Reference Commands

```bash
# Connect to the target application server via Jump Host
ssh banner@stapp03

# Confirm host identity immediately after connection
hostname && whoami

# Check Docker daemon status
sudo systemctl status docker

# Start Docker daemon if not running
sudo systemctl start docker

# Pull image from Docker Hub
sudo docker pull busybox:musl

# Re-tag the pulled image with a new tag
sudo docker tag busybox:musl busybox:blog

# Verify both tags exist and share the same IMAGE ID
sudo docker images busybox

# Production-grade: Pin pull to SHA256 digest for immutability
sudo docker pull busybox@sha256:19b646668802469d968a05342a601e78da4322a414a7c09b1c9ee25165042138

# Inspect full image metadata for both tags
sudo docker inspect busybox:blog
sudo docker inspect busybox:musl
```

---





<img width="1026" height="860" alt="Image" src="https://github.com/user-attachments/assets/334ab182-7fb1-47d2-9e34-e59cc1a49886" />

<img width="1036" height="426" alt="Image" src="https://github.com/user-attachments/assets/828a5ccf-84a2-44c7-a595-e686b49d1f6c" />
