# Container Runtime Provisioning on CentOS Stream 9

![Docker](https://img.shields.io/badge/Docker-29.3.0-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker_Compose-v5.1.0-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![CentOS](https://img.shields.io/badge/CentOS_Stream_9-262577?style=for-the-badge&logo=centos&logoColor=white)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure Details](#infrastructure-details)
- [Prerequisites](#prerequisites)
- [Resolution](#resolution)
  - [Phase 1: Access and Connection](#phase-1-access-and-connection)
  - [Phase 2: System Preparation](#phase-2-system-preparation)
  - [Phase 3: Docker Engine Installation](#phase-3-docker-engine-installation)
  - [Phase 4: Service Initiation and Validation](#phase-4-service-initiation-and-validation)
- [Known Issues and Resolutions](#known-issues-and-resolutions)
- [Validation Summary](#validation-summary)
- [Repository Structure](#repository-structure)

---

## Overview

This document details the end-to-end provisioning of the Docker container runtime on **App Server 3 (stapp03)** within the Nautilus DevOps Stratos datacenter environment. The scope covers installing `docker-ce`, `docker-compose-plugin`, and initiating the Docker daemon as a managed systemd service on a CentOS Stream 9 host.

This runbook serves as the authoritative reference for this task and is intended to be reproducible across equivalent lab and staging environments.

---

## Problem Statement

The Nautilus DevOps team is containerizing applications across the Stratos datacenter fleet following an alignment meeting with the application development team. As an initial provisioning step, the following actions were required on **App Server 3**:

1. Install the `docker-ce` and `docker-compose` packages.
2. Initiate the `docker` service and ensure it persists across reboots.

**Target Host:** `stapp03` (Nautilus App 3)
**Access Path:** Jump Host (`jump-host`) to App Server 3 via SSH

---

## Infrastructure Details

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| `stapp01` | `stapp01.stratos.xfusioncorp.com` | `tony` | Nautilus App 1 |
| `stapp02` | `stapp02.stratos.xfusioncorp.com` | `steve` | Nautilus App 2 |
| **`stapp03`** | `stapp03.stratos.xfusioncorp.com` | **`banner`** | **Nautilus App 3 (Target)** |
| `stlb01` | `stlb01.stratos.xfusioncorp.com` | `loki` | Nautilus HTTP LBR |
| `stdb01` | `stdb01.stratos.xfusioncorp.com` | `peter` | Nautilus DB Server |
| `ststor01` | `ststor01.stratos.xfusioncorp.com` | `natasha` | Nautilus Storage Server |
| `stbkp01` | `stbkp01.stratos.xfusioncorp.com` | `clint` | Nautilus Backup Server |
| `stmail01` | `stmail01.stratos.xfusioncorp.com` | `groot` | Nautilus Mail Server |
| `jump_host` | `jump_host.stratos.xfusioncorp.com` | `thor` | Jump Server to Access Stork DC |
| `jenkins` | `jenkins.stratos.xfusioncorp.com` | `jenkins` | Jenkins Server for CI/CD |

---

## Prerequisites

Before executing this runbook, confirm the following:

- You have active access to the jump host as user `thor`
- `stapp03` is reachable via short hostname from the jump host
- The `banner` account has `sudo` privileges on `stapp03`
- Internet connectivity is available from `stapp03` to `download.docker.com`

---

## Resolution

### Phase 1: Access and Connection

> **Context:** The jump host uses `/etc/hosts` for internal name resolution within the Stratos datacenter. The fully qualified domain name (`stapp03.stratos.xfusioncorp.com`) is **not** resolvable via public DNS from the jump host. The short hostname `stapp03` must be used instead.

#### Step 1.1: Attempt FQDN Connection (Expected Failure)

```bash
ssh banner@stapp03.stratos.xfusioncorp.com
```

**Expected Output:**

```
ssh: Could not resolve hostname stapp03.stratos.xfusioncorp.com: Name or service not known
```

> **Root Cause:** The Stratos datacenter runs a Kubernetes-managed `/etc/hosts` file. Internal hosts are mapped by short name only. The FQDN is not registered in any reachable DNS resolver from the jump host.

---

#### Step 1.2: Connect Using Short Hostname

```bash
ssh banner@stapp03
```

When prompted, type `yes` to accept the host fingerprint, then enter the password.

**Password:** 

**Expected Output:**

```
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
banner@stapp03's password:
[banner@stapp03 ~]$
```

> **Screenshot**
<img width="1030" height="385" alt="image" src="https://github.com/user-attachments/assets/365fa231-86c1-44ce-89c0-18395e2e817a" />

> *Caption: Successful SSH session established to stapp03 via short hostname from the jump host.*

---

#### Step 1.3: Verify Host Resolution (Optional Diagnostic)

```bash
cat /etc/hosts
```

**Expected Output (Relevant Entries):**

```
10.244.244.244  stapp03

# Entries added by HostAliases.
10.0.15.5       docker-registry-mirror.kodekloud.com
```

> **Note:** The entry `docker-registry-mirror.kodekloud.com` confirms a pre-configured local Docker registry mirror is available at `10.0.15.5`. Image pulls may route through this mirror.

---

### Phase 2: System Preparation

> **Context:** Before adding external repositories, the system package index must be updated to the latest state to avoid dependency conflicts during the Docker installation.

#### Step 2.1: Update All System Packages

```bash
sudo yum update -y
```

Enter password `BigGr33n` when prompted.

**Expected Output (Summary):**

```
Upgrade  72 Packages
Total download size: 144 M
...
Complete!
```

> **Screenshots**
<img width="1033" height="861" alt="image" src="https://github.com/user-attachments/assets/9aeaecea-a40e-45de-bdc6-26460c33339b" />
<img width="1029" height="870" alt="image" src="https://github.com/user-attachments/assets/f82546b9-81d8-4d03-8b17-31651adc7fac" />
<img width="1030" height="855" alt="image" src="https://github.com/user-attachments/assets/ca2d4379-aa0d-4c53-b9ff-6d638c3a67f9" />

> *Caption: yum update completing successfully, showing 72 packages upgraded on CentOS Stream 9.*

---

#### Step 2.2: Install yum-utils

```bash
sudo yum install -y yum-utils
```

**Expected Output:**

```
Package yum-utils-4.3.0-26.el9.noarch is already installed.
Nothing to do.
Complete!
```

> `yum-utils` provides the `yum-config-manager` binary required to add the Docker repository in the next phase. In this case, it was already present from the system update.

---

### Phase 3: Docker Engine Installation

> **Context:** Docker does not ship in the default CentOS Stream 9 repositories. The official Docker CE repository must be added explicitly before packages can be installed.

#### Step 3.1: Add the Official Docker CE Repository

```bash
sudo yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

**Expected Output:**

```
Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
```

> **Screenshot**
<img width="1035" height="258" alt="image" src="https://github.com/user-attachments/assets/6ec77742-42bd-470b-b714-be6e322a2613" />

> *Caption: Docker CE repository successfully registered via yum-config-manager.*

---

#### Step 3.2: Install Docker CE, CLI, containerd, and Compose Plugin

```bash
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

**Packages Installed:**

| Package | Version | Source |
|---|---|---|
| `docker-ce` | `3:29.3.0-1.el9` | `docker-ce-stable` |
| `docker-ce-cli` | `1:29.3.0-1.el9` | `docker-ce-stable` |
| `containerd.io` | `2.2.2-1.el9` | `docker-ce-stable` |
| `docker-compose-plugin` | `5.1.0-1.el9` | `docker-ce-stable` |

**Total:** 22 packages installed (4 primary + 18 dependencies)

**Expected Final Line:**

```
Complete!
```

> **Screenshot Placeholder**
> `screenshots/phase3-docker-install-complete.png`
> *Caption: All 22 packages including docker-ce 29.3.0 and docker-compose-plugin 5.1.0 installed successfully.*

> **Why `docker-compose-plugin` instead of standalone `docker-compose`?**
> The plugin architecture integrates Compose directly into the Docker CLI (`docker compose` with a space, not a hyphen). It is installed and updated through the same `yum` repository as the engine, eliminating manual binary management and version drift.

---

### Phase 4: Service Initiation and Validation

> **Context:** Installing packages does not start the Docker daemon. The service must be explicitly started and enabled to ensure it runs now and survives future reboots.

#### Step 4.1: Start the Docker Service

```bash
sudo systemctl start docker
```

**Expected Output:** No output (silent success indicates the service started without error).

---

#### Step 4.2: Enable Docker to Start on Boot

```bash
sudo systemctl enable docker
```

**Expected Output:**

```
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service \
  -> /usr/lib/systemd/system/docker.service.
```

> **Screenshot Placeholder**
> `screenshots/phase4-docker-enable-symlink.png`
> *Caption: systemd symlink created confirming Docker is enabled for automatic startup at boot.*

---

#### Step 4.3: Verify Service Status

```bash
sudo systemctl status docker
```

**Expected Output (Key Fields):**

```
docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
   Active: active (running) since Fri 2026-03-13 03:12:07 UTC; 30s ago
  Main PID: 36647 (dockerd)
```

Press `q` to exit the status pager and return to the prompt.

> **Screenshot Placeholder**
> `screenshots/phase4-systemctl-status-active.png`
> *Caption: Docker service showing "active (running)" status with enabled boot persistence confirmed.*

> **Service Log Warnings (Non-Critical):**
>
> The following two warnings appear in the service log and are expected in this Kubernetes-managed environment:
>
> | Warning | Cause | Impact |
> |---|---|---|
> | `ip6tables is enabled, but cannot...` | IPv6 iptables not fully configured in the pod network | None |
> | `configuring DOCKER-USER error` | iptables chain does not pre-exist | None, Docker creates it automatically |

---

#### Step 4.4: Verify Installed Versions

```bash
docker --version && docker compose version
```

**Expected Output:**

```
Docker version 29.3.0, build 5927d80
Docker Compose version v5.1.0
```

> **Screenshot Placeholder**
> `screenshots/phase4-version-verification.png`
> *Caption: Final version verification confirming docker-ce 29.3.0 and Docker Compose v5.1.0 are operational.*

---

## Known Issues and Resolutions

### Issue 1: FQDN Not Resolvable from Jump Host

**Symptom:**
```
ssh: Could not resolve hostname stapp03.stratos.xfusioncorp.com: Name or service not known
```

**Root Cause:** The Stratos datacenter uses a Kubernetes-managed `/etc/hosts` file for internal host resolution. The FQDN is not registered in public DNS.

**Resolution:** Use the short hostname `stapp03` instead of the FQDN.

```bash
# INCORRECT
ssh banner@stapp03.stratos.xfusioncorp.com

# CORRECT
ssh banner@stapp03
```

---

### Issue 2: Docker Repository Not Available by Default

**Symptom:** `docker-ce` is not found in any configured repository.

**Root Cause:** CentOS Stream 9 default repositories do not include Docker CE.

**Resolution:** Register the official Docker CE repository before attempting installation.

```bash
sudo yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

---

### Issue 3: Docker Installed but Not Running

**Symptom:** `docker` binary exists but commands fail with a connection error to the daemon socket.

**Root Cause:** Package installation does not auto-start the systemd service.

**Resolution:** Explicitly start and enable the service.

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

---

## Validation Summary

| Phase | Action | Command | Result |
|---|---|---|---|
| 1 | SSH to stapp03 | `ssh banner@stapp03` | Connected as `banner` |
| 2 | System update | `sudo yum update -y` | 72 packages upgraded |
| 2 | yum-utils present | `sudo yum install -y yum-utils` | Confirmed installed |
| 3 | Docker repo added | `yum-config-manager --add-repo ...` | Repo registered |
| 3 | Docker packages installed | `sudo yum install docker-ce ...` | 22 packages installed |
| 4 | Service started | `sudo systemctl start docker` | Active |
| 4 | Service enabled | `sudo systemctl enable docker` | Symlink created |
| 4 | Status verified | `sudo systemctl status docker` | `active (running)` |
| 4 | Versions confirmed | `docker --version && docker compose version` | `29.3.0` / `v5.1.0` |

---

## Repository Structure

```
devops/
  docker/
    container-runtime-provisioning/
      README.md                         # This document
      screenshots/
        phase1-ssh-connection-success.png
        phase2-yum-update-complete.png
        phase3-docker-repo-added.png
        phase3-docker-install-complete.png
        phase4-docker-enable-symlink.png
        phase4-systemctl-status-active.png
        phase4-version-verification.png
```

---

## Environment Reference

| Item | Value |
|---|---|
| Target Host | `stapp03` / `10.244.244.244` |
| OS | CentOS Stream 9 |
| Docker CE Version | `29.3.0` |
| Docker Compose Version | `v5.1.0` |
| containerd Version | `2.2.2` |
| Access User | `banner` |
| Entry Point | `jump_host` as `thor` |
| Task Completed | Fri 2026-03-13 03:12:07 UTC |

---


<img width="1033" height="607" alt="image" src="https://github.com/user-attachments/assets/61af7d2b-e8ca-4300-8461-7d839a302805" />

<img width="1037" height="863" alt="image" src="https://github.com/user-attachments/assets/093a6fd7-0646-45ea-b9f1-87315c7df7a6" />

<img width="1036" height="857" alt="image" src="https://github.com/user-attachments/assets/6d5a9a81-ac0b-4412-a540-1647ef720adc" />
<img width="1040" height="859" alt="image" src="https://github.com/user-attachments/assets/6c893816-d29d-4894-a295-7c773d594756" />
<img width="1030" height="870" alt="image" src="https://github.com/user-attachments/assets/e62d1862-f343-41ab-b287-25da9677068a" />
<img width="1035" height="559" alt="image" src="https://github.com/user-attachments/assets/2fe46f04-6003-42d2-a322-f34fca611852" />
<img width="1035" height="559" alt="image" src="https://github.com/user-attachments/assets/9a0a38be-8d30-42c8-9127-305e8e671322" />
<img width="1032" height="632" alt="image" src="https://github.com/user-attachments/assets/10d8500c-3398-4c93-8301-bbbe5ecccba5" />
