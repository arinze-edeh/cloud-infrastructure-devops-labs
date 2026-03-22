# Docker Compose: Containerized Apache HTTPD Deployment with Bind Mount Volume

[![Docker](https://img.shields.io/badge/Docker-26.1.3-2496ED?style=flat-square&logo=docker)](https://www.docker.com/)
[![Docker Compose](https://img.shields.io/badge/Docker%20Compose-v5.0.2-2496ED?style=flat-square&logo=docker)](https://docs.docker.com/compose/)
[![Apache HTTPD](https://img.shields.io/badge/Apache-httpd%3Alatest-D22128?style=flat-square&logo=apache)](https://hub.docker.com/_/httpd)
[![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20RHEL--based-FCC624?style=flat-square&logo=linux)](https://www.linux.org/)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=flat-square)]()

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Infrastructure Overview](#infrastructure-overview)
- [Prerequisites](#prerequisites)
- [Architecture Diagram](#architecture-diagram)
- [Solution Walkthrough](#solution-walkthrough)
  - [Step 1: SSH Access and Privilege Escalation](#step-1-ssh-access-and-privilege-escalation)
  - [Step 2: Verify Docker Environment](#step-2-verify-docker-environment)
  - [Step 3: Inspect Target Directories](#step-3-inspect-target-directories)
  - [Step 4: Author the Docker Compose File](#step-4-author-the-docker-compose-file)
  - [Step 5: Validate the Compose Configuration](#step-5-validate-the-compose-configuration)
  - [Step 6: Pull the HTTPD Image](#step-6-pull-the-httpd-image)
  - [Step 7: Launch the Container Stack](#step-7-launch-the-container-stack)
  - [Step 8: Verify Running Container](#step-8-verify-running-container)
  - [Step 9: Inspect Container Configuration](#step-9-inspect-container-configuration)
  - [Step 10: Validate Port Bindings and Volume Mounts](#step-10-validate-port-bindings-and-volume-mounts)
  - [Step 11: End-to-End HTTP Connectivity Test](#step-11-end-to-end-http-connectivity-test)
- [Configuration Reference](#configuration-reference)
- [Verification Checklist](#verification-checklist)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)

---

## Problem Statement

The Nautilus application development team requires static website content to be served via the **Apache HTTPD** web server on a containerized platform. The DevOps team was tasked with provisioning this environment on **App Server 2 (stapp02)** within the Stratos DC infrastructure, fulfilling the following requirements:

| Requirement | Specification |
|---|---|
| Compose File Location | `/opt/docker/docker-compose.yml` |
| Container Image | `httpd:latest` |
| Container Name | `httpd` |
| Host Port | `8089` |
| Container Port | `80` |
| Host Volume Path | `/opt/itadmin` |
| Container Volume Path | `/usr/local/apache2/htdocs` |

> **Constraint:** No data within `/opt/itadmin` or `/usr/local/apache2/htdocs` must be modified during provisioning.

---

## Infrastructure Overview

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| Application Server 2 | stapp02 | steve | Target deployment host |
| Jump Host | jump-host | thor | Secure entry point into Stratos DC |

> All access to Stratos DC application servers is routed through the **jump-host** for security compliance.

---

## Prerequisites

Ensure the following are available before executing this runbook:

- SSH access to `jump-host` as user `thor`
- SSH access from jump-host to `stapp02` as user `steve`
- `sudo` privileges for `steve` on `stapp02`
- Docker Engine installed and the `docker` daemon in `active (running)` state
- Docker Compose plugin (`docker compose`) available
- Outbound internet access from `stapp02` to `registry-1.docker.io` (for image pull)
- Host directories `/opt/docker` and `/opt/itadmin` already present on `stapp02`

---

## Architecture Diagram

```
+-----------------+        SSH         +------------------+       SSH        +------------------+
|   Operator      |  ------------->    |   Jump Host      |  ------------>   |  App Server 2    |
|   Workstation   |                    |  (jump-host)     |                  |  (stapp02)       |
+-----------------+                    |   User: thor     |                  |  User: steve     |
                                       +------------------+                  +--------+---------+
                                                                                      |
                                                                        sudo su -     |
                                                                                      v
                                                                             +--------+---------+
                                                                             |      root        |
                                                                             |                  |
                                                                             |  /opt/docker/    |
                                                                             |  docker-compose  |
                                                                             |      .yml        |
                                                                             +--------+---------+
                                                                                      |
                                                                          docker compose up -d
                                                                                      |
                                                                                      v
                                                                    +-----------------+----------+
                                                                    |    Docker Engine           |
                                                                    |                            |
                                                                    |  Container: httpd          |
                                                                    |  Image: httpd:latest       |
                                                                    |  Port: 0.0.0.0:8089->80   |
                                                                    |  Volume:                   |
                                                                    |   /opt/itadmin ->          |
                                                                    |   /usr/local/apache2/      |
                                                                    |   htdocs (bind, rw)        |
                                                                    +----------------------------+
```

---

## Solution Walkthrough

### Step 1: SSH Access and Privilege Escalation

Establish a secure shell connection from the **jump-host** to **App Server 2**, then escalate to `root` to perform administrative operations.

```bash
# From the jump-host terminal
ssh steve@stapp02

# Escalate to root
sudo su -

# Confirm root identity
whoami
```

**Expected Output:**

```
root
```

> **Note:** The first SSH connection to `stapp02` from the jump-host will prompt for host key verification. Confirm with `yes` to permanently add the host fingerprint to `~/.ssh/known_hosts`.

---

**SCREENSHOT:** 

<img width="1033" height="560" alt="image" src="https://github.com/user-attachments/assets/87710ba2-9b6a-4ac6-b0da-95d73775a968" />

>Terminal showing successful SSH connection to stapp02 and root escalation via `sudo su -`

---

### Step 2: Verify Docker Environment

Confirm that the Docker Engine and Docker Compose plugin are installed and operational before proceeding.

```bash
# Confirm Docker version
docker --version

# Confirm Docker daemon is active
systemctl status docker

# Confirm Docker Compose plugin version
docker compose version
```

**Expected Output:**

```
Docker version 26.1.3, build b72abbb
...
Active: active (running) since ...
...
Docker Compose version v5.0.2
```

> **Critical Check:** The Docker daemon must be in `active (running)` state. If the service is inactive, start it with `systemctl start docker` and enable it on boot with `systemctl enable docker`.

---

**SCREENSHOT:** 

<img width="1034" height="615" alt="image" src="https://github.com/user-attachments/assets/d16cca94-afa0-441e-b043-e5e2a5e9073d" />


>Terminal output showing `docker --version`, `systemctl status docker` showing active state, and `docker compose version`

---

### Step 3: Inspect Target Directories

Verify that the required host directories exist and are accessible before authoring the Compose file.

```bash
# Verify the Compose file directory
ls -ld /opt/docker

# Verify the web content volume directory
ls -ld /opt/itadmin
```

**Expected Output:**

```
drwxr-xr-x 2 root root 4096 Mar 22 03:27 /opt/docker
drwxr-xr-x 2 root root 4096 Mar 22 03:27 /opt/itadmin
```

> **Important:** Both directories must already exist on the host. The bind mount will fail at container startup if the source path (`/opt/itadmin`) is absent. Do **not** modify any content inside `/opt/itadmin`.

---

**SCREENSHOT:** 

<img width="1033" height="351" alt="image" src="https://github.com/user-attachments/assets/7d471edf-86b0-4355-ab50-194347c8f461" />

>Terminal output of `ls -ld /opt/docker` and `ls -ld /opt/itadmin` confirming both directories exist

---

### Step 4: Author the Docker Compose File

Navigate to `/opt/docker` and create the `docker-compose.yml` file with the exact specifications provided by the development team.

```bash
cd /opt/docker

cat > /opt/docker/docker-compose.yml << 'EOF'
version: '3'
services:
  web:
    image: httpd:latest
    container_name: httpd
    ports:
      - "8089:80"
    volumes:
      - /opt/itadmin:/usr/local/apache2/htdocs
EOF
```

Confirm the file was written correctly:

```bash
cat /opt/docker/docker-compose.yml
```

**Expected File Content:**

```yaml
version: '3'
services:
  web:
    image: httpd:latest
    container_name: httpd
    ports:
      - "8089:80"
    volumes:
      - /opt/itadmin:/usr/local/apache2/htdocs
```

**Compose Key Fields Explained:**

| Field | Value | Purpose |
|---|---|---|
| `image` | `httpd:latest` | Pulls the official Apache HTTPD image from Docker Hub |
| `container_name` | `httpd` | Assigns an explicit, deterministic name to the container |
| `ports` | `"8089:80"` | Maps host port 8089 to container port 80 (HTTP) |
| `volumes` | `/opt/itadmin:/usr/local/apache2/htdocs` | Bind mounts host web content directory into the container |

> **Note on `version: '3'`:** Docker Compose v2 and later treats the `version` key as obsolete and will emit a `WARN` at runtime. This is cosmetic only and does not affect functionality. It is safe to remove the `version` key in newer deployments.

---

**SCREENSHOT:** 

<img width="1030" height="683" alt="image" src="https://github.com/user-attachments/assets/01edc9cc-a6e5-4da7-a691-24ccf81746ff" />

>Terminal showing `cat /opt/docker/docker-compose.yml` with full YAML content displayed correctly

---

### Step 5: Validate the Compose Configuration

Before launching the stack, validate that the Compose file is syntactically and semantically correct using the built-in `config` subcommand. This resolves all relative paths, expands defaults, and surfaces any configuration errors.

```bash
docker compose -f /opt/docker/docker-compose.yml config
```

**Expected Output (normalized config):**

```yaml
name: docker
services:
  web:
    container_name: httpd
    image: httpd:latest
    networks:
      default: null
    ports:
      - mode: ingress
        target: 80
        published: "8089"
        protocol: tcp
    volumes:
      - type: bind
        source: /opt/itadmin
        target: /usr/local/apache2/htdocs
        bind: {}
networks:
  default:
    name: docker_default
```

> **Pro Tip:** `docker compose config` is a mandatory pre-flight check in any CI/CD pipeline. A clean output with no `ERROR` lines guarantees the file can be safely applied. The `WARN` on the `version` key is expected and non-blocking.

---

**SCREENSHOT:** 

<img width="1032" height="687" alt="image" src="https://github.com/user-attachments/assets/b97ac666-ea78-4f9d-b03e-1e16b31bc863" />

>Full terminal output of `docker compose -f /opt/docker/docker-compose.yml config` showing normalized configuration with no errors

---

### Step 6: Pull the HTTPD Image

Pre-pull the `httpd:latest` image to ensure it is present in the local Docker image cache before stack launch. This separates the network-dependent pull step from the container startup step, simplifying troubleshooting.

```bash
docker pull httpd:latest
```

**Expected Output:**

```
latest: Pulling from library/httpd
ec781dee3f47: Pull complete
09625190bc81: Pull complete
4f4fb700ef54: Pull complete
a1897530540b: Pull complete
753dca1c3a38: Pull complete
5184cf4f524b: Pull complete
Digest: sha256:331548c5249bdeced0f048bc2fb8c6b6427d2ec6508bed9c1fec6c57d0b27a60
Status: Downloaded newer image for httpd:latest
docker.io/library/httpd:latest
```

Confirm the image is locally available:

```bash
docker images | grep httpd
```

**Expected Output:**

```
httpd   latest   95e97c51ad04   5 days ago   117MB
```

---

**SCREENSHOT:** 

<img width="1038" height="292" alt="image" src="https://github.com/user-attachments/assets/7cb65b6d-6d46-4ffd-bc4e-4db4d79269e6" />

>Terminal output of `docker pull httpd:latest` showing all layers pulled and digest confirmed, followed by `docker images | grep httpd` confirming local availability]**

---

### Step 7: Launch the Container Stack

Deploy the containerized HTTPD service in detached mode using Docker Compose.

```bash
cd /opt/docker && docker compose up -d
```

**Expected Output:**

```
WARN[0000] /opt/docker/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
[+] up 2/2
 Network docker_default  Created  0.1s
 Container httpd         Created  0.2s
```

> **The `-d` flag (detached mode)** runs the container in the background and returns the terminal prompt immediately. This is the standard deployment posture for long-running services in production environments.

---

**SCREENSHOT:** 

<img width="1032" height="440" alt="image" src="https://github.com/user-attachments/assets/f193e882-605e-4a82-82eb-21f95860f19c" />

>Terminal output of `docker compose up -d` showing the network and container creation events with success checkmarks]**

---

### Step 8: Verify Running Container

Confirm the container is in a healthy `Up` state and the port mapping is active.

```bash
docker ps
```

**Expected Output:**

```
CONTAINER ID   IMAGE          COMMAND              CREATED         STATUS         PORTS                                   NAMES
d00b6dfa43b6   httpd:latest   "httpd-foreground"   2 minutes ago   Up 2 minutes   0.0.0.0:8089->80/tcp, :::8089->80/tcp   httpd
```

**Key Fields to Verify:**

| Field | Expected Value | Meaning |
|---|---|---|
| `IMAGE` | `httpd:latest` | Correct image is in use |
| `STATUS` | `Up X minutes` | Container is running without restarts |
| `PORTS` | `0.0.0.0:8089->80/tcp` | Host port 8089 bound to container port 80 on all interfaces |
| `NAMES` | `httpd` | Explicit container name matches specification |

---

**SCREENSHOT:** 

<img width="1015" height="177" alt="image" src="https://github.com/user-attachments/assets/8a3d8244-fc90-40c7-a81b-4380e3e4968f" />


>Terminal output of `docker ps` showing the httpd container in `Up` status with the correct port mapping `0.0.0.0:8089->80/tcp`

---

### Step 9: Inspect Container Configuration

Perform a deep inspection of the running container to validate all configuration parameters including network settings, volume mounts, port bindings, and image metadata.

```bash
docker inspect httpd
```

**Critical fields to verify in the JSON output:**

**Mounts Section:**

```json
"Mounts": [
    {
        "Type": "bind",
        "Source": "/opt/itadmin",
        "Destination": "/usr/local/apache2/htdocs",
        "Mode": "rw",
        "RW": true,
        "Propagation": "rprivate"
    }
]
```

**Port Bindings Section:**

```json
"PortBindings": {
    "80/tcp": [
        {
            "HostIp": "",
            "HostPort": "8089"
        }
    ]
}
```

> **What to look for:** Confirm `Type` is `bind`, `Source` is `/opt/itadmin`, `Destination` is `/usr/local/apache2/htdocs`, and `RW` is `true`. Any deviation here means the volume was not mounted as specified.

---

**SCREENSHOTS:**

<img width="1036" height="857" alt="image" src="https://github.com/user-attachments/assets/9ba022f1-b1d3-4603-91db-376af9a2354d" />
<img width="1032" height="862" alt="image" src="https://github.com/user-attachments/assets/ac60fbd4-096a-4a5d-82cd-6bf158f95689" />
<img width="1030" height="856" alt="image" src="https://github.com/user-attachments/assets/a9c70ed3-9924-42bd-baff-bba352c1a09f" />
<img width="1035" height="854" alt="image" src="https://github.com/user-attachments/assets/04e1175d-4626-432d-932c-ee5e103e16bb" />

>Terminal output of `docker inspect httpd` with the `Mounts` and `PortBindings` sections visible, confirming correct bind mount source, destination, and read-write mode

---

### Step 10: Validate Port Bindings and Volume Mounts

Use targeted commands to confirm the port binding and volume mount in isolation, providing clean audit-ready output.

```bash
# Verify port mappings
docker port httpd

# Verify volume mounts in structured JSON
docker inspect httpd --format='{{json .Mounts}}' | python3 -m json.tool
```

**Expected Output - Port Check:**

```
80/tcp -> 0.0.0.0:8089
80/tcp -> [::]:8089
```

**Expected Output - Volume Mount Check:**

```json
[
    {
        "Type": "bind",
        "Source": "/opt/itadmin",
        "Destination": "/usr/local/apache2/htdocs",
        "Mode": "rw",
        "RW": true,
        "Propagation": "rprivate"
    }
]
```

> **IPv4 and IPv6:** The port is bound on both `0.0.0.0:8089` (IPv4 all interfaces) and `:::8089` (IPv6 all interfaces), confirming the service is reachable from the host and the network.

---

**SCREENSHOT:** 


<img width="1035" height="510" alt="image" src="https://github.com/user-attachments/assets/1c1d1064-6ebe-4297-b4c8-a51ce2230574" />


>Terminal showing output of `docker port httpd` and the formatted JSON output of `docker inspect httpd --format='{{json .Mounts}}'` piped through `python3 -m json.tool`

---

### Step 11: End-to-End HTTP Connectivity Test

Perform a live HTTP request to the containerized Apache HTTPD instance to confirm the web server is serving content from the bind-mounted volume.

```bash
curl http://localhost:8089
```

**Expected Output:**

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
 <head>
  <title>Index of /</title>
 </head>
 <body>
<h1>Index of /</h1>
<ul><li><a href="index1.html"> index1.html</a></li>
</ul>
</body></html>
```

> **Result Interpretation:** The Apache directory listing confirms that the bind mount is functional. The file `index1.html` present in `/opt/itadmin` on the host is being served correctly by HTTPD through the container. The volume mount is live and read-write.

---

**SCREENSHOT:** 


<img width="1035" height="510" alt="image" src="https://github.com/user-attachments/assets/1c1d1064-6ebe-4297-b4c8-a51ce2230574" />


>Terminal showing `curl http://localhost:8089` returning the Apache HTTPD directory listing HTML with `index1.html` listed as a link, confirming end-to-end connectivity and volume mount integrity

---

## Configuration Reference

### docker-compose.yml

```yaml
version: '3'
services:
  web:
    image: httpd:latest
    container_name: httpd
    ports:
      - "8089:80"
    volumes:
      - /opt/itadmin:/usr/local/apache2/htdocs
```

**File Location:** `/opt/docker/docker-compose.yml`

### Runtime Environment Variables (from `docker inspect`)

| Variable | Value |
|---|---|
| `HTTPD_PREFIX` | `/usr/local/apache2` |
| `HTTPD_VERSION` | `2.4.66` |

### Network

| Property | Value |
|---|---|
| Network Name | `docker_default` |
| Driver | `bridge` |
| Container IP | `172.17.0.2` |
| Gateway | `172.17.0.1` |

---

## Verification Checklist

Use this checklist after every deployment to confirm full compliance with the specification.

- [ ] Compose file exists at `/opt/docker/docker-compose.yml`
- [ ] `docker compose config` returns no `ERROR` lines
- [ ] `docker ps` shows container `httpd` in `Up` state
- [ ] `docker ps` shows `PORTS` as `0.0.0.0:8089->80/tcp`
- [ ] `docker port httpd` returns `80/tcp -> 0.0.0.0:8089`
- [ ] `docker inspect httpd` `Mounts[0].Source` is `/opt/itadmin`
- [ ] `docker inspect httpd` `Mounts[0].Destination` is `/usr/local/apache2/htdocs`
- [ ] `docker inspect httpd` `Mounts[0].RW` is `true`
- [ ] `curl http://localhost:8089` returns HTTP 200 with valid HTML
- [ ] No files were created, modified, or deleted inside `/opt/itadmin`

---

## Troubleshooting

### Container Fails to Start

```bash
# Inspect recent container logs
docker logs httpd

# Check for port conflicts on the host
ss -tlnp | grep 8089
```

**Common Cause:** Another process is already bound to port `8089`. Identify and stop it, or adjust the host port mapping in the Compose file.

### Volume Mount Not Reflecting Host Files

```bash
# Verify source directory exists and has correct permissions
ls -la /opt/itadmin

# Exec into container to verify bind mount target
docker exec -it httpd ls /usr/local/apache2/htdocs
```

**Common Cause:** The host source directory `/opt/itadmin` did not exist at container creation time. Docker creates the directory automatically in this case, but it will be empty. Always pre-create host volume directories.

### Image Pull Failure

```bash
# Test outbound connectivity to Docker Hub
curl -I https://registry-1.docker.io/v2/

# Verify DNS resolution
nslookup registry-1.docker.io
```

**Common Cause:** The host is behind a firewall or proxy that blocks outbound HTTPS to Docker Hub. Coordinate with the network team to whitelist `registry-1.docker.io` on port 443.

### `docker compose` Command Not Found

```bash
# Verify the plugin is installed
docker compose version

# If using legacy docker-compose binary
docker-compose version
```

**Common Cause:** The Docker Compose plugin (`compose`) is distinct from the legacy standalone binary (`docker-compose`). This runbook targets Docker Compose v2+ (`docker compose`).

### Container Exits Immediately

```bash
# Check exit code and logs
docker ps -a | grep httpd
docker logs httpd
```

**Common Cause:** The `httpd-foreground` entrypoint failed due to a misconfigured volume or missing configuration file inside the container. Review logs for `AH0` error codes.

---

## Best Practices

### Security

- **Least Privilege Access:** Always SSH as a non-root user (`steve`) and escalate to root only for the duration of the administrative task. Avoid configuring containers to run as root where the application supports it.
- **Read-Only Volumes:** If the web server only needs to read static content, scope the bind mount to read-only (`/opt/itadmin:/usr/local/apache2/htdocs:ro`) to prevent container processes from modifying host data.
- **Network Isolation:** Restrict published ports to specific host interfaces when the service should not be exposed on all interfaces (e.g., `"127.0.0.1:8089:80"` for localhost-only access).
- **Image Pinning:** For production workloads, pin the image to a specific digest or semantic version tag rather than `latest` to ensure reproducible deployments and avoid unintended upstream changes.

### Operational Reliability

- **Always Pre-Create Volume Directories:** Create host bind mount source directories before running `docker compose up`. If Docker creates them automatically (when they do not exist), the resulting directories may have incorrect ownership (`root:root`) and may be empty, causing the application to serve no content.
- **Pre-Pull Images Separately:** Use `docker pull` as a distinct step before `docker compose up` in pipelines. This isolates network failures from application startup failures, making root-cause analysis faster.
- **Use `docker compose config` as a Pre-Flight Gate:** Integrate `docker compose config` as a mandatory CI gate before any `docker compose up` execution to catch syntax and schema errors early.
- **Explicit Container Names:** Always set `container_name` in the Compose file. Without it, Docker generates a name based on the project and service name, which can change across environments and complicate log aggregation and monitoring.

### Maintainability

- **Remove Obsolete Fields:** The `version` top-level key in Compose files is deprecated in Docker Compose v2. Remove it from new files to eliminate the warning and reduce confusion for other engineers reading the file.
- **Store Compose Files in Version Control:** The `/opt/docker/docker-compose.yml` file should be tracked in a Git repository with meaningful commit messages, enabling change auditing and rollback.
- **Document Port Assignments:** Maintain a central port registry for the environment to prevent port conflicts as the number of containerized services grows.

---

## Lessons Learned

### 1. The `version` Key Deprecation Warning is Non-Blocking

Docker Compose v2 emits `WARN: the attribute 'version' is obsolete` when it encounters the top-level `version` key. This is a warning, not an error. The stack deploys successfully. However, in environments where pipelines fail on any warning output, this can cause false negatives. **Action:** Remove the `version` key from all new Compose files authored against Docker Compose v2+.

### 2. `docker compose config` is an Indispensable Validation Tool

Running `docker compose config` before `docker compose up` saved significant debugging time. The command normalizes and resolves the full configuration, revealing whether port mappings, volume paths, and network names are interpreted as intended. It is the difference between discovering a misconfiguration before and after the container starts. **Action:** Mandate `docker compose config` as the first step in any Compose-based deployment procedure.

### 3. Bind Mount Source Directories Must Pre-Exist

Docker Engine will automatically create a missing host directory when a bind mount is defined in a Compose file, but the created directory will be owned by `root` and will be empty. This causes the web server to serve no content and creates subtle permission issues for non-root processes. **Action:** Always verify host volume directories exist with `ls -ld` before container startup. Pre-create them with the correct ownership if they do not exist.

### 4. Separate Image Pull from Stack Startup in Automated Pipelines

Combining `docker pull` and `docker compose up` into a single step makes it harder to distinguish network failures (image pull) from runtime failures (container startup). Pre-pulling the image as an isolated step provides a clean boundary. **Action:** In CI/CD pipelines, structure Docker deployments as: validate, pull, deploy, verify. Each step should be independently idempotent.

### 5. Always Perform an End-to-End Connectivity Test

`docker ps` showing `Up` does not mean the application is healthy. The process inside the container may have started and then encountered a runtime error. A `curl` to the exposed port is the only confirmation that the full stack (container runtime, networking, web server process, and volume mount) is functioning end to end. **Action:** Always conclude a container deployment procedure with an HTTP-level smoke test against the published port.

### 6. `docker inspect` is the Source of Truth for Runtime Configuration

While the Compose file defines intent, `docker inspect` shows what Docker actually applied at runtime. Discrepancies between the two (such as a volume mount silently falling back to a named volume instead of a bind mount) are only visible in `docker inspect` output. **Action:** Incorporate `docker inspect` verification of `Mounts` and `PortBindings` into the post-deployment checklist for every containerized service.

### 7. Privilege Escalation Scope Should Be Minimized

All Docker operations in this runbook required `root` because the target directories (`/opt/docker`, `/opt/itadmin`) are owned by root and the Docker socket is accessible only to the `docker` group or root. **Action:** As a longer-term hardening measure, add the service account (`steve`) to the `docker` group (`usermod -aG docker steve`) and set correct directory ownership, eliminating the need for `sudo su -` for routine Docker operations.

---
