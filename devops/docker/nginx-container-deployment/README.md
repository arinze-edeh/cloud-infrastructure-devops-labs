# Docker Nginx Deployment on Application Server 1
## Nautilus DevOps Infrastructure | Stratos Datacenter

---

> **Category:** Container Orchestration | **Severity:** Standard | **Environment:** Production-Like Lab
> **Platform:** Docker 26.1.3 | **OS:** Linux (stapp01) | **Author:** Tony (App Server 1 Admin)

---

## Table of Contents

* [Problem Statement](#problem-statement)
* [Environment Overview](#environment-overview)
* [Prerequisites](#prerequisites)
* [Resolution Walkthrough](#resolution-walkthrough)
* [Verification and Validation](#verification-and-validation)
* [Architecture Diagram](#architecture-diagram)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Problem Statement

The Nautilus DevOps team required a containerized nginx web server to be provisioned on **Application Server 1 (stapp01)** within the **Stratos Datacenter** to support a planned e-commerce application hosting initiative.

The following acceptance criteria were defined:

* Pull the `nginx:stable` Docker image on Application Server 1
* Create and name the container `ecommerce`
* Bind **host port `8084`** to **container port `80`**
* Ensure the container remains in a **running state** post-deployment

---

## Environment Overview

| Property | Value |
|---|---|
| **Host** | stapp01 |
| **IP Address** | 10.244.49.46 |
| **Operating User** | tony |
| **Docker Version** | 26.1.3, build b72abbb |
| **Target Image** | nginx:stable |
| **Container Name** | ecommerce |
| **Host Port** | 8084 |
| **Container Port** | 80 |
| **Docker Service Status** | active (running) |

---

## Prerequisites

Before executing the deployment steps, confirm the following are satisfied:

* SSH access to `stapp01` via jump host is established
* The user `tony` has `sudo` privileges on `stapp01`
* Docker service (`docker.service`) is active and enabled via `systemd`
* Outbound internet access is available on `stapp01` to pull from Docker Hub

---

## Resolution Walkthrough

### Step 1: Establish SSH Session to Application Server 1

Connect from the jump host to `stapp01` using the assigned user credentials.

```bash
ssh tony@stapp01
```

**Expected Output:**

```
The authenticity of host 'stapp01 (10.244.49.46)' can't be established.
ED25519 key fingerprint is SHA256:Yf1I1eVYqdb0+CKs1Mml7VPPw57SXFAX7ydbClHF4Ho.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp01' (ED25519) to the list of known hosts.
tony@stapp01's password:
```

> **Note:** On first connection, SSH will prompt for host key verification. Type `yes` to trust and permanently add the host fingerprint to `~/.ssh/known_hosts`.

**SCREENSHOT:** 

<img width="1025" height="372" alt="image" src="https://github.com/user-attachments/assets/3312fe02-d605-4f48-a48d-f0b56d05bc1d" />

>SSH login to stapp01 from jump host showing successful authentication

---

### Step 2: Confirm Hostname and Docker Installation

Verify you are on the correct host and that Docker is installed and available.

```bash
hostname
docker --version
```

**Expected Output:**

```
stapp01
Docker version 26.1.3, build b72abbb
```

**SCREENSHOT:** 

<img width="1015" height="214" alt="image" src="https://github.com/user-attachments/assets/56343ee7-894a-4034-8a17-a821ad765da2" />


>Terminal output confirming hostname as stapp01 and Docker version 26.1.3

---

### Step 3: Verify Docker Service Health via systemd

Confirm that the Docker daemon is active and running before performing any container operations.

```bash
sudo systemctl status docker
```

**Expected Output (key indicators):**

```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: disabled)
     Active: active (running) since Sat 2026-03-21 04:00:24 UTC; 30min ago
   Main PID: 1382 (dockerd)
```

> **Key Indicators to Validate:**
> * `Active: active (running)` confirms the daemon is healthy
> * `enabled` in the `Loaded` line confirms the service auto-starts on reboot
> * `Main PID` should reference `dockerd`

**SCREENSHOT:** 

<img width="1031" height="857" alt="image" src="https://github.com/user-attachments/assets/738d9627-01ab-4326-8d65-6f24b95c0181" />

>`systemctl status docker` output showing active (running) state with green dot indicator

---

### Step 4: Pull the nginx:stable Docker Image

Pull the official `nginx:stable` image from Docker Hub onto the application server.

```bash
sudo docker pull nginx:stable
```

**Expected Output:**

```
stable: Pulling from library/nginx
ec781dee3f47: Pull complete
7ae953564ec1: Pull complete
e804dd183f84: Pull complete
2788c982dfb3: Pull complete
11c8966ecbb1: Pull complete
31fbdf624c67: Pull complete
21f82cc9dfd5: Pull complete
Digest: sha256:42e026ae5315aa0deec22fb00c364fc5ec8d9af1c4833ad5317e2a433e4de0df
Status: Downloaded newer image for nginx:stable
docker.io/library/nginx:stable
```

> **Why `nginx:stable`?** The `stable` tag tracks the latest LTS-equivalent nginx release, offering a balance of security patches and API stability compared to `latest` or `mainline` tags.

**SCREENSHOT:** 

<img width="1032" height="854" alt="image" src="https://github.com/user-attachments/assets/6522bad1-8e45-4f4f-bf23-1a48e97dc0f3" />

>docker pull nginx:stable command showing all layers pulled and digest confirmation

---

### Step 5: Confirm the Image Is Present Locally

After pulling, validate the image exists in the local Docker image registry.

```bash
sudo docker images | grep nginx
```

**Expected Output:**

```
nginx   stable   4fc974d655ce   4 days ago   161MB
```

**SCREENSHOT:** 

<img width="1033" height="487" alt="image" src="https://github.com/user-attachments/assets/c0e9feb5-a6b1-4e4a-9ab9-cad5ec131771" />

>docker images output filtered for nginx showing the stable tag, image ID, and size

---

### Step 6: Create and Run the ecommerce Container

Instantiate the container with the required name, port binding, and detached mode flag.

```bash
sudo docker run -d \
  --name ecommerce \
  -p 8084:80 \
  nginx:stable
```

**Flag Breakdown:**

| Flag | Purpose |
|---|---|
| `-d` | Run container in detached (background) mode |
| `--name ecommerce` | Assign the container a human-readable name |
| `-p 8084:80` | Bind host port 8084 to container port 80 (nginx default) |
| `nginx:stable` | The image to use for the container |

**Expected Output:**

```
574385ac4aa87e9521685bb783e492e778552804054004113facc8c7ca6b53d0
```

> The returned 64-character string is the full container ID, confirming successful creation.

**SCREENSHOT:** 

<img width="1034" height="394" alt="image" src="https://github.com/user-attachments/assets/46cdc338-53f4-445b-b4fa-1527cb43bd43" />

>docker run command with full container ID returned confirming successful container creation

---

### Step 7: Confirm Container Is in Running State

List all active containers and verify `ecommerce` is running with the correct port mapping.

```bash
sudo docker ps
```

**Expected Output:**

```
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                   NAMES
574385ac4aa8   nginx:stable   "/docker-entrypoint..."   36 seconds ago   Up 34 seconds   0.0.0.0:8084->80/tcp, :::8084->80/tcp   ecommerce
```

> **Validation Points:**
> * `STATUS` must show `Up` with an uptime value
> * `PORTS` must reflect `0.0.0.0:8084->80/tcp` confirming both IPv4 and IPv6 binding
> * `NAMES` must show `ecommerce`

**[SCREENSHOT PLACEHOLDER: docker ps output showing ecommerce container in Up status with 0.0.0.0:8084->80/tcp port mapping]**

---

### Step 8: Inspect Port Bindings via docker inspect

Use `docker inspect` to programmatically confirm the port binding configuration at the container metadata level.

```bash
sudo docker inspect ecommerce | grep -A 5 "PortBindings"
```

**Expected Output:**

```json
"PortBindings": {
    "80/tcp": [
        {
            "HostIp": "",
            "HostPort": "8084"
        }
```

> An empty `HostIp` value (`""`) means Docker binds to all available host interfaces, which is the expected behavior here.

**[SCREENSHOT PLACEHOLDER: docker inspect output showing PortBindings with 80/tcp mapped to HostPort 8084]**

---

### Step 9: Validate nginx Is Serving HTTP Traffic

Perform a live HTTP request to `localhost:8084` from within the server to confirm end-to-end traffic flow through the container.

```bash
curl http://localhost:8084
```

**Expected Output:**

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
</body>
</html>
```

> A successful response with nginx's default HTML page confirms:
> * The container is running and healthy
> * Port 8084 on the host correctly forwards traffic to port 80 inside the container
> * nginx is serving requests inside the container as expected

**SCREENSHOT:** 

<img width="1132" height="784" alt="image" src="https://github.com/user-attachments/assets/61609576-906e-4939-8d9e-808f491b3596" />

>curl http://localhost:8084 returning the nginx welcome HTML confirming full end-to-end connectivity

---

## Verification and Validation

Use the following checklist to confirm all acceptance criteria have been met before closing the ticket:

- [ ] SSH access to `stapp01` established successfully
- [ ] Docker daemon is `active (running)` via `systemctl status docker`
- [ ] `nginx:stable` image pulled and present in `docker images`
- [ ] Container named `ecommerce` exists in `docker ps`
- [ ] Container `STATUS` shows `Up`
- [ ] Port binding is `0.0.0.0:8084->80/tcp`
- [ ] `docker inspect` confirms `HostPort: 8084`
- [ ] `curl http://localhost:8084` returns nginx HTML response

---

## Architecture Diagram

```
+---------------------------+
|      Jump Host            |
|  (thor@jump-host)         |
+------------+--------------+
             |
             | SSH (port 22)
             v
+---------------------------+
|   Application Server 1    |
|   stapp01 (10.244.49.46)  |
|                           |
|  +---------------------+  |
|  |  Docker Engine      |  |
|  |  v26.1.3            |  |
|  |                     |  |
|  |  +---------------+  |  |
|  |  | ecommerce     |  |  |
|  |  | nginx:stable  |  |  |
|  |  |               |  |  |
|  |  | HOST:8084 --> |  |  |
|  |  | CTR:80        |  |  |
|  |  +---------------+  |  |
|  +---------------------+  |
+---------------------------+
```

---

## Best Practices

### Image Management

* **Always pin to a specific tag or digest** rather than using `latest` in production environments. `nginx:stable` is acceptable, but `nginx:stable@sha256:...` provides full immutability.
* **Audit pulled images** with `docker images` immediately after pull to verify image ID and size match expected values. Compare the `Digest` from the pull output against known good values.
* **Scan images for vulnerabilities** before deploying in production using tools like `docker scout`, `trivy`, or `snyk`.

### Container Naming and Lifecycle

* **Use meaningful, task-specific container names** (`ecommerce` in this case) to enable simpler identification in `docker ps`, logs, and monitoring systems.
* **Always run application containers in detached mode** (`-d`) to prevent process hangs if the SSH session drops.
* **Set container restart policies** in non-lab environments to ensure availability. Example:

```bash
sudo docker run -d \
  --name ecommerce \
  --restart unless-stopped \
  -p 8084:80 \
  nginx:stable
```

### Port Binding Security

* **Avoid binding to `0.0.0.0`** on internet-facing hosts unless a firewall or security group rule restricts external access to port 8084. Prefer binding to a specific interface IP in production:

```bash
-p 127.0.0.1:8084:80
```

* **Document all port allocations** in a central registry to prevent conflicts across containerized services on shared hosts.

### Verification Discipline

* **Always run `curl` or `wget` locally** after a container deployment to confirm the service is responding before marking the task complete.
* **Use `docker inspect`** to validate configuration programmatically, not just visually from `docker ps` output.
* **Check `docker logs <container_name>`** to inspect application-level logs for errors that would not surface in `docker ps`:

```bash
sudo docker logs ecommerce
```

---

## Lessons Learned

### 1. Host Key Verification on First SSH Connection

When connecting to a new server for the first time via SSH, the host key fingerprint must be explicitly trusted. In automated pipelines, this must be handled using `StrictHostKeyChecking=no` or by pre-seeding `~/.ssh/known_hosts`. Manual trust confirmation (`yes`) is only acceptable for interactive one-time sessions.

### 2. Sudo Is Required for Docker on Non-Root Users

The user `tony` is not part of the `docker` group on `stapp01`, requiring all Docker commands to be prefixed with `sudo`. In managed environments, adding a user to the `docker` group (`usermod -aG docker tony`) eliminates the sudo requirement but carries a privilege escalation risk equivalent to root access. This tradeoff must be evaluated per environment policy.

### 3. Detached Mode Is Non-Negotiable for SSH-Based Deployments

Running `docker run` without `-d` in an SSH session ties the container process lifecycle to the terminal session. If the SSH connection drops, the container will exit. Always use `-d` for any container intended to outlive the deployment session.

### 4. Port 8084 Must Be Free Before Binding

If another process is already bound to host port 8084, `docker run` will fail with a bind error. Always verify port availability before running:

```bash
sudo ss -tlnp | grep 8084
```

or

```bash
sudo lsof -i :8084
```

### 5. Validate at the Application Layer, Not Just the Infrastructure Layer

Confirming `docker ps` shows `Up` is a necessary but insufficient check. The `curl` test validates that nginx inside the container is actually responding to HTTP requests, which rules out cases where the container is running but the service inside has crashed or failed to bind to port 80.

### 6. nginx:stable vs nginx:latest

The `stable` tag follows nginx's stable release branch, which receives security and bug fix backports without introducing breaking changes. The `latest` tag tracks the mainline (development) branch, which may include experimental features. For production and production-like environments, `stable` is always the correct choice unless the workload explicitly requires mainline-only features.

---

## References

* [Docker Official Documentation: docker run](https://docs.docker.com/engine/reference/commandline/run/)
* [Docker Official Documentation: docker inspect](https://docs.docker.com/engine/reference/commandline/inspect/)
* [nginx Docker Hub Official Image](https://hub.docker.com/_/nginx)
* [Docker Networking: Published Ports](https://docs.docker.com/config/containers/container-networking/)
* [systemd Docker Service Management](https://docs.docker.com/config/daemon/systemd/)
* [Docker Security Best Practices](https://docs.docker.com/engine/security/)

---








<img width="1131" height="465" alt="image" src="https://github.com/user-attachments/assets/d7d8f7c6-ab11-4dd7-a6b2-3642c5eba560" />
<img width="1133" height="593" alt="image" src="https://github.com/user-attachments/assets/50ef05c1-f857-47a0-aa7a-5884148e39c5" />


