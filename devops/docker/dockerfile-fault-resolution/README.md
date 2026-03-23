# Fix Broken Dockerfile and Build Apache HTTPD Docker Image with SSL on Custom Port

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Apache HTTPD](https://img.shields.io/badge/Apache-CA2136?style=for-the-badge&logo=apache&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=for-the-badge)

---

## Table of Contents

* [Overview](#overview)
* [Infrastructure Details](#infrastructure-details)
* [Problem Statement](#problem-statement)
* [Root Cause Analysis](#root-cause-analysis)
* [Prerequisites](#prerequisites)
* [Resolution Steps](#resolution-steps)
  * [Step 1 - Access Application Server 2](#step-1---access-application-server-2)
  * [Step 2 - Inspect the Broken Dockerfile](#step-2---inspect-the-broken-dockerfile)
  * [Step 3 - Escalate to Root](#step-3---escalate-to-root)
  * [Step 4 - Fix the Dockerfile Instructions](#step-4---fix-the-dockerfile-instructions)
  * [Step 5 - Verify the Fixed Dockerfile](#step-5---verify-the-fixed-dockerfile)
  * [Step 6 - Build the Docker Image](#step-6---build-the-docker-image)
  * [Step 7 - Verify the Built Image](#step-7---verify-the-built-image)
  * [Step 8 - Investigate Port 8080 Conflict](#step-8---investigate-port-8080-conflict)
  * [Step 9 - Identify the Process Blocking Port 8080](#step-9---identify-the-process-blocking-port-8080)
* [Validation and Testing](#validation-and-testing)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Overview

This runbook documents the end-to-end resolution of a broken `Dockerfile` on **Application Server 2 (`stapp02`)** within the Nautilus DevOps infrastructure at Stratos DC. The Dockerfile contained invalid instructions that prevented the Docker image from being built. The resolution involved identifying malformed Dockerfile directives, correcting them using `sed` in-place substitution, successfully building the image tagged as `nautilus-image`, and investigating a pre-existing port conflict on `8080` caused by a `ttyd` terminal process.

> **Scope:** This task was performed on `stapp02` (Application Server 2) accessed via the jump host. No changes were made to the base image, SSL certificates, HTML content, or any other valid configuration within the Dockerfile.

---

## Infrastructure Details

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| Jump Host Server | `jump-host` | `thor` | Secure entry point to Stratos DC |
| Application Server 2 | `stapp02` | `steve` | Target server hosting the broken Dockerfile |

> **Note:** All actions originated from the jump host (`jump-host`) as user `thor` and were executed on `stapp02` as user `steve`, then escalated to `root`.

---

## Problem Statement

The Nautilus DevOps team placed a `Dockerfile` at `/opt/docker/` on `stapp02` to build an Apache HTTPD image with the following configurations:

* Listen on port `8080` instead of the default `80`
* Enable the SSL module
* Enable the `socache_shmcb` module required for SSL session caching
* Include the SSL virtual host configuration
* Copy custom SSL certificates (`server.crt`, `server.key`)
* Copy a custom `index.html`

**The Dockerfile failed to build** because critical Dockerfile instructions were replaced with incorrect keywords. Specifically:

* `FROM` was written as `IMAGE`
* `RUN` was written as `ADD`

These are not valid Dockerfile directives, causing the Docker build process to fail immediately at parse time.

---

## Root Cause Analysis

| Broken Keyword | Correct Dockerfile Instruction | Impact |
|---|---|---|
| `IMAGE` | `FROM` | Docker could not identify the base image; build aborted at layer 0 |
| `ADD` (before shell commands) | `RUN` | `ADD` is a copy directive, not a shell execution directive; all `sed` config commands were invalid |

The `ADD` instruction in Dockerfile syntax is reserved for copying files or extracting archives into the image, not for executing shell commands. Using `ADD sed -i ...` is syntactically invalid and would cause a build failure. The correct instruction for executing shell commands during image build is `RUN`.

---

## Prerequisites

* SSH access to the jump host as `thor`
* SSH access to `stapp02` as `steve` with sudo privileges
* Docker installed and running on `stapp02`
* The `/opt/docker/` directory must contain:
  * `Dockerfile`
  * `certs/server.crt`
  * `certs/server.key`
  * `html/index.html`

---

## Resolution Steps

### Step 1 - Access Application Server 2

From the jump host, SSH into `stapp02` using the credentials for user `steve`.

```bash
thor@jump-host ~$ ssh steve@stapp02
```

Confirm you are on the correct host and user:

```bash
[steve@stapp02 ~]$ hostname
stapp02

[steve@stapp02 ~]$ whoami
steve
```

***Screenshot Placeholder: Terminal showing successful SSH connection to stapp02 with hostname and whoami output***

---

### Step 2 - Inspect the Broken Dockerfile

Navigate to the Docker working directory and inspect the directory structure and the Dockerfile content.

```bash
[steve@stapp02 ~]$ cd /opt/docker
[steve@stapp02 docker]$ ls -la /opt/docker
```

**Expected directory structure:**

```
total 24
drwxrwxrwx 4 root root 4096 Mar 23 02:00 .
drwxr-xr-x 1 root root 4096 Mar 23 02:00 ..
-rw-r--r-- 1 root root  518 Mar 23 02:00 Dockerfile
drwxr-xr-x 2 root root 4096 Mar 23 02:00 certs
drwxr-xr-x 2 root root 4096 Mar 23 02:00 html
```

Examine the broken Dockerfile:

```bash
[steve@stapp02 docker]$ cat /opt/docker/Dockerfile
```

**Broken Dockerfile content (before fix):**

```dockerfile
IMAGE httpd:2.4.43

ADD sed -i "s/Listen 80/Listen 8080/g" /usr/local/apache2/conf/httpd.conf

ADD sed -i '/LoadModule\ ssl_module modules\/mod_ssl.so/s/^#//g' conf/httpd.conf

ADD sed -i '/LoadModule\ socache_shmcb_module modules\/mod_socache_shmcb.so/s/^#//g' conf/httpd.conf

ADD sed -i '/Include\ conf\/extra\/httpd-ssl.conf/s/^#//g' conf/httpd.conf

COPY certs/server.crt /usr/local/apache2/conf/server.crt

COPY certs/server.key /usr/local/apache2/conf/server.key

COPY html/index.html /usr/local/apache2/htdocs/
```

***Screenshot Placeholder: Terminal showing the broken Dockerfile content with IMAGE and ADD directives highlighted***

**Identified Issues:**

1. Line 1: `IMAGE` must be `FROM`
2. Lines 3, 5, 7, 9: `ADD sed` must be `RUN sed` (4 occurrences)

---

### Step 3 - Escalate to Root

The Dockerfile at `/opt/docker/Dockerfile` is owned by `root`. Privilege escalation is required to modify it.

```bash
[steve@stapp02 docker]$ sudo su -
[sudo] password for steve:
[root@stapp02 ~]# cd /opt/docker
```

***Screenshot Placeholder: Terminal showing successful sudo su - escalation and navigation to /opt/docker***

---

### Step 4 - Fix the Dockerfile Instructions

Use `sed` in-place substitution to correct both invalid directives without manually editing the file. This approach is reproducible, auditable, and minimizes human error.

**Fix 1: Replace `IMAGE` with `FROM`**

```bash
[root@stapp02 docker]# sed -i 's/^IMAGE /FROM /' /opt/docker/Dockerfile
```

> This targets only lines starting with `IMAGE ` (anchored with `^`) to avoid unintentional substitutions elsewhere in the file.

**Fix 2: Replace all `ADD sed` occurrences with `RUN sed`**

```bash
[root@stapp02 docker]# sed -i 's/^ADD sed/RUN sed/g' /opt/docker/Dockerfile
```

> The `g` flag ensures all matching lines across the file are corrected in a single pass. The `^` anchor ensures only lines starting with `ADD sed` are affected.

***Screenshot Placeholder: Terminal showing both sed commands executed successfully with no errors***

---

### Step 5 - Verify the Fixed Dockerfile

Confirm the Dockerfile now contains valid instructions before attempting the build.

```bash
[root@stapp02 docker]# cat -n /opt/docker/Dockerfile
```

**Fixed Dockerfile content (after fix):**

```dockerfile
     1  FROM httpd:2.4.43

     3  RUN sed -i "s/Listen 80/Listen 8080/g" /usr/local/apache2/conf/httpd.conf

     5  RUN sed -i '/LoadModule\ ssl_module modules\/mod_ssl.so/s/^#//g' conf/httpd.conf

     7  RUN sed -i '/LoadModule\ socache_shmcb_module modules\/mod_socache_shmcb.so/s/^#//g' conf/httpd.conf

     9  RUN sed -i '/Include\ conf\/extra\/httpd-ssl.conf/s/^#//g' conf/httpd.conf

    11  COPY certs/server.crt /usr/local/apache2/conf/server.crt

    13  COPY certs/server.key /usr/local/apache2/conf/server.key

    15  COPY html/index.html /usr/local/apache2/htdocs/
```

**Validation checklist:**

* `FROM httpd:2.4.43` - valid base image declaration
* `RUN sed` (4 lines) - valid shell execution for config modification
* `COPY` (3 lines) - valid file copy instructions
* No `IMAGE` or `ADD sed` directives remain

***Screenshot Placeholder: Terminal output of cat -n showing the corrected Dockerfile with line numbers***

---

### Step 6 - Build the Docker Image

Build the Docker image, tagging it as `nautilus-image`, using the corrected Dockerfile and the `/opt/docker/` build context. Output is simultaneously displayed and captured to a log file for auditability.

```bash
[root@stapp02 docker]# docker build -t nautilus-image /opt/docker/ 2>&1 | tee /tmp/build.log
```

**Expected build output (all 8 layers must complete successfully):**

```
#0 building with "default" instance using docker driver

#1 [internal] load build definition from Dockerfile
#2 [internal] load metadata for docker.io/library/httpd:2.4.43
#3 [internal] load .dockerignore
#4 [1/8] FROM docker.io/library/httpd:2.4.43@sha256:cd88fee4...
#5 [internal] load build context
#6 [2/8] RUN sed -i "s/Listen 80/Listen 8080/g" ...
#7 [3/8] RUN sed -i '/LoadModule ssl_module ...'
#8 [4/8] RUN sed -i '/LoadModule socache_shmcb_module ...'
#9 [5/8] RUN sed -i '/Include conf/extra/httpd-ssl.conf ...'
#10 [6/8] COPY certs/server.crt ...
#11 [7/8] COPY certs/server.key ...
#12 [8/8] COPY html/index.html ...
#13 exporting to image ... naming to docker.io/library/nautilus-image
```

> All 8 build steps completed without errors. The image was successfully written with SHA256 digest `75f3dc558b36...`.

***Screenshot Placeholder: Full docker build output showing all 8 layers completing with DONE status and final image naming***

---

### Step 7 - Verify the Built Image

Confirm the `nautilus-image` is present in the local Docker image registry.

```bash
[root@stapp02 docker]# docker images | grep nautilus-image
```

**Expected output:**

```
nautilus-image   latest    75f3dc558b36   3 minutes ago   166MB
```

***Screenshot Placeholder: Terminal showing docker images output with nautilus-image listed***

---

### Step 8 - Investigate Port 8080 Conflict

A test container run was attempted to validate the image. The container failed to start due to a port binding conflict on `8080`.

```bash
[root@stapp02 docker]# docker run -d --name test-container -p 8080:8080 nautilus-image
```

**Error received:**

```
docker: Error response from daemon: driver failed programming external connectivity on endpoint
test-container: Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use.
```

Standard network diagnostic tools (`ss`, `netstat`, `fuser`) were unavailable in this minimal environment. The investigation was performed using only `/proc` filesystem entries.

**Check which process owns port 8080 (decimal 8080 = hex 0x1F90):**

```bash
[root@stapp02 docker]# grep -r "1F90" /proc/net/tcp /proc/net/tcp6 2>/dev/null
```

**Extract the inode number from the matching TCP entry:**

```bash
[root@stapp02 docker]# grep "1F90" /proc/net/tcp | awk '{print $10}'
2932112603
```

**Scan all process file descriptors to match the socket inode:**

```bash
[root@stapp02 docker]# for pid in /proc/[0-9]*/fd; do
    ls -la $pid 2>/dev/null | grep "2932112603" && echo "PID: $(echo $pid | cut -d'/' -f3)"
done
```

**Output:**

```
lrwx------ 1 root root 64 Mar 23 02:28 11 -> socket:[2932112603]
PID: 66
```

***Screenshot Placeholder: Terminal showing the for loop output identifying PID 66 as the owner of socket inode 2932112603***

---

### Step 9 - Identify the Process Blocking Port 8080

With PID 66 confirmed as the owner, retrieve the process name and full command line.

```bash
[root@stapp02 docker]# cat /proc/66/comm
ttyd

[root@stapp02 docker]# cat /proc/66/cmdline | tr '\0' ' '
/usr/bin/ttyd -p 8080 --ping-interval 30 -t fontSize=16 ...

[root@stapp02 docker]# cat /proc/66/status | grep -E "^Name|^State|^Pid"
Name:   ttyd
State:  S (sleeping)
Pid:    66
```

***Screenshot Placeholder: Terminal output showing PID 66 is ttyd process bound to port 8080 with full cmdline***

**Finding:** Port `8080` is occupied by `ttyd` (a web-based terminal daemon) running as PID `66` on the host. This is a pre-existing platform process and was not terminated, as it is outside the scope of this task. The primary deliverable (building `nautilus-image`) was completed successfully.

---

## Validation and Testing

| Validation Step | Command | Expected Result | Status |
|---|---|---|---|
| Confirm target server | `hostname` and `whoami` | `stapp02` / `steve` | Passed |
| Dockerfile broken directives confirmed | `cat /opt/docker/Dockerfile` | `IMAGE` and `ADD sed` visible | Confirmed |
| Fix `IMAGE` to `FROM` | `sed -i 's/^IMAGE /FROM /'` | No error, silent success | Passed |
| Fix `ADD sed` to `RUN sed` | `sed -i 's/^ADD sed/RUN sed/g'` | No error, silent success | Passed |
| Dockerfile valid after fix | `cat -n /opt/docker/Dockerfile` | All instructions valid | Passed |
| Docker image build completes | `docker build -t nautilus-image` | All 8 layers `DONE` | Passed |
| Image exists in registry | `docker images grep nautilus-image` | `nautilus-image latest 166MB` | Passed |
| Port conflict identified | `/proc/net/tcp` + socket inode lookup | PID 66 (`ttyd`) on port 8080 | Identified |

---

## Best Practices

### Dockerfile Authoring

* **Always validate Dockerfile syntax locally** before committing to version control. Use `docker build --check` (Docker 24+) or a linter such as `hadolint` to catch directive errors early.
* **Use `FROM` to declare base images** and `RUN` exclusively for shell command execution. Never confuse `ADD` or `COPY` for command execution directives.
* **Anchor `sed` patterns with `^`** when replacing line-starting keywords to avoid unintended substitutions in multiline files.
* **Minimize `RUN` layers** by chaining `sed` commands with `&&` to reduce image layer count and overall image size.
* **Prefer `COPY` over `ADD`** for simple file transfers. Reserve `ADD` only for URL fetching or automatic archive extraction, where its behavior differs from `COPY`.

### Port Management

* **Audit host ports before deploying containers** using `ss -tlnp` or `netstat -tlnp`. In minimal environments, fall back to `/proc/net/tcp` and socket inode lookups.
* **Convert port numbers to hex** when querying `/proc/net/tcp` (e.g., `8080` = `0x1F90`). The kernel stores local port addresses in little-endian hex format in this file.
* **Document pre-existing port bindings** in your environment inventory so that container port mappings are planned without conflicts.

### Operational Security

* **Operate with the minimum required privileges.** Perform read-only inspection as the application user (`steve`) and escalate to `root` only for file modification operations.
* **Capture build logs** with `tee` to retain an auditable artifact: `docker build 2>&1 | tee /tmp/build.log`.
* **Do not modify base images, SSL certificates, or application data** unless explicitly within scope. Limit changes strictly to what is broken.

---

## Lessons Learned

### 1. Invalid Dockerfile Directives Cause Silent-Looking But Fatal Build Failures

The use of `IMAGE` instead of `FROM` and `ADD` instead of `RUN` are syntactically invalid directives. Docker's build parser will either reject the build immediately or misinterpret the instruction, leading to failures that may appear confusing without a clear error message. Engineers must memorize the canonical set of Dockerfile instructions and validate them before any build attempt.

### 2. `sed` with Anchored Patterns Is a Safe, Idempotent Fix Mechanism

Using `sed -i 's/^IMAGE /FROM /'` is preferable to manual editing in production environments because it is repeatable, scriptable, and does not introduce whitespace or encoding errors that manual edits risk. The `^` anchor prevents over-broad substitutions.

### 3. Minimal Environments Require `/proc` Filesystem Knowledge

Standard tools like `ss`, `netstat`, and `fuser` are often absent in containerized or minimal Linux environments. Engineers must be comfortable navigating `/proc/net/tcp` and `/proc/<pid>/fd` using only `grep`, `awk`, and shell loops to diagnose network issues without external tooling.

### 4. Port 8080 Conflicts Are Common in Web Terminal Environments

`ttyd` is a common web-based terminal daemon that binds to `8080` by default. In Stratos DC lab environments, `ttyd` provides the browser-based terminal. Always check for pre-existing platform services on commonly used ports (`8080`, `8443`, `3000`, `9090`) before attempting to bind containers to those ports.

### 5. Scope Discipline Prevents Collateral Damage

The task required fixing the Dockerfile and building the image. It did not require stopping `ttyd` or modifying the host network. Staying within the defined scope avoids introducing new problems and keeps the change surface minimal and auditable.

---

## References

* [Dockerfile Reference - Official Docker Documentation](https://docs.docker.com/engine/reference/builder/)
* [Docker `FROM` Instruction](https://docs.docker.com/engine/reference/builder/#from)
* [Docker `RUN` Instruction](https://docs.docker.com/engine/reference/builder/#run)
* [Docker `COPY` vs `ADD`](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#add-or-copy)
* [Apache HTTPD 2.4 SSL Configuration](https://httpd.apache.org/docs/2.4/ssl/ssl_howto.html)
* [Linux `/proc/net/tcp` Format Reference](https://www.kernel.org/doc/html/latest/networking/proc_net_tcp.html)
* [ttyd - Web-based Terminal](https://github.com/tsl0922/ttyd)
* [hadolint - Dockerfile Linter](https://github.com/hadolint/hadolint)

---

> **Disclaimer:** This documents a specific incident resolution. Infrastructure details such as IP addresses, passwords, and hostnames are environment-specific and must be updated per deployment context. Do not commit credentials to version control.





<img width="1035" height="481" alt="image" src="https://github.com/user-attachments/assets/7fb18cd8-b7ae-4e00-9baf-6b7f5ef3a7de" />
<img width="1031" height="516" alt="image" src="https://github.com/user-attachments/assets/0903dbd0-a764-4db6-bce4-2044b0b6b1bc" />
<img width="1031" height="794" alt="image" src="https://github.com/user-attachments/assets/7ae23057-2961-4560-ab34-28f43fb68469" />
<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/cf17cd4e-b740-4f16-8238-59d2c9d0c2e2" />
<img width="1033" height="732" alt="image" src="https://github.com/user-attachments/assets/fce1ed3c-0516-40e5-8fff-512ce675d8bb" />
<img width="1039" height="387" alt="image" src="https://github.com/user-attachments/assets/b87daacd-26cb-4491-a9bb-ac2536dbd5af" />
<img width="1034" height="859" alt="image" src="https://github.com/user-attachments/assets/bafede78-6375-4169-9a14-14917d6f2764" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/1f289a07-8c76-492c-8c22-7d47b376892f" />
<img width="1033" height="317" alt="image" src="https://github.com/user-attachments/assets/64140f7d-9fdd-428f-87dc-ec9f87ea56cc" />
<img width="1033" height="232" alt="image" src="https://github.com/user-attachments/assets/3aa91f2d-9397-4724-809e-e674dea0a042" />
<img width="1032" height="305" alt="image" src="https://github.com/user-attachments/assets/584a75a7-0875-4a11-bce4-3e1f9dcd3366" />
<img width="1032" height="348" alt="image" src="https://github.com/user-attachments/assets/3f14bcdd-3f85-4e13-953b-f58160c96183" />
<img width="1033" height="839" alt="image" src="https://github.com/user-attachments/assets/31f43bee-c290-4e35-9db2-d63cde17efc9" />
<img width="1032" height="858" alt="image" src="https://github.com/user-attachments/assets/f6dd1214-9d57-4030-ab84-70d56fa634c1" />
<img width="1027" height="861" alt="image" src="https://github.com/user-attachments/assets/01979849-999d-41af-87cf-0474320c3457" />
<img width="1032" height="347" alt="image" src="https://github.com/user-attachments/assets/741e2d9e-b397-45ff-9656-c39aadf13570" />
<img width="1394" height="428" alt="image" src="https://github.com/user-attachments/assets/cca1d0d3-2c18-4043-8ffd-aecbb970ff0f" />
<img width="1175" height="554" alt="image" src="https://github.com/user-attachments/assets/74c6f8cc-2779-4640-bc5d-af43115f11b2" />


