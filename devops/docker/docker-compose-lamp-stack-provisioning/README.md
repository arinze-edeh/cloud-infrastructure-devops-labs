# Containerized Application Stack Deployment with Docker Compose

> **Enterprise-style DevOps | Nautilus Infrastructure | Stratos Datacenter**

> Deploying a production-ready, multi-service containerized stack (PHP/Apache + MariaDB) on Application Server 3 using Docker Compose with persistent volume mounts and environment-based secrets management.

---

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Infrastructure Details](#infrastructure-details)
* [Step-by-Step Deployment](#step-by-step-deployment)
* [Verification and Validation](#verification-and-validation)
* [Configuration Reference](#configuration-reference)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [Troubleshooting](#troubleshooting)

---

## Overview

### Problem Statement

The Nautilus Application Development and DevOps teams needed to deploy a multi-service application on a containerized platform. Prior to going live, the team required a validated, reproducible deployment of the full containerized stack on **Application Server 3 (stapp03)** using a Docker Compose manifest. The stack consists of a PHP/Apache web frontend and a MariaDB database backend, wired together via a shared Docker network with host volume mounts for data persistence.

### Resolution Summary

A Docker Compose file was authored at the canonical path `/opt/sysops/docker-compose.yml` defining two services: `php_web` (PHP with Apache) and `mysql_web` (MariaDB). Both containers were deployed in detached mode, port-mapped to the host, and validated end-to-end via `curl` against the web service endpoint.

---

## Architecture

```
                        Stratos Datacenter
                       +------------------+
                       |   stapp03        |
                       |                  |
   Host Port 8085 ---->|  [php_web]       |
                       |  php:apache      |
                       |  /var/www/html   |
                       |        |         |
                       |  [Docker Network]|
                       |        |         |
   Host Port 3306 ---->|  [mysql_web]     |
                       |  mariadb:latest  |
                       |  /var/lib/mysql  |
                       +------------------+
```

**Services:**

| Service | Image | Host Port | Container Port | Volume Mount |
|---|---|---|---|---|
| `php_web` | `php:apache` | `8085` | `80` | `/var/www/html:/var/www/html` |
| `mysql_web` | `mariadb:latest` | `3306` | `3306` | `/var/lib/mysql:/var/lib/mysql` |

---

## Prerequisites

* Docker Engine `26.1.3+` installed on the target server
* Docker Compose `v5.0.2+` available as a CLI plugin
* `sudo` privileges for the operating user (`banner`)
* SSH access via Jump Host (`jump-host`) to `stapp03`
* Host directories `/var/www/html` and `/var/lib/mysql` created before stack launch

---

## Infrastructure Details

| Server | Hostname | User | Purpose |
|---|---|---|---|
| Application Server 3 | `stapp03` | `banner` | Target deployment host |
| Jump Host Server | `jump-host` | `thor` | Secure ingress to Stratos DC |

---

## Step-by-Step Deployment

### Step 1: SSH from Jump Host into Application Server 3

From the Jump Host, establish an SSH session to `stapp03` using the `banner` account.

```bash
thor@jump-host ~$ ssh banner@stapp03
```

Accept the host key fingerprint on first connection:

```
The authenticity of host 'stapp03 (10.244.73.170)' can't be established.
ED25519 key fingerprint is SHA256:gPc0C/gbXPN3vdqrWn8Pcff3bzSpjC1GwXqgappknMQ.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
banner@stapp03's password:
```

> **Note:** The fingerprint is stored permanently in `~/.ssh/known_hosts` after first acceptance. For production environments, validate the fingerprint out-of-band before accepting.

**SCREENSHOT:** 

<img width="1032" height="469" alt="image" src="https://github.com/user-attachments/assets/15d97e48-f4f4-485d-a485-62fe3be8e1a7" />

>Successful SSH login to stapp03 from jump-host

---

### Step 2: Verify Docker and Docker Compose Installation

Confirm that both Docker Engine and the Compose plugin are available and at the expected versions.

```bash
[banner@stapp03 ~]$ docker --version
docker compose version
```

**Expected output:**

```
Docker version 26.1.3, build b72abbb
Docker Compose version v5.0.2
```

**SCREENSHOT:** 

<img width="1035" height="501" alt="image" src="https://github.com/user-attachments/assets/e1bd9115-8369-4008-b317-06ef9f771c6e" />


>docker --version and docker compose version output on stapp03

---

### Step 3: Create the Sysops Working Directory

Create the directory `/opt/sysops` which will house the Compose manifest. The `-p` flag ensures the full path is created non-destructively.

```bash
[banner@stapp03 ~]$ sudo mkdir -p /opt/sysops
```

Verify ownership and permissions:

```bash
[banner@stapp03 ~]$ ls -ld /opt/sysops
```

**Expected output:**

```
drwxr-xr-x 2 root root 4096 Mar 24 01:01 /opt/sysops
```

**SCREENSHOT:** 

<img width="1036" height="613" alt="image" src="https://github.com/user-attachments/assets/65964fae-3f44-43b8-9c99-853aeea2a13a" />

>ls -ld /opt/sysops showing correct directory creation

---

### Step 4: Author the Docker Compose Manifest

Open the Compose file at the exact path required: `/opt/sysops/docker-compose.yml`.

```bash
[banner@stapp03 ~]$ sudo vi /opt/sysops/docker-compose.yml
```

Paste the following configuration exactly:

```yaml
services:
  php_web:
    container_name: php_web
    image: php:apache
    ports:
      - "8085:80"
    volumes:
      - /var/www/html:/var/www/html

  mysql_web:
    container_name: mysql_web
    image: mariadb:latest
    ports:
      - "3306:3306"
    volumes:
      - /var/lib/mysql:/var/lib/mysql
    environment:
      MYSQL_DATABASE: database_web
      MYSQL_USER: db_user
      MYSQL_PASSWORD: Str0ngP@ssw0rd
      MYSQL_ROOT_PASSWORD: R00tStr0ng@Pass
```

Verify the written file:

```bash
[banner@stapp03 ~]$ cat /opt/sysops/docker-compose.yml
```

**SCREENSHOTS:** 


<img width="1036" height="767" alt="image" src="https://github.com/user-attachments/assets/10479e1a-d4bf-47db-be67-c97830acae03" />
<img width="1034" height="545" alt="image" src="https://github.com/user-attachments/assets/b3dd8fa9-757d-4428-a1c2-f6f8c5317cac" />

>cat output of /opt/sysops/docker-compose.yml showing full contents

---

### Step 5: Pre-Create Host Volume Mount Directories

Before launching the stack, ensure the host-side bind mount paths exist to prevent Docker from creating them with incorrect permissions or ownership.

```bash
[banner@stapp03 ~]$ sudo mkdir -p /var/www/html
[banner@stapp03 ~]$ sudo mkdir -p /var/lib/mysql
```

**SCREENSHOT:** 


<img width="1034" height="579" alt="image" src="https://github.com/user-attachments/assets/c7df766a-f5f0-42c3-a387-3f81e4c29b41" />

>mkdir commands for /var/www/html and /var/lib/mysql

---

### Step 6: Launch the Stack with Docker Compose

Navigate to the Compose file directory and start all services in detached mode.

```bash
[banner@stapp03 ~]$ cd /opt/sysops
[banner@stapp03 sysops]$ sudo docker compose up -d
```

**Expected output:**

```
[+] up 27/27
 ✔ Image mariadb:latest   Pulled                                                                                          11.0s
 ✔ Image php:apache       Pulled                                                                                          14.0s
 ✔ Network sysops_default Created                                                                                         0.1s
 ✔ Container php_web      Created                                                                                         0.2s
 ✔ Container mysql_web    Created                                                                                         0.2s
```

> Both images are pulled from Docker Hub, a default bridge network (`sysops_default`) is created automatically, and both containers are started.

**SCREENSHOT:** 

<img width="1031" height="732" alt="image" src="https://github.com/user-attachments/assets/75efba69-ddf9-4d34-b089-2087c4156166" />

>docker compose up -d output showing all 27 layers pulled and both containers created

---

### Step 7: Validate Running Containers

Confirm both containers are in a healthy `Up` state with correct port bindings.

```bash
[banner@stapp03 sysops]$ sudo docker ps
```

**Expected output:**

```
CONTAINER ID   IMAGE            COMMAND                  CREATED          STATUS          PORTS                                       NAMES
880461c89284   php:apache       "docker-php-entrypoi…"   31 seconds ago   Up 29 seconds   0.0.0.0:8085->80/tcp, :::8085->80/tcp       php_web
30e14804de45   mariadb:latest   "docker-entrypoint.s…"   31 seconds ago   Up 29 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp   mysql_web
```

**SCREENSHOT:** 

<img width="1176" height="803" alt="image" src="https://github.com/user-attachments/assets/1a836970-11d4-4ecf-9e66-3eb4e9d022c0" />

>docker ps output showing php_web on port 8085 and mysql_web on port 3306 both in Up state

---

### Step 8: End-to-End Web Service Validation

Perform an HTTP request against the PHP/Apache service to confirm it is serving traffic correctly.

```bash
[banner@stapp03 sysops]$ curl http://localhost:8085
```

**Expected output:**

```html
<html>
    <head>
        <title>Welcome to xFusionCorp Industries!</title>
    </head>
    <body>
        Welcome to xFusionCorp Industries!    </body>
</html>
```

A `200 OK` response with the application HTML confirms full stack health.

**SCREENSHOT:** 


<img width="1173" height="863" alt="image" src="https://github.com/user-attachments/assets/b7e730d5-57cd-4a33-950d-5521e0de13d6" />

>curl http://localhost:8085 returning the xFusionCorp Industries welcome page HTML

---

## Verification and Validation

### Checklist

* [x] SSH access to `stapp03` via Jump Host confirmed
* [x] Docker Engine `26.1.3` verified
* [x] Docker Compose `v5.0.2` verified
* [x] Directory `/opt/sysops` created with correct permissions
* [x] `docker-compose.yml` authored at exact canonical path
* [x] Host bind mount directories pre-created
* [x] Both images pulled successfully from Docker Hub
* [x] Both containers running in `Up` state
* [x] Port `8085` mapped correctly for `php_web`
* [x] Port `3306` mapped correctly for `mysql_web`
* [x] `curl http://localhost:8085` returns expected HTML response
* [x] `MYSQL_DATABASE` set to `database_web`
* [x] Non-root MySQL user configured with complex password

---

## Configuration Reference

### docker-compose.yml Fields Explained

| Field | Value | Purpose |
|---|---|---|
| `container_name` | `php_web` / `mysql_web` | Deterministic naming; avoids random suffix generation |
| `image` | `php:apache` | Official PHP image with Apache pre-bundled |
| `image` | `mariadb:latest` | Official MariaDB image; latest tag used as per task spec |
| `ports` | `8085:80` | Maps host port 8085 to container Apache port 80 |
| `ports` | `3306:3306` | Exposes MySQL-compatible port directly to host |
| `volumes` | `/var/www/html:/var/www/html` | Bind mount for web content persistence |
| `volumes` | `/var/lib/mysql:/var/lib/mysql` | Bind mount for database file persistence |
| `MYSQL_DATABASE` | `database_web` | Bootstrap database created on first run |
| `MYSQL_USER` | `db_user` | Non-root application user (root excluded per policy) |
| `MYSQL_PASSWORD` | `Str0ngP@ssw0rd` | Complex password for application user |
| `MYSQL_ROOT_PASSWORD` | `R00tStr0ng@Pass` | Root password required by MariaDB on init |

---

## Best Practices

### Secrets Management

* **Never hardcode credentials in Compose files for production.** Use Docker Secrets (`docker secret create`) or an external secrets manager such as HashiCorp Vault, AWS Secrets Manager, or environment variable injection via CI/CD pipelines.
* Reference secrets in Compose using the `secrets:` top-level key and `secrets` block per service.

### Image Tagging Strategy

* Avoid `latest` tags in production workloads. Pin to explicit digest or semantic version (for example, `mariadb:11.3.2`) to guarantee reproducibility and prevent unexpected breaking changes during re-pulls.
* Use image digest pinning for immutable deployments: `mariadb@sha256:<digest>`.

### Volume Management

* Pre-create bind mount directories with explicit ownership before compose up to prevent root-owned directory creation by the Docker daemon.
* For stateful services in production, prefer **named volumes** over bind mounts for improved portability and Docker-managed lifecycle.

### Network Isolation

* In production, define explicit named networks in the Compose file and segment frontend and backend services onto separate networks. Only expose the web service externally; keep the database on an internal-only network.

### Health Checks

* Add `healthcheck` directives to each service to enable dependency ordering and liveness monitoring. Example for MariaDB:

```yaml
healthcheck:
  test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
  interval: 10s
  timeout: 5s
  retries: 3
```

### Resource Limits

* Define `deploy.resources.limits` for CPU and memory to prevent a rogue container from exhausting host resources in shared environments.

### Logging

* Configure a centralized logging driver (`json-file` with rotation, `fluentd`, or `loki`) on each service to maintain log visibility across container restarts.

---

## Lessons Learned

**1. File naming is exact and non-negotiable.**
The task specified `docker-compose.yml` at `/opt/sysops/docker-compose.yml`. Any deviation in path or filename causes the compose stack to be undetected by validation tooling. Always confirm the exact expected artifact path before authoring.

**2. Pre-creating bind mount directories prevents permission footguns.**
When Docker creates host directories automatically via bind mounts, it does so as `root`. Applications running as non-root inside containers may encounter permission errors. Pre-creating directories and setting ownership explicitly eliminates this entire failure class.

**3. `sudo docker compose` vs `docker compose` matters in multi-user environments.**
On systems where the operating user is not in the `docker` group, all Docker commands require `sudo`. This is expected for hardened, production-adjacent hosts. In team environments, explicitly document whether Docker group membership is provisioned or whether `sudo` is the intended pattern.

**4. `docker compose up -d` pulls images automatically.**
No separate `docker pull` step is needed. The Compose CLI resolves and pulls all referenced images before container creation. This simplifies the deployment sequence to a single command after the manifest is in place.

**5. Validating with `curl` is the simplest, most reliable smoke test.**
A successful HTTP response from `curl http://localhost:<port>` confirms the container is running, the port binding is active, the web server is operational, and the application content is being served. This single command validates multiple layers simultaneously.

**6. Jump Host SSH fingerprint acceptance is a one-time operation.**
The `Warning: Permanently added` message is expected and benign on first connection. In fully automated CI/CD pipelines, use `StrictHostKeyChecking=no` or pre-populate `known_hosts` via a provisioning step to prevent interactive prompts from blocking automation.

**7. Environment variables in Compose are not encrypted at rest.**
Credentials visible in `docker inspect` or `cat docker-compose.yml` are a real security risk. Treat Compose files with embedded secrets as sensitive artifacts and protect them with filesystem ACLs, `.gitignore` exclusions, and vault-backed secret injection at runtime.

---

## Troubleshooting

| Symptom | Probable Cause | Resolution |
|---|---|---|
| `permission denied` on `docker compose up` | User not in `docker` group | Prefix with `sudo` or add user to docker group |
| Port already in use on `3306` or `8085` | Existing service occupying host port | Run `sudo ss -tlnp \| grep <port>` and stop the conflicting process |
| Container exits immediately after creation | Missing environment variable or image entrypoint failure | Run `sudo docker logs <container_name>` to inspect startup errors |
| `curl` returns connection refused on port 8085 | Container not yet healthy or wrong port mapping | Verify with `sudo docker ps` and inspect port bindings |
| Volume data not persisting across restarts | Bind mount path incorrect or permissions mismatch | Verify host path exists with `ls -ld <path>` and check container user UID |
| `image not found` error during pull | Incorrect image name or tag / network unrestricted | Verify image name spelling and Docker Hub connectivity from the host |

---
