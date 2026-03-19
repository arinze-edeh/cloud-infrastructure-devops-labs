# Docker Custom Image Build: Apache2 on Ubuntu 24.04

![Status](https://img.shields.io/badge/Status-Resolved-brightgreen)
![Platform](https://img.shields.io/badge/Platform-Linux%2FDocker-blue)
![Skill Level](https://img.shields.io/badge/Level-DevOps%20%7C%20SRE-orange)
![Environment](https://img.shields.io/badge/Environment-Stratos%20DC-purple)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure Context](#infrastructure-context)
- [Prerequisites](#prerequisites)
- [Solution Walkthrough](#solution-walkthrough)
- [Dockerfile Reference](#dockerfile-reference)
- [Verification](#verification)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)

---

## Overview

This document details the end-to-end process of creating a custom Docker image for the Nautilus application development team within Stratos DC. The image is built on **Ubuntu 24.04** with **Apache2** configured to listen on a non-standard port (`6100`). The Dockerfile is placed at `/opt/docker/Dockerfile` on **Application Server 3 (stapp03)**.

This runbook follows enterprise DevOps standards and is intended for use by SRE, platform engineering, and DevOps teams.

---

## Problem Statement

The Nautilus application development team required a custom Docker base image to support one of their internal projects. The image needed to satisfy the following hard requirements:

| Requirement | Value |
|---|---|
| Base Image | `ubuntu:24.04` |
| Package | `apache2` |
| Listening Port | `6100` (non-default) |
| Dockerfile Path | `/opt/docker/Dockerfile` |
| Target Server | App Server 3 (`stapp03`) |
| Constraint | Do not modify document root or other Apache config settings |

The default Apache2 port (`80`) had to be remapped to `6100` by patching `ports.conf` and the default virtual host configuration file, without altering any other Apache directives.

---

## Infrastructure Context

The following infrastructure details from Stratos DC were referenced for this task.

| Server Name | Hostname | User | Password | Purpose |
|---|---|---|---|---|
| Application Server 1 | stapp01 | tony | Ir0nM@n | Hosts Nautilus Application 1 |
| Application Server 2 | stapp02 | steve | Am3ric@ | Hosts Nautilus Application 2 |
| **Application Server 3** | **stapp03** | **banner** | **BigGr33n** | **Hosts Nautilus Application 3** |
| LoadBalancer Server | stlb01 | loki | Mischi3f | Distributes traffic for Nautilus HTTP |
| Database Server | stdb01 | peter | Sp!dy | Hosts Nautilus Database |
| Storage Server | ststor01 | natasha | Bl@kW | Stores data for Nautilus Servers |
| Backup Server | stbkp01 | clint | H@wk3y3 | Manages backups for Nautilus Servers |
| Mail Server | stmail01 | groot | Gr00T123 | Manages email services for Nautilus Servers |
| Jump Host Server | jump-host | thor | mjolnir123 | Provides secure access to Stork DC |
| Jenkins Server | jenkins | jenkins | j@rv!s | Runs Jenkins for CI/CD pipeline |

> **Note:** All server IPs are dynamic. Access to app servers is routed through the Jump Host (`jump-host`) using the user `thor`.

---

## Prerequisites

Before executing this runbook, confirm the following:

- [ ] SSH access to the Jump Host (`jump-host`) as user `thor`
- [ ] Credentials for `banner@stapp03` are available
- [ ] The directory `/opt/docker/` exists on `stapp03`
- [ ] `sudo` privileges are granted to user `banner` on `stapp03`
- [ ] Docker is installed and the daemon is running on `stapp03` (if building the image)
- [ ] Network connectivity from the Jump Host to `stapp03` is confirmed

---

## Solution Walkthrough

### Step 1: Connect to the Jump Host

From your local workstation, SSH into the Jump Host using the credentials for `thor`.

```bash
ssh thor@<jump-host-ip>
```

---

### Step 2: SSH from Jump Host into Application Server 3

From the Jump Host, SSH into `stapp03` using the `banner` account.

```bash
ssh banner@stapp03
```

Accept the host key fingerprint when prompted by typing `yes`.

> **Screenshot**

<img width="1028" height="425" alt="image" src="https://github.com/user-attachments/assets/edbe443e-93fc-454f-a465-fd914d878c36" />

> `SSH session from jump-host to stapp03 showing host key prompt and successful authentication`

---

### Step 3: Verify Your Hostname

Confirm you are on the correct server before making any changes.

```bash
hostname
```

**Expected output:**
```
stapp03
```

> **Screenshot**

<img width="1028" height="425" alt="image" src="https://github.com/user-attachments/assets/edbe443e-93fc-454f-a465-fd914d878c36" />

> `Terminal output of hostname command returning stapp03`

---

### Step 4: Verify the Target Directory Exists

Confirm that `/opt/docker/` exists on this server before writing the Dockerfile.

```bash
ls -ld /opt/docker/
```

**Expected output:**
```
drwxr-xr-x 2 root root 4096 Mar 19 01:23 /opt/docker/
```

> **Screenshot**

<img width="1036" height="397" alt="image" src="https://github.com/user-attachments/assets/6daaa36d-d491-4f39-a092-68141b62186f" />

> `ls -ld output confirming /opt/docker/ directory exists with correct permissions`

---

### Step 5: Create the Dockerfile

Use `sudo` to create the Dockerfile at the required path. Note that the filename must be capitalized exactly as `Dockerfile`.

```bash
sudo vi /opt/docker/Dockerfile
```

Enter your `sudo` password (`BigGr33n`) when prompted.

Inside the editor, add the following content:

```dockerfile
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y apache2

RUN sed -i 's/Listen 80/Listen 6100/' /etc/apache2/ports.conf && \
    sed -i 's/<VirtualHost \*:80>/<VirtualHost *:6100>/' /etc/apache2/sites-available/000-default.conf

EXPOSE 6100

CMD ["apache2ctl", "-D", "FOREGROUND"]
```

Save and exit the editor (`:wq` in vi).

> **Screenshot**

<img width="1042" height="867" alt="image" src="https://github.com/user-attachments/assets/f958116a-6273-4dd3-b599-19519128e2c6" />

> `vi editor open at /opt/docker/Dockerfile showing all Dockerfile instructions`

---

### Step 6: Verify the Dockerfile Contents

After saving, confirm the file content is correct by printing it to the terminal.

```bash
cat /opt/docker/Dockerfile
```

**Expected output:**
```dockerfile
FROM ubuntu:24.04

RUN apt-get update && apt-get install -y apache2

RUN sed -i 's/Listen 80/Listen 6100/' /etc/apache2/ports.conf && \
    sed -i 's/<VirtualHost \*:80>/<VirtualHost *:6100>/' /etc/apache2/sites-available/000-default.conf

EXPOSE 6100

CMD ["apache2ctl", "-D", "FOREGROUND"]
```

> **Screenshot**

<img width="1030" height="749" alt="image" src="https://github.com/user-attachments/assets/6e3d2839-2ef9-4316-b350-ad65134dd52a" />

> `cat /opt/docker/Dockerfile output confirming all lines match the specification exactly`

---

### Step 7: Exit the App Server and Return to Jump Host

Once the file is confirmed, exit `stapp03` to return to the Jump Host.

```bash
exit
```

**Expected output:**
```
logout
Connection to stapp03 closed.
```

> **Screenshot**

<img width="1030" height="780" alt="image" src="https://github.com/user-attachments/assets/8087fb49-b26d-4b0d-913b-202676d84a82" />

> `Terminal showing exit and connection closed confirmation, returning to jump-host prompt`

---

## Dockerfile Reference

Below is the complete Dockerfile with inline annotations explaining each instruction.

```dockerfile
# Use Ubuntu 24.04 as the base image (LTS release for stability)
FROM ubuntu:24.04

# Update package index and install Apache2 in a single layer
# -y flag ensures non-interactive installation
RUN apt-get update && apt-get install -y apache2

# Patch Apache2 configuration to listen on port 6100
# Only port references are changed; document root and other settings remain untouched
RUN sed -i 's/Listen 80/Listen 6100/' /etc/apache2/ports.conf && \
    sed -i 's/<VirtualHost \*:80>/<VirtualHost *:6100>/' /etc/apache2/sites-available/000-default.conf

# Document the container's listening port for orchestration tools
EXPOSE 6100

# Start Apache2 in the foreground to keep the container running
CMD ["apache2ctl", "-D", "FOREGROUND"]
```

### Instruction Breakdown

| Instruction | Purpose |
|---|---|
| `FROM ubuntu:24.04` | Pulls the official Ubuntu 24.04 LTS base image |
| `RUN apt-get update && apt-get install -y apache2` | Installs Apache2 web server |
| `RUN sed -i ... ports.conf` | Replaces port `80` with `6100` in Apache's global port config |
| `RUN sed -i ... 000-default.conf` | Updates the default virtual host to bind on port `6100` |
| `EXPOSE 6100` | Declares the container's network listening port |
| `CMD [...]` | Runs `apache2ctl` in the foreground as the container entry point |

---

## Verification (Optional)

After creating the Dockerfile, optionally build and test the image to confirm correctness.

### Build the Image

```bash
cd /opt/docker/
sudo docker build -t nautilus-apache2:v1 .
```


### Run a Test Container

```bash
sudo docker run -d -p 6100:6100 --name nautilus-test nautilus-apache2:v1
```

### Confirm the Container Is Running

```bash
sudo docker ps | grep nautilus-test
```

### Test Apache2 Is Responding on Port 6100

```bash
curl -I http://localhost:6100
```

**Expected response:**
```
HTTP/1.1 200 OK
...
Server: Apache/2.x.x (Ubuntu)
```



### Inspect the Port Inside the Container

```bash
sudo docker exec nautilus-test ss -tlnp | grep 6100
```


---

## Best Practices

### Dockerfile Authoring

* **Combine related `RUN` instructions** into a single layer using `&&` and `\` continuations. This reduces image size and improves build cache efficiency.
* **Use specific base image tags** (e.g., `ubuntu:24.04`) rather than `ubuntu:latest` to guarantee build reproducibility across environments and over time.
* **Never install unnecessary packages.** The `--no-install-recommends` flag can be added to `apt-get install` to further minimize image bloat:

  ```dockerfile
  RUN apt-get update && apt-get install -y --no-install-recommends apache2
  ```

* **Clean the apt cache** in the same `RUN` layer to avoid caching package index files in the image:

  ```dockerfile
  RUN apt-get update && apt-get install -y apache2 && rm -rf /var/lib/apt/lists/*
  ```

* **Always use `EXPOSE`** to document the ports your service uses. This is used by container orchestration systems and is a contract for consumers of the image.

### Configuration Management

* **Use `sed` for targeted configuration changes** rather than overwriting entire config files. This is safer and more maintainable as it preserves upstream configuration structure.
* **Patch only what is required.** In this task, only the port directives in `ports.conf` and `000-default.conf` were modified. Document root, log paths, and module configurations were left untouched, exactly as specified.
* **Avoid hardcoding environment-specific values** in production Dockerfiles. Consider using `ARG` or `ENV` instructions to make the port configurable at build time:

  ```dockerfile
  ARG APACHE_PORT=6100
  ENV APACHE_PORT=${APACHE_PORT}
  RUN sed -i "s/Listen 80/Listen ${APACHE_PORT}/" /etc/apache2/ports.conf
  ```

### Access and Security

* **Use a Jump Host for all access** to production and staging servers. Never expose app server SSH ports directly to the internet.
* **Avoid running containers as root** in production. Add a dedicated non-root user in the Dockerfile for runtime security:

  ```dockerfile
  RUN useradd -r -s /bin/false appuser
  USER appuser
  ```

* **Rotate credentials regularly.** The static passwords used in lab environments must never be reused or referenced in production systems.

### Operational Habits

* **Always verify your target host** (`hostname`) before making changes. Editing the wrong server is a common and costly mistake.
* **Use `cat` to validate file contents** immediately after writing. Do not assume the file was saved correctly.
* **Tag Docker images semantically** (e.g., `nautilus-apache2:v1.0.0`) to enable rollback and audit trails.

---

## Lessons Learned

### 1. Port Configuration Requires Two File Changes

Apache2 on Ubuntu uses two separate files to control which port a virtual host listens on:

* `/etc/apache2/ports.conf` controls the global `Listen` directive.
* `/etc/apache2/sites-available/000-default.conf` controls the `VirtualHost` binding.

Changing only one of these files will cause Apache to either fail to start or serve on the wrong port. Both files must be updated atomically in the same `RUN` instruction.

### 2. Dockerfile Filename Casing Is Case-Sensitive

The Docker build system looks for `Dockerfile` with an uppercase `D` by default. A file named `dockerfile` or `DockerFile` will not be found by `docker build .` unless explicitly specified with the `-f` flag. Always use the canonical capitalization.

### 3. `sed` Escaping in Shell Commands

The `sed` expression for the VirtualHost line requires escaping the `*` character and the `:` in some shells. Using single quotes for the `sed` expression avoids most shell interpolation issues, but care must be taken when patterns contain special regex characters such as `.`, `*`, `[`, and `\`.

### 4. Sudo Privilege Awareness

The `/opt/docker/` directory is owned by `root`. The non-root user `banner` must use `sudo` to write files into this directory. This is expected behavior in hardened environments and should not be circumvented by changing directory ownership permanently.

### 5. CMD vs ENTRYPOINT for Service Containers

Using `CMD ["apache2ctl", "-D", "FOREGROUND"]` keeps Apache running in the foreground, which is required for Docker containers (a container exits when its PID 1 process exits). In production, using `ENTRYPOINT` instead of `CMD` provides better flexibility when overriding arguments at runtime.

### 6. Layer Caching and Build Efficiency

Placing `apt-get update` and `apt-get install` in the same `RUN` layer is critical. If they are separated into two `RUN` instructions, Docker may use a stale cache for `apt-get update` when only the install list changes, leading to installation failures or outdated packages.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `docker build` fails at `apt-get install` | No internet access from container build context | Verify Docker daemon has external network access; check proxy settings |
| Apache does not start in container | Missing `apache2` service configuration | Ensure `CMD` uses `apache2ctl -D FOREGROUND` and not `service apache2 start` |
| Port 6100 not reachable after `docker run` | Port not mapped with `-p` flag | Re-run with `-p 6100:6100` |
| `sed` command fails silently | Pattern not found in config file | Inspect the actual config file inside the image with `docker run ... bash -c "cat /etc/apache2/ports.conf"` |
| `Permission denied` writing Dockerfile | Missing sudo / file owned by root | Use `sudo vi /opt/docker/Dockerfile` |
| Wrong server modified | Skipped hostname verification | Always run `hostname` immediately after SSH login |

---
