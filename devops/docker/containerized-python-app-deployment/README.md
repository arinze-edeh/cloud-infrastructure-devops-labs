# Containerizing a Python Flask Application with Docker on a Remote Linux Server

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=for-the-badge&logo=flask&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
- [Environment Details](#environment-details)
- [Prerequisites](#prerequisites)
- [Step-by-Step Resolution](#step-by-step-resolution)
  - [Step 1: Remote Server Access via SSH](#step-1-remote-server-access-via-ssh)
  - [Step 2: Inspect the Application Directory](#step-2-inspect-the-application-directory)
  - [Step 3: Review Application Source Files](#step-3-review-application-source-files)
  - [Step 4: Author the Dockerfile](#step-4-author-the-dockerfile)
  - [Step 5: Build the Docker Image](#step-5-build-the-docker-image)
  - [Step 6: Run the Container](#step-6-run-the-container)
  - [Step 7: Verify the Deployment](#step-7-verify-the-deployment)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Project File Structure](#project-file-structure)

---

## Problem Statement

A Python Flask web application required containerization and deployment on **App Server 2 (stapp02)** within a Kubernetes-managed Nautilus cluster environment. The application source code and its dependency manifest (`requirements.txt`) were pre-staged under `/python_app/src/` on the target host. The objective was to:

1. Author a valid `Dockerfile` under `/python_app/`
2. Build a Docker image tagged `nautilus/python-app`
3. Launch a container named `pythonapp_nautilus` with the internal port `8089` mapped to host port `8095`
4. Validate the running application responds correctly via `curl`

---

## Architecture Overview

```
Jump Host (thor)
     |
     | SSH
     v
stapp02 (App Server 2) -- 10.244.244.168
     |
     |-- /python_app/
     |       |-- Dockerfile          <-- Created in this task
     |       |-- src/
     |               |-- server.py
     |               |-- requirements.txt
     |
     |-- Docker Engine
             |
             |-- Image: nautilus/python-app
             |-- Container: pythonapp_nautilus
                     |-- Container Port: 8089
                     |-- Host Port: 8095
```

---

## Environment Details

| Parameter | Value |
|---|---|
| Target Host | stapp02 |
| Host IP | 10.244.244.168 |
| OS User | steve |
| App Root | `/python_app/` |
| Source Directory | `/python_app/src/` |
| Docker Image Name | `nautilus/python-app` |
| Container Name | `pythonapp_nautilus` |
| Container Port | `8089` |
| Host Port | `8095` |
| Base Image | `python:latest` |
| Application Framework | Flask |

---

## Prerequisites

- SSH access to `stapp02` from the jump host
- `sudo` privileges for the `steve` user on `stapp02`
- Docker Engine installed and running on `stapp02`
- Application source files pre-staged at `/python_app/src/`

---

## Step-by-Step Resolution

### Step 1: Remote Server Access via SSH

From the jump host (`thor`), connect to the target application server via SSH.

```bash
thor@jump-host ~$ ssh steve@stapp02
```

**Expected Output:**

```
The authenticity of host 'stapp02 (10.244.244.168)' can't be established.
ED25519 key fingerprint is SHA256:IGsOHa+VgF6m5stVFeHMQH6A2w2/mvhCCuHge/97Vt4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
steve@stapp02's password:
```

> **Note:** The SSH host key warning appears because this is the first connection to `stapp02` from this jump host. Typing `yes` permanently adds the fingerprint to `~/.ssh/known_hosts`. In production pipelines, host key verification should be pre-configured or managed via Ansible/Terraform to avoid interactive prompts.

> **Screenshot**

<img width="1034" height="442" alt="image" src="https://github.com/user-attachments/assets/ac455554-1cd0-4b2d-b7da-95c077ed405c" />

> `Terminal showing successful SSH login to stapp02 with host key fingerprint acceptance prompt`

---

### Step 2: Inspect the Application Directory

After logging in, verify the pre-staged application structure.

```bash
[steve@stapp02 ~]$ ls -la /python_app/src/
```

**Output:**

```
total 16
drwxr-xr-x 2 root root 4096 Mar 25 20:46 .
drwxr-xr-x 3 root root 4096 Mar 25 20:46 ..
-rw-r--r-- 1 root root    5 Mar 25 20:46 requirements.txt
-rw-r--r-- 1 root root  278 Mar 25 20:46 server.py
```

Also verify the parent directory structure:

```bash
[steve@stapp02 ~]$ ls -la /python_app/
```

**Output:**

```
total 12
drwxr-xr-x 3 root root 4096 Mar 25 20:46 .
dr-xr-xr-x 1 root root 4096 Mar 25 20:57 ..
drwxr-xr-x 2 root root 4096 Mar 25 20:46 src
```

> **Observation:** All files and directories under `/python_app/` are owned by `root`. The `steve` user has read and execute permissions but **no write access**. This is a critical observation that will cause a permission failure in a subsequent step.

> **Screenshot**

<img width="1029" height="620" alt="image" src="https://github.com/user-attachments/assets/de438761-1400-4698-a625-735fc4fc7343" />

> `Terminal output showing ls -la /python_app/ and /python_app/src/ with root ownership`

---

### Step 3: Review Application Source Files

Inspect the dependency manifest to understand what packages the application requires.

```bash
[steve@stapp02 ~]$ cat /python_app/src/requirements.txt
```

**Output:**

```
flask
```

> **Observation:** The only declared dependency is `flask`. There is no pinned version, meaning the latest available Flask will be installed during the image build. In production workloads, always pin dependency versions (e.g., `flask==3.0.3`) to ensure reproducible builds.

Verify the server script exists with correct permissions:

```bash
[steve@stapp02 ~]$ ls -la /python_app/src/server.py
```

**Output:**

```
-rw-r--r-- 1 root root 278 Mar 25 20:46 /python_app/src/server.py
```

> **Screenshot**

<img width="1032" height="600" alt="image" src="https://github.com/user-attachments/assets/4eddd673-8f3a-43c2-b63b-c09f9985ec87" />

> `cat output of requirements.txt showing flask dependency`

---

### Step 4: Author the Dockerfile

Navigate to the application root and create the `Dockerfile`.

```bash
[steve@stapp02 ~]$ cd /python_app
```

#### First Attempt (Failed -- Permission Denied)

```bash
[steve@stapp02 python_app]$ cat > /python_app/Dockerfile << 'EOF'
FROM python:latest
...
EOF
```

**Error Output:**

```
-bash: /python_app/Dockerfile: Permission denied
```

> **Root Cause:** Since `/python_app/` is owned by `root` and `steve` lacks write access, a direct shell redirect (`>`) fails. The shell attempts to open the target file for writing under the `steve` user context before executing any command, and the OS denies it immediately.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing Permission denied error when attempting to write Dockerfile as steve]`

#### Resolution -- Escalate with sudo tee

Use `sudo tee` to write the file as root, redirecting the heredoc content through `tee` with elevated privileges:

```bash
[steve@stapp02 python_app]$ sudo tee /python_app/Dockerfile > /dev/null << 'EOF'
FROM python:latest

WORKDIR /python_app/src

COPY src/requirements.txt .

RUN pip install -r requirements.txt

EXPOSE 8089

COPY src/ .

CMD ["python", "server.py"]
EOF
```

> **How it works:** `sudo tee` runs with root privileges and handles the file write. Redirecting `tee`'s standard output to `/dev/null` suppresses the content echo to the terminal. The heredoc feeds the Dockerfile content to `tee`'s standard input.

Enter the `steve` sudo password when prompted.

#### Verify the Dockerfile Was Written Correctly

```bash
[steve@stapp02 python_app]$ cat /python_app/Dockerfile
```

**Expected Output:**

```dockerfile
FROM python:latest

WORKDIR /python_app/src

COPY src/requirements.txt .

RUN pip install -r requirements.txt

EXPOSE 8089

COPY src/ .

CMD ["python", "server.py"]
```

**Dockerfile Breakdown:**

| Instruction | Purpose |
|---|---|
| `FROM python:latest` | Sets the official Python image as the base layer |
| `WORKDIR /python_app/src` | Sets the working directory inside the container |
| `COPY src/requirements.txt .` | Copies only the dependency file first to leverage layer caching |
| `RUN pip install -r requirements.txt` | Installs Flask inside the image |
| `EXPOSE 8089` | Documents the port the application listens on |
| `COPY src/ .` | Copies all application source files into the working directory |
| `CMD ["python", "server.py"]` | Defines the default command to start the Flask server |

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing cat /python_app/Dockerfile with complete Dockerfile contents verified]`

---

### Step 5: Build the Docker Image

Build the image from `/python_app/` (the directory containing the `Dockerfile`), tagging it as `nautilus/python-app`.

```bash
[steve@stapp02 python_app]$ sudo docker build -t nautilus/python-app .
```

**Build Output (summarized):**

```
[+] Building 33.0s (10/10) FINISHED                         docker:default
 => [internal] load build definition from Dockerfile         0.2s
 => [internal] load metadata for docker.io/library/python:latest  1.5s
 => [1/5] FROM docker.io/library/python:latest@sha256:ffeb... 22.0s
 => [2/5] WORKDIR /python_app/src                            0.1s
 => [3/5] COPY src/requirements.txt .                        0.1s
 => [4/5] RUN pip install -r requirements.txt                3.9s
 => [5/5] COPY src/ .                                        0.1s
 => exporting to image                                       4.7s
 => naming to docker.io/nautilus/python-app                  0.0s
```

> **Observation:** The total build time was approximately 33 seconds. The majority of time was consumed pulling the `python:latest` base image layers (22 seconds) and installing pip packages (3.9 seconds). Subsequent builds with no changes to `requirements.txt` will be significantly faster due to Docker layer caching.

> **Screenshot Placeholder**
> `[SCREENSHOT: Full docker build output showing all 10 build steps completing with FINISHED status]`

---

### Step 6: Run the Container

Launch a detached container from the built image, mapping host port `8095` to container port `8089`.

```bash
[steve@stapp02 python_app]$ sudo docker run -d \
  --name pythonapp_nautilus \
  -p 8095:8089 \
  nautilus/python-app
```

**Output:**

```
4e38cef2bf71d8a40698c2cb51cd6aacfd27cdb2b81aac5f5612478d1478d322
```

> The long hex string is the full container ID returned by the Docker daemon, confirming the container was successfully created and started in detached mode.

**Flag Reference:**

| Flag | Description |
|---|---|
| `-d` | Run container in detached (background) mode |
| `--name pythonapp_nautilus` | Assign a human-readable name to the container |
| `-p 8095:8089` | Map host port 8095 to container port 8089 (`host:container`) |

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing docker run command with the full container ID returned on success]`

---

### Step 7: Verify the Deployment

#### Confirm the Container Is Running

```bash
[steve@stapp02 python_app]$ sudo docker ps | grep pythonapp_nautilus
```

**Output:**

```
4e38cef2bf71   nautilus/python-app   "python server.py"   29 seconds ago   Up 28 seconds   0.0.0.0:8095->8089/tcp, :::8095->8089/tcp   pythonapp_nautilus
```

> **Key fields to verify:**
> - Status shows `Up 28 seconds` confirming the container is healthy and running
> - Port mapping shows `0.0.0.0:8095->8089/tcp` confirming traffic on all interfaces on host port 8095 is forwarded to container port 8089
> - Container name is `pythonapp_nautilus` as required

#### Test Application Endpoint with curl

```bash
[steve@stapp02 python_app]$ curl http://localhost:8095/
```

**Output:**

```
Welcome to xFusionCorp Industries!
```

> The Flask application is live, responding to HTTP requests, and serving the expected response body. The deployment is complete and validated.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing docker ps grep output with Up status and port mapping, followed by curl response showing Welcome to xFusionCorp Industries!]`

---

## Errors Encountered and Resolutions

| # | Step | Error | Root Cause | Resolution |
|---|---|---|---|---|
| 1 | SSH Connection | Host key authenticity warning | First-time connection to `stapp02` from jump host | Typed `yes` to accept and permanently add the ED25519 fingerprint to `known_hosts` |
| 2 | Dockerfile Creation | `-bash: /python_app/Dockerfile: Permission denied` | `/python_app/` is owned by `root`; `steve` has no write permission on the directory | Replaced the shell redirect (`cat >`) with `sudo tee /python_app/Dockerfile > /dev/null` to write the file with elevated privileges |

---

## Best Practices Applied

### Dockerfile Layer Caching Optimization

The `requirements.txt` is copied and `pip install` is executed **before** copying the full application source. This ensures that as long as dependencies do not change, Docker reuses the cached pip installation layer on subsequent builds, dramatically reducing rebuild time.

```dockerfile
# Correct order -- cache pip install layer
COPY src/requirements.txt .
RUN pip install -r requirements.txt
COPY src/ .
```

### Exec Form for CMD

The `CMD` instruction uses the exec form (`["python", "server.py"]`) rather than the shell form (`python server.py`). The exec form runs the process directly without a shell wrapper, allowing the application process to receive OS signals (SIGTERM, SIGINT) properly for graceful shutdown.

### Port Exposure Documentation

`EXPOSE 8089` is included to document the container's listening port. While `EXPOSE` does not publish the port (that is done at runtime with `-p`), it communicates intent to operators and tooling.

### Privilege Separation with sudo tee

Rather than running the entire session as root, `sudo` was scoped specifically to the `tee` command for the file write operation. This follows the principle of least privilege -- only escalating when strictly necessary.

### Container Naming

The container was given an explicit `--name pythonapp_nautilus` rather than relying on Docker's randomly generated names. Named containers are easier to manage in scripts, CI/CD pipelines, and operational tooling.

---

## Lessons Learned

**1. Always audit directory ownership before writing files.**
Running `ls -la` on the parent directory before attempting any file creation would have revealed the `root` ownership of `/python_app/` and avoided the `Permission denied` error entirely. Add this as a pre-check habit.

**2. `sudo tee` is the correct pattern for privileged file writes in restricted directories.**
Shell redirects (`>`, `>>`) are processed by the invoking shell under the user's own permissions, before the command runs. `sudo tee` correctly elevates the write operation. This is a foundational Linux privilege escalation pattern that every DevOps engineer must internalize.

**3. The SSH host key prompt is a first-connection artifact.**
In automated pipelines (CI/CD, Ansible, Terraform), this prompt will block execution. Use `StrictHostKeyChecking=no` for transient lab environments or pre-populate `known_hosts` via configuration management for production environments.

**4. Unpinned base images and dependencies create non-reproducible builds.**
`FROM python:latest` and `flask` (with no version) will produce different images at different points in time. For production workloads, pin both:

```dockerfile
FROM python:3.12.3-slim
```

```
flask==3.0.3
```

**5. Validate application response immediately after container startup.**
Using `curl` right after `docker run` confirmed end-to-end connectivity: host networking, port mapping, Flask startup, and HTTP response correctness. This is the minimal viable smoke test for any web application container.

**6. Use `docker ps | grep` for targeted container health checks.**
Piping `docker ps` output through `grep` allows precise targeting of a specific container by name, giving a quick operational view of status, uptime, and port bindings.

---




<img width="1038" height="619" alt="image" src="https://github.com/user-attachments/assets/5e43fea8-42bc-4cf1-a778-0bae09f3d3aa" />
<img width="1032" height="808" alt="image" src="https://github.com/user-attachments/assets/c82903a4-7263-4544-9b34-999ae03677a5" />
<img width="1031" height="866" alt="image" src="https://github.com/user-attachments/assets/e0e2135f-d652-4c4b-b75f-073a2f9d8362" />
<img width="1028" height="834" alt="image" src="https://github.com/user-attachments/assets/9ad00123-6e9a-4c4f-9447-373ad33cff0c" />
<img width="1036" height="866" alt="image" src="https://github.com/user-attachments/assets/2559d680-a243-4746-ba7a-669df2d354af" />
<img width="1036" height="849" alt="image" src="https://github.com/user-attachments/assets/f8e9e95a-7c89-4909-913b-bdbb1cd8abe9" />
<img width="1037" height="854" alt="image" src="https://github.com/user-attachments/assets/bccaf754-f742-4500-b79b-7915b5d74e31" />
<img width="1034" height="596" alt="image" src="https://github.com/user-attachments/assets/7916afdd-6309-44ae-86f9-420608602162" />
