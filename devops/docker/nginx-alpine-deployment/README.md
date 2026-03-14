# Containerized Nginx Deployment on Alpine Linux

> **Category:** Docker / Container Lifecycle Management
> **Difficulty:** Foundational
> **Environment:** Nautilus DevOps Lab
> **Status:** Resolved

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Environment Overview](#environment-overview)
- [Prerequisites](#prerequisites)
- [Resolution](#resolution)
  - [Phase 1: Establish Remote Connection](#phase-1-establish-remote-connection)
  - [Phase 2: Pull the Target Image](#phase-2-pull-the-target-image)
  - [Phase 3: Run the Container](#phase-3-run-the-container)
  - [Phase 4: Verify Container State](#phase-4-verify-container-state)
- [Validation Criteria](#validation-criteria)
- [Key Concepts](#key-concepts)

---

## Problem Statement

The Nautilus DevOps team required an `nginx` container deployment on **Application Server 1** as part of ongoing application deployment testing across selected infrastructure nodes.

**Objective:** Deploy a container named `nginx_1` using the `nginx:alpine` image on Application Server 1 and confirm it is in a `running` state.

---

## Environment Overview

| Role | Hostname | User | Purpose |
|---|---|---|---|
| Jump Host | `jump-host` | `thor` | Secure gateway into the Stork DC |
| Application Server 1 | `stapp01` | `tony` | Target deployment node |

> **Note:** All connections to application servers are routed through the Jump Host. Direct external access to application servers is not available.

---

## Prerequisites

- Active session on the Jump Host as user `thor`
- SSH access to `stapp01` with valid credentials
- Docker Engine installed and running on `stapp01`
- Outbound internet access from `stapp01` to pull from Docker Hub

---

## Resolution

### Phase 1: Establish Remote Connection

From the Jump Host, initiate an SSH session into Application Server 1.

```bash
ssh tony@stapp01
```

When prompted to confirm the host fingerprint, type `yes` to permanently add the host to the known hosts list.

```
The authenticity of host 'stapp01 (10.244.81.22)' can't be established.
ED25519 key fingerprint is SHA256:uBFlEewDvpAFOZL1+OG9IGer5Bwrfq5jJNbL1VdXYjE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp01' (ED25519) to the list of known hosts.
```

Enter the password when prompted:

```
tony@stapp01's password:
```

**Confirm your shell prompt reflects the correct host before proceeding:**

```
[tony@stapp01 ~]$
```

> **Screenshot: Terminal showing successful SSH login with prompt [tony@stapp01 ~]$**


---

### Phase 2: Pull the Target Image

Pull the `nginx:alpine` image from Docker Hub using elevated privileges.

```bash
sudo docker pull nginx:alpine
```

Enter the sudo password when prompted:

```
[sudo] password for tony:
```

**Expected output confirming successful image pull:**

```
alpine: Pulling from library/nginx
589002ba0eae: Pull complete
d2a46166eee6: Pull complete
593488f95c35: Pull complete
e19aff8f2cce: Pull complete
1549d7aec962: Pull complete
1f25242adbdb: Pull complete
c32126d2b96c: Pull complete
c24026275c33: Pull complete
Digest: sha256:f46cb72c7df02710e693e863a983ac42f6a9579058a59a35f1ae36c9958e4ce0
Status: Downloaded newer image for nginx:alpine
docker.io/library/nginx:alpine
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing all image layers pulled with Status: Downloaded newer image for nginx:alpine ]`

---

### Phase 3: Run the Container

Create and start the container in detached mode using the exact name specified in the task requirements.

```bash
sudo docker run -d --name nginx_1 nginx:alpine
```

**Flag Reference:**

| Flag | Description |
|---|---|
| `run` | Create and start a new container |
| `-d` | Detached mode; runs container in the background |
| `--name nginx_1` | Assigns the required container name |
| `nginx:alpine` | Specifies the image and tag to use |

**Expected output:**

```
b2712254a7e63d80f11ea33ad2aa45a364894ec75078d9b028e2f3b30efebf0e
```

The returned string is the full container ID confirming the container was created and started successfully.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing the docker run command with the returned container ID ]`

---

### Phase 4: Verify Container State

Confirm the container is actively running before submitting for validation.

```bash
sudo docker ps
```

**Expected output:**

```
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS     NAMES
b2712254a7e6   nginx:alpine   "/docker-entrypoint..."   47 seconds ago   Up 46 seconds   80/tcp    nginx_1
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing docker ps output with nginx_1 listed as Up and running ]`

---

## Validation Criteria

All of the following must be true before the task is marked complete:

| Check | Expected Value | Result |
|---|---|---|
| Container Name | `nginx_1` | Passed |
| Image | `nginx:alpine` | Passed |
| Status | `Up` (running) | Passed |
| Exposed Port | `80/tcp` | Passed |
| Target Host | `stapp01` | Passed |

---

## Key Concepts

**Why `-d` (detached mode)?**
Running without `-d` ties the container process to your terminal session. If the session ends, the container stops. Detached mode ensures the container continues running independently, satisfying the `running` state requirement.

**Why `nginx:alpine`?**
The `alpine` tag refers to an image built on Alpine Linux, a minimal base image. It results in a significantly smaller container footprint compared to the default `nginx:latest` image, making it preferred for lightweight deployments and testing environments.

**Why SSH through the Jump Host?**
The Jump Host acts as a hardened bastion node providing the only authorized entry point into the Stork DC network. Application servers are not directly internet-accessible, enforcing network segmentation and access control.

---


