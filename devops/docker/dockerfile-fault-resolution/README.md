# Fix Broken Dockerfile and Build Apache HTTPD Docker Image with SSL on Custom Port

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Apache HTTPD](https://img.shields.io/badge/Apache-CA2136?style=for-the-badge&logo=apache&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Stratos DC](https://img.shields.io/badge/Stratos%20DC-Nautilus-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=for-the-badge)

---

## Table of Contents

* [Overview](#overview)
* [Infrastructure Details](#infrastructure-details)
* [Problem Statement](#problem-statement)
* [Root Cause Analysis](#root-cause-analysis)
* [Prerequisites](#prerequisites)
* [Complete Resolution Walkthrough](#complete-resolution-walkthrough)
  * [Step 1 - SSH from Jump Host into Application Server 2](#step-1---ssh-from-jump-host-into-application-server-2)
  * [Step 2 - Verify Target Identity](#step-2---verify-target-identity)
  * [Step 3 - Navigate to the Docker Working Directory](#step-3---navigate-to-the-docker-working-directory)
  * [Step 4 - Inspect the Directory Structure](#step-4---inspect-the-directory-structure)
  * [Step 5 - Read the Broken Dockerfile](#step-5---read-the-broken-dockerfile)
  * [Step 6 - Escalate Privileges to Root](#step-6---escalate-privileges-to-root)
  * [Step 7 - Navigate Back to the Docker Directory as Root](#step-7---navigate-back-to-the-docker-directory-as-root)
  * [Step 8 - Fix Instruction 1: Replace IMAGE with FROM](#step-8---fix-instruction-1-replace-image-with-from)
  * [Step 9 - Fix Instruction 2: Replace ADD sed with RUN sed](#step-9---fix-instruction-2-replace-add-sed-with-run-sed)
  * [Step 10 - Verify the Fixed Dockerfile with Line Numbers](#step-10---verify-the-fixed-dockerfile-with-line-numbers)
  * [Step 11 - Build the Docker Image](#step-11---build-the-docker-image)
  * [Step 12 - Verify the Built Image in Local Registry](#step-12---verify-the-built-image-in-local-registry)
  * [Step 13 - Attempt to Run a Test Container (FAILED - Port Conflict)](#step-13---attempt-to-run-a-test-container-failed---port-conflict)
  * [Step 14 - Confirm No Running Container Was Created](#step-14---confirm-no-running-container-was-created)
  * [Step 15 - Attempt ss to Find Port Owner (FAILED - Tool Not Found)](#step-15---attempt-ss-to-find-port-owner-failed---tool-not-found)
  * [Step 16 - Remove the Failed Container](#step-16---remove-the-failed-container)
  * [Step 17 - Attempt netstat to Find Port Owner (FAILED - Tool Not Found)](#step-17---attempt-netstat-to-find-port-owner-failed---tool-not-found)
  * [Step 18 - Attempt fuser to Find Port Owner (FAILED - Tool Not Found)](#step-18---attempt-fuser-to-find-port-owner-failed---tool-not-found)
  * [Step 19 - Fall Back to /proc/net/tcp Hex Search](#step-19---fall-back-to-procnettcp-hex-search)
  * [Step 20 - Attempt Overly Broad ls -la Grep Against All PIDs (Noisy, Unusable Output)](#step-20---attempt-overly-broad-ls--la-grep-against-all-pids-noisy-unusable-output)
  * [Step 21 - Isolate the Socket Inode Number Cleanly](#step-21---isolate-the-socket-inode-number-cleanly)
  * [Step 22 - Loop Through All PIDs to Match the Socket Inode](#step-22---loop-through-all-pids-to-match-the-socket-inode)
  * [Step 23 - Attempt to Read cmdline via Dynamic Path Construction (FAILED - Wrong Field Extracted)](#step-23---attempt-to-read-cmdline-via-dynamic-path-construction-failed---wrong-field-extracted)
  * [Step 24 - Read Process Name from /proc/66/comm](#step-24---read-process-name-from-proc66comm)
  * [Step 25 - Read Full Command Line from /proc/66/cmdline](#step-25---read-full-command-line-from-proc66cmdline)
  * [Step 26 - Confirm Process Status via /proc/66/status](#step-26---confirm-process-status-via-proc66status)
  * [Step 27 - Exit Root Shell and Disconnect from stapp02](#step-27---exit-root-shell-and-disconnect-from-stapp02)
* [Error Catalogue](#error-catalogue)
* [Validation Matrix](#validation-matrix)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Overview

This runbook is the complete, unabridged, step-by-step record of fixing a broken `Dockerfile` on **Application Server 2 (`stapp02`)** within the Nautilus DevOps infrastructure at Stratos DC. Every command executed, every error encountered, every failed tool invocation, every diagnostic dead end, and every successful resolution step is captured in exact sequence as it occurred in the live session.

The Dockerfile contained syntactically invalid instructions that prevented the Docker image from building. After fixing those instructions and successfully building the image tagged as `nautilus-image`, a test container run revealed a pre-existing port conflict on `8080`. The three standard port diagnostic tools (`ss`, `netstat`, `fuser`) were all missing from the environment, requiring a full `/proc` filesystem-based investigation. An initial broad diagnostic attempt produced noisy unusable output. A targeted PID loop then identified the offending process as `ttyd` (PID 66), the platform's browser-based terminal daemon, which is a reserved platform service outside the scope of this task.

> **Scope Constraint:** No changes were made to the base image, SSL certificates, HTML content, or any valid configuration within the Dockerfile. The task was strictly limited to fixing the broken directives and building the image.

---

## Infrastructure Details

| Server Name | Hostname | IP | User | Password | Purpose |
|---|---|---|---|---|---|
| Jump Host Server | `jump-host` | Dynamic | `thor` | `mjolnir123` | Secure entry point to Stratos DC |
| Application Server 2 | `stapp02` | `10.244.234.200` | `steve` | `Am3ric@` | Target server hosting the broken Dockerfile |

> **Security Note:** Credentials above are lab environment values from the Nautilus DevOps platform. Never store production credentials in documentation or version control.

---

## Problem Statement

The Nautilus DevOps team prepared a `Dockerfile` at `/opt/docker/` on `stapp02` to build an Apache HTTPD 2.4.43 image configured to:

* Listen on port `8080` instead of the default `80`
* Enable `mod_ssl` for HTTPS
* Enable `mod_socache_shmcb` required for SSL session caching
* Include the SSL virtual host configuration (`httpd-ssl.conf`)
* Bundle custom SSL certificates (`server.crt`, `server.key`)
* Serve a custom `index.html`

**The build was broken** because the Dockerfile author used incorrect instruction keywords. Two fundamental Dockerfile directives were wrong across five lines:

* `FROM` was written as `IMAGE` (1 occurrence)
* `RUN` was written as `ADD` on all four shell command lines (4 occurrences)

Neither `IMAGE` nor `ADD <shell-command>` are valid Dockerfile constructs, causing the Docker build to fail at parse time before any layer could execute.

---

## Root Cause Analysis

| Line | Broken Directive | Correct Directive | Why It Is Wrong |
|---|---|---|---|
| 1 | `IMAGE httpd:2.4.43` | `FROM httpd:2.4.43` | `IMAGE` is not a Dockerfile instruction. `FROM` is the only valid way to declare the base image. Without it, Docker cannot identify what to pull. |
| 3 | `ADD sed -i "s/Listen 80/..."` | `RUN sed -i "s/Listen 80/..."` | `ADD` is a file-copy directive. It accepts a source and destination path, not a shell command string. `RUN` is the only instruction for executing shell commands during a build. |
| 5 | `ADD sed -i '/LoadModule ssl_module...'` | `RUN sed -i '/LoadModule ssl_module...'` | Same as line 3. |
| 7 | `ADD sed -i '/LoadModule socache_shmcb...'` | `RUN sed -i '/LoadModule socache_shmcb...'` | Same as line 3. |
| 9 | `ADD sed -i '/Include conf/extra/httpd-ssl...'` | `RUN sed -i '/Include conf/extra/httpd-ssl...'` | Same as line 3. |

**Total broken lines: 5** (1 `FROM` equivalent, 4 `RUN` equivalents)

**Lines that were valid and must NOT be changed:**

* `COPY certs/server.crt /usr/local/apache2/conf/server.crt`
* `COPY certs/server.key /usr/local/apache2/conf/server.key`
* `COPY html/index.html /usr/local/apache2/htdocs/`

---

## Prerequisites

* SSH access to jump host (`jump-host`) as user `thor`
* SSH access from jump host to `stapp02` as user `steve`
* `steve` must have `sudo` privileges on `stapp02`
* Docker daemon must be running on `stapp02`
* `/opt/docker/` must already contain: `Dockerfile`, `certs/server.crt`, `certs/server.key`, `html/index.html`

---

## Complete Resolution Walkthrough

### Step 1 - SSH from Jump Host into Application Server 2

From the jump host terminal as user `thor`, initiate an SSH connection to `stapp02`. Because this is the first time this jump host has connected to `stapp02`, SSH presents the host's ED25519 fingerprint and requires manual trust confirmation.

```bash
thor@jump-host ~$ ssh steve@stapp02
```

**Full output:**

```
The authenticity of host 'stapp02 (10.244.234.200)' can't be established.
ED25519 key fingerprint is SHA256:qO5NjXeeu/ayS6IAR96rfpS7j4P2wEyNmeXVC4t/zR4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
steve@stapp02's password:
```

Type `yes` to accept and permanently store the fingerprint, then enter `steve`'s password (`Am3ric@`).

> **What happened:** The jump host had no prior record of `stapp02`'s host key. Typing `yes` appended the fingerprint to `~/.ssh/known_hosts` on the jump host. On all future connections, SSH will silently verify against this cached fingerprint. Typing `no` would have aborted the connection. Supplying the fingerprint directly would have verified it before trusting.

---

### Step 2 - Verify Target Identity

Before making any changes, confirm the correct server and user are active.

```bash
[steve@stapp02 ~]$ hostname
stapp02

[steve@stapp02 ~]$ whoami
steve
```

Both outputs match expected values. Proceeding.

***Screenshot: Terminal showing hostname returning stapp02 and whoami returning steve on consecutive lines***

<img width="1035" height="481" alt="image" src="https://github.com/user-attachments/assets/7fb18cd8-b7ae-4e00-9baf-6b7f5ef3a7de" />

---

### Step 3 - Navigate to the Docker Working Directory

Change directory to `/opt/docker` where the Dockerfile and build context reside.

```bash
[steve@stapp02 ~]$ cd /opt/docker
```

No output. Prompt changes to `[steve@stapp02 docker]$` confirming the navigation.

---

### Step 4 - Inspect the Directory Structure

List all contents of `/opt/docker` with full metadata to verify all required build context files are present and to understand ownership and permissions before attempting any modification.

```bash
[steve@stapp02 docker]$ ls -la /opt/docker
```

**Output:**

```
total 24
drwxrwxrwx 4 root root 4096 Mar 23 02:00 .
drwxr-xr-x 1 root root 4096 Mar 23 02:00 ..
-rw-r--r-- 1 root root  518 Mar 23 02:00 Dockerfile
drwxr-xr-x 2 root root 4096 Mar 23 02:00 certs
drwxr-xr-x 2 root root 4096 Mar 23 02:00 html
```

**Key observations:**

* `Dockerfile` is present, 518 bytes, owned by `root:root`, permissions `-rw-r--r--`
* `steve` has read access to the Dockerfile but not write access
* `certs/` and `html/` directories are present with the expected build context assets
* The parent directory `/opt/docker` is world-writable (`drwxrwxrwx`) but individual files inside are root-owned

> **Critical finding:** `steve` cannot modify the Dockerfile. Privilege escalation to `root` will be required before applying fixes.

***Screenshot: Terminal showing the full ls -la /opt/docker output with permissions, ownership and file sizes for Dockerfile, certs and html entries***

<img width="1031" height="516" alt="image" src="https://github.com/user-attachments/assets/0903dbd0-a764-4db6-bce4-2044b0b6b1bc" />

---

### Step 5 - Read the Broken Dockerfile

Examine the full Dockerfile content to identify every broken instruction.

```bash
[steve@stapp02 docker]$ cat /opt/docker/Dockerfile
```

**Output (broken Dockerfile as found):**

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

**Broken lines confirmed:**

| Line | Instruction Found | Should Be |
|---|---|---|
| 1 | `IMAGE` | `FROM` |
| 3 | `ADD sed` | `RUN sed` |
| 5 | `ADD sed` | `RUN sed` |
| 7 | `ADD sed` | `RUN sed` |
| 9 | `ADD sed` | `RUN sed` |

***Screenshot: Terminal showing the full cat /opt/docker/Dockerfile output with the broken IMAGE keyword on the first line and all four ADD sed lines visible***


<img width="1031" height="794" alt="image" src="https://github.com/user-attachments/assets/7ae23057-2961-4560-ab34-28f43fb68469" />

---

### Step 6 - Escalate Privileges to Root

`steve` cannot write to the `root`-owned Dockerfile. Escalate to a full root login shell using `sudo su -`. The `-` flag resets the environment to root's login environment, including PATH and working directory.

```bash
[steve@stapp02 docker]$ sudo su -
```

**Full output:**

```
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for steve:
```

Enter `Am3ric@` when prompted. The prompt changes to `[root@stapp02 ~]#`.

> **Why `sudo su -` and not just `sudo`:** `sudo su -` creates a full root login shell with root's complete environment. The `~` in the resulting prompt confirms the working directory has reset to `/root`. This is intentional and documented. It means the next step must re-navigate to `/opt/docker`.

***Screenshot: Terminal showing the sudo lecture text, the password prompt, and the successful root prompt [root@stapp02 ~]#***

<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/cf17cd4e-b740-4f16-8238-59d2c9d0c2e2" />

---

### Step 7 - Navigate Back to the Docker Directory as Root

The `sudo su -` login shell reset the working directory to `/root`. Navigate back to `/opt/docker`.

```bash
[root@stapp02 ~]# cd /opt/docker
```

Prompt changes to `[root@stapp02 docker]#`.

***Screenshot Placeholder: Terminal showing the cd /opt/docker command and prompt update to [root@stapp02 docker]#***

---

### Step 8 - Fix Instruction 1: Replace `IMAGE` with `FROM`

Use `sed` in-place substitution (`-i`) to replace the invalid `IMAGE` keyword with the correct Dockerfile instruction `FROM`. The pattern is anchored to the beginning of the line with `^` and includes a trailing space to prevent unintended partial substitutions.

```bash
[root@stapp02 docker]# sed -i 's/^IMAGE /FROM /' /opt/docker/Dockerfile
```

**Output:** None. A silent exit confirms the substitution was applied without error.

> **Pattern breakdown:**
> * `^IMAGE ` - matches only lines starting exactly with `IMAGE ` (space included to separate from the argument)
> * `FROM ` - replaces with `FROM ` preserving the space before `httpd:2.4.43`
> * Without the `^` anchor, a line containing the word `IMAGE` anywhere would also be matched, risking unintended changes

***Screenshot Placeholder: Terminal showing the sed -i command for IMAGE to FROM executing silently with no error output***

---

### Step 9 - Fix Instruction 2: Replace `ADD sed` with `RUN sed`

Use a second `sed` in-place substitution to replace all four occurrences of `ADD sed` with `RUN sed` in a single pass. The `g` flag applies the substitution globally across all matching lines in the file.

```bash
[root@stapp02 docker]# sed -i 's/^ADD sed/RUN sed/g' /opt/docker/Dockerfile
```

**Output:** None. Silent success.

> **Why this is safe for `COPY` lines:** The `^ADD sed` pattern specifically matches lines starting with `ADD sed`. The three `COPY` lines begin with `COPY`, not `ADD`, and are completely unaffected. The three valid `COPY` instructions remain unchanged.

***Screenshot Placeholder: Terminal showing the sed -i command for ADD sed to RUN sed executing silently***

---

### Step 10 - Verify the Fixed Dockerfile with Line Numbers

Use `cat -n` to display the Dockerfile with line numbers to confirm all five broken instructions have been corrected and no valid instructions were inadvertently modified.

```bash
[root@stapp02 docker]# cat -n /opt/docker/Dockerfile
```

**Output (fixed Dockerfile):**

```
     1  FROM httpd:2.4.43
     2
     3  RUN sed -i "s/Listen 80/Listen 8080/g" /usr/local/apache2/conf/httpd.conf
     4
     5  RUN sed -i '/LoadModule\ ssl_module modules\/mod_ssl.so/s/^#//g' conf/httpd.conf
     6
     7  RUN sed -i '/LoadModule\ socache_shmcb_module modules\/mod_socache_shmcb.so/s/^#//g' conf/httpd.conf
     8
     9  RUN sed -i '/Include\ conf\/extra\/httpd-ssl.conf/s/^#//g' conf/httpd.conf
    10
    11  COPY certs/server.crt /usr/local/apache2/conf/server.crt
    12
    13  COPY certs/server.key /usr/local/apache2/conf/server.key
    14
    15  COPY html/index.html /usr/local/apache2/htdocs/
```

**Verification checklist:**

* Line 1: `FROM httpd:2.4.43` - valid base image declaration
* Line 3: `RUN sed -i` - valid shell execution to change port
* Line 5: `RUN sed -i` - valid shell execution to enable SSL module
* Line 7: `RUN sed -i` - valid shell execution to enable socache module
* Line 9: `RUN sed -i` - valid shell execution to include SSL config
* Lines 11, 13, 15: `COPY` - unchanged and valid
* No `IMAGE` or `ADD sed` directives remain anywhere

***Screenshot Placeholder: Terminal showing the full cat -n /opt/docker/Dockerfile output with all 15 numbered lines and all instructions showing as corrected***

---

### Step 11 - Build the Docker Image

Build the Docker image using the corrected Dockerfile. The build context is `/opt/docker/`. The image is tagged `nautilus-image`. Output is simultaneously streamed to the terminal and saved to `/tmp/build.log` via `tee`.

```bash
[root@stapp02 docker]# docker build -t nautilus-image /opt/docker/ 2>&1 | tee /tmp/build.log
```

**Full build output:**

```
#0 building with "default" instance using docker driver

#1 [internal] load build definition from Dockerfile
#1 transferring dockerfile: 556B done
#1 DONE 0.1s

#2 [internal] load metadata for docker.io/library/httpd:2.4.43
#2 DONE 1.5s

#3 [internal] load .dockerignore
#3 transferring context: 2B done
#3 DONE 0.0s

#4 [1/8] FROM docker.io/library/httpd:2.4.43@sha256:cd88fee4eab37f0d8cd04b06ef97285ca981c27b4d685f0321e65c5d4fd49357
#4 resolve docker.io/library/httpd:2.4.43@sha256:cd88fee4... 0.1s done
#4 sha256:cd88fee4... 1.86kB / 1.86kB done
#4 sha256:53729354... 1.37kB / 1.37kB done
#4 sha256:f1455599... 7.35kB / 7.35kB done
#4 sha256:bf595293... 0B / 27.09MB 0.1s
#4 sha256:3d3fecf6... 0B / 146B 0.1s
#4 sha256:b5fc3125... 0B / 10.37MB 0.1s
#4 sha256:bf595293... 11.53MB / 27.09MB 0.6s
#4 sha256:3d3fecf6... 146B / 146B 0.5s done
#4 sha256:3c61041... 0B / 24.47MB 0.6s
#4 sha256:bf595293... 19.92MB / 27.09MB 0.7s
#4 sha256:bf595293... 27.09MB / 27.09MB 0.9s done
#4 sha256:b5fc3125... 10.37MB / 10.37MB 0.9s done
#4 sha256:3c61041... 24.47MB / 24.47MB 1.3s done
#4 sha256:34b7e905... 298B / 298B 1.2s done
#4 extracting sha256:bf595293... 1.1s done
#4 extracting sha256:3d3fecf6... 0.0s done
#4 extracting sha256:b5fc3125... 0.6s done
#4 extracting sha256:3c61041... 0.8s done
#4 extracting sha256:34b7e905... 0.0s done
#4 DONE 4.4s

#6 [2/8] RUN sed -i "s/Listen 80/Listen 8080/g" /usr/local/apache2/conf/httpd.conf
#6 DONE 0.4s

#7 [3/8] RUN sed -i '/LoadModule\ ssl_module modules\/mod_ssl.so/s/^#//g' conf/httpd.conf
#7 DONE 0.3s

#8 [4/8] RUN sed -i '/LoadModule\ socache_shmcb_module modules\/mod_socache_shmcb.so/s/^#//g' conf/httpd.conf
#8 DONE 0.4s

#9 [5/8] RUN sed -i '/Include\ conf\/extra\/httpd-ssl.conf/s/^#//g' conf/httpd.conf
#9 DONE 0.4s

#10 [6/8] COPY certs/server.crt /usr/local/apache2/conf/server.crt
#10 DONE 0.1s

#11 [7/8] COPY certs/server.key /usr/local/apache2/conf/server.key
#11 DONE 0.1s

#12 [8/8] COPY html/index.html /usr/local/apache2/htdocs/
#12 DONE 0.1s

#13 exporting to image
#13 exporting layers
#13 exporting layers 2.0s done
#13 writing image sha256:75f3dc558b364ea4a579032defe94513a9e82360bd43aebfd07c6966f5151461 done
#13 naming to docker.io/library/nautilus-image 0.0s done
#13 DONE 2.1s
```

All 8 build stages completed with `DONE`. Image SHA256: `75f3dc558b364ea4a579032defe94513a9e82360bd43aebfd07c6966f5151461`. Build log captured at `/tmp/build.log`.

***Screenshot Placeholder: Full terminal output of the docker build command showing all 13 build stages from #0 through #13 with DONE on every step and the final naming to nautilus-image***

---

### Step 12 - Verify the Built Image in Local Registry

Confirm `nautilus-image` is present in the local Docker registry with the correct tag and expected size.

```bash
[root@stapp02 docker]# docker images | grep nautilus-image
```

**Output:**

```
nautilus-image   latest    75f3dc558b36   3 minutes ago   166MB
```

Image confirmed: repository `nautilus-image`, tag `latest`, ID `75f3dc558b36`, size `166MB`.

***Screenshot Placeholder: Terminal showing docker images grep output with nautilus-image latest 75f3dc558b36 3 minutes ago 166MB***

---

### Step 13 - Attempt to Run a Test Container (FAILED - Port Conflict)

A test container was launched to validate the built image end-to-end by mapping host port `8080` to container port `8080`.

```bash
[root@stapp02 docker]# docker run -d --name test-container -p 8080:8080 nautilus-image
```

**Output:**

```
53821d6a4c1937b739713fc4ba103206d76b4e13504d8b3fbaf98a776894d3f3
docker: Error response from daemon: driver failed programming external connectivity on endpoint test-container (a3cbc64120bf1fe0fdb4964b3b00b0bb8f9d043ff4a2d7a918947790dc1afb38): Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use.
```

**Error breakdown:**

* **First line:** Docker created the container and assigned ID `53821d6a4c19...` before attempting port binding
* **Second line:** The Docker daemon failed to configure host networking because `0.0.0.0:8080` was already bound by another process on the host
* **Result:** Container was created in a stopped/exited state but never started

> **Important distinction:** Docker allocates the container ID and writes container metadata before it attempts to bind host ports. The port binding failure happens after container creation. This means the container exists in `docker ps -a` (stopped state) even though it never ran, and must be explicitly removed.

***Screenshot Placeholder: Terminal showing the docker run command with the container ID on line 1 followed by the full Error response from daemon port binding error on line 2***

---

### Step 14 - Confirm No Running Container Was Created

Verify that the failed `docker run` did not produce a running container.

```bash
[root@stapp02 docker]# docker ps | grep test-container
```

**Output:**

```
(no output)
```

`docker ps` lists only containers in the running state. Empty output confirms `test-container` is not running. The container exists stopped in `docker ps -a` but is not active. The port investigation and cleanup proceed next.

***Screenshot Placeholder: Terminal showing docker ps grep test-container returning no output confirming no running container***

---

### Step 15 - Attempt `ss` to Find Port Owner (FAILED - Tool Not Found)

The first instinct was to use `ss` (socket statistics), the modern `iproute2`-based replacement for `netstat`, to identify which process holds port `8080`.

```bash
[root@stapp02 docker]# ss -tlnp | grep 8080
```

**Error:**

```
-bash: ss: command not found
```

**What happened:** `ss` is provided by the `iproute2` package. It is not installed in this minimal server environment. A different tool must be tried.

***Screenshot Placeholder: Terminal showing the ss -tlnp command returning -bash: ss: command not found***

---

### Step 16 - Remove the Failed Container

Before continuing port investigation, clean up the stopped `test-container` to avoid container name conflicts if the run is retried later.

```bash
[root@stapp02 docker]# docker rm test-container
```

**Output:**

```
test-container
```

Docker echoes the container name on successful removal. The container is now fully deleted from Docker's state.

***Screenshot Placeholder: Terminal showing docker rm test-container returning test-container as confirmation of removal***

---

### Step 17 - Attempt `netstat` to Find Port Owner (FAILED - Tool Not Found)

The second diagnostic attempt was `netstat`, the traditional BSD-era network statistics utility.

```bash
[root@stapp02 docker]# netstat -tlnp | grep 8080
```

**Error:**

```
-bash: netstat: command not found
```

**What happened:** `netstat` is part of the `net-tools` package, which is considered deprecated on modern Linux distributions and is absent from this minimal environment. A third tool must be tried.

***Screenshot Placeholder: Terminal showing netstat -tlnp returning -bash: netstat: command not found***

---

### Step 18 - Attempt `fuser` to Find Port Owner (FAILED - Tool Not Found)

The third diagnostic attempt was `fuser`, which identifies processes using files, sockets, or mount points.

```bash
[root@stapp02 docker]# fuser 8080/tcp
```

**Error:**

```
-bash: fuser: command not found
```

**What happened:** `fuser` is part of the `psmisc` package, which is also absent from this environment. With all three standard port investigation tools unavailable (`ss`, `netstat`, `fuser`), the only remaining option is the Linux `/proc` filesystem, which is always available regardless of installed packages.

***Screenshot Placeholder: Terminal showing fuser 8080/tcp returning -bash: fuser: command not found***

---

### Step 19 - Fall Back to `/proc/net/tcp` Hex Search

The Linux kernel exposes all active TCP connections in `/proc/net/tcp` (IPv4) and `/proc/net/tcp6` (IPv6). Port numbers in these files are stored in hexadecimal format.

**Port conversion:** `8080` decimal = `1F90` hexadecimal.

```bash
[root@stapp02 docker]# grep -r "1F90" /proc/net/tcp /proc/net/tcp6 2>/dev/null
```

**Output:**

```
/proc/net/tcp:   0: 00000000:1F90 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 2932112603 1 0000000000000000 100 0 0 10 0
```

**Parsing the critical fields:**

| Field | Raw Value | Decoded Meaning |
|---|---|---|
| Local address | `00000000:1F90` | `0.0.0.0:8080` - listening on all interfaces |
| Remote address | `00000000:0000` | No remote peer - this is a listening socket |
| TCP state | `0A` | Hex `0A` = decimal 10 = `TCP_LISTEN` |
| Socket inode | `2932112603` | The kernel inode for this socket |

The socket inode `2932112603` is the key. This inode number appears as a symlink target in the file descriptor directory of the owning process under `/proc/<PID>/fd/`.

***Screenshot Placeholder: Terminal showing the grep -r 1F90 /proc/net/tcp output with the full hex encoded TCP entry line***

---

### Step 20 - Attempt Overly Broad `ls -la` Grep Against All PIDs (Noisy, Unusable Output)

The first attempt to match socket inode `2932112603` to a PID used a single broad pipeline that dumped and searched all process file descriptor listings simultaneously.

```bash
[root@stapp02 docker]# ls -la /proc/*/fd 2>/dev/null | grep $(grep "1F90" /proc/net/tcp /proc/net/tcp6 2>/dev/null | awk '{print $10}' | head -1)
```

**What this attempted:**

1. `grep "1F90" /proc/net/tcp ... | awk '{print $10}' | head -1` - extract inode `2932112603`
2. `ls -la /proc/*/fd` - list every file descriptor for every process in a single dump
3. `grep 2932112603` - search the entire dump for the inode string

**Problem encountered:** The command produced pages of irrelevant output. The output of `ls -la /proc/*/fd` for all processes is an enormous stream that includes directory headers (`total 0`, `dr-x------`), timestamps, permission strings, inode numbers, and symlink targets for hundreds of processes. The pattern `2932112603` also partially matched other numeric sequences within this large dump, generating false matches across multiple unrelated process directory listings.

The output scrolled through dozens of `/proc/<PID>/fd` directory listings with no clear signal identifying which single PID owned the matching socket. This approach was abandoned as too noisy to be reliable.

***Screenshot Placeholder: Terminal showing the overly broad ls -la /proc/*/fd pipeline producing many lines of output across multiple process directories making it impossible to identify the correct PID***

---

### Step 21 - Isolate the Socket Inode Number Cleanly

Before running the targeted PID loop, extract only the socket inode number to use it as a precise search term.

```bash
[root@stapp02 docker]# grep "1F90" /proc/net/tcp | awk '{print $10}'
```

**Output:**

```
2932112603
```

The inode `2932112603` is cleanly isolated. This will now be used in a controlled per-PID loop.

***Screenshot Placeholder: Terminal showing grep 1F90 /proc/net/tcp piped to awk print 10 returning only 2932112603***

---

### Step 22 - Loop Through All PIDs to Match the Socket Inode

Use a shell loop to iterate through each process's file descriptor directory individually and look for the exact socket symlink target `socket:[2932112603]`.

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

**Loop logic explained:**

* `/proc/[0-9]*/fd` - the glob expands to every `/proc/<numeric-PID>/fd` directory, ignoring non-PID entries like `/proc/net`, `/proc/sys`, etc.
* `ls -la $pid 2>/dev/null` - lists that process's file descriptors, suppressing permission errors for protected processes
* `grep "2932112603"` - looks for the exact inode in symlink targets like `socket:[2932112603]`
* `echo "PID: $(echo $pid | cut -d'/' -f3)"` - extracts and prints the PID from the path when a match is found

**Result:** PID `66` owns file descriptor `11`, which is a socket pointing to inode `2932112603` - the same inode as the process listening on port `8080`.

***Screenshot Placeholder: Terminal showing the for loop executing and producing the single output line with socket:[2932112603] and PID: 66***

---

### Step 23 - Attempt to Read cmdline via Dynamic Path Construction (FAILED - Wrong Field Extracted)

An attempt was made to dynamically construct the cmdline path using the `ls -la` output from the grep match, rather than directly referencing the known PID.

```bash
[root@stapp02 docker]# cat /proc/$(ls -la /proc/[0-9]*/fd 2>/dev/null | grep "2932112603" | head -1 | awk '{print $NF}' | cut -d'/' -f3)/cmdline | tr '\0' ' '
```

**Error:**

```
cat: '/proc/socket:[2932112603]/cmdline': No such file or directory
```

**What went wrong - dissected:**

The `ls -la` line that matched the socket was:
```
lrwx------ 1 root root 64 Mar 23 02:28 11 -> socket:[2932112603]
```

* `awk '{print $NF}'` extracted the **last field** of this line, which was `socket:[2932112603]` (the symlink target)
* `cut -d'/' -f3` attempted to extract the third `/`-delimited field from `socket:[2932112603]`, but that string contains no `/` characters, so it returned `socket:[2932112603]` unchanged
* The constructed path became `/proc/socket:[2932112603]/cmdline` - a path that does not exist on the filesystem

> **Root cause:** `$NF` in awk gives the last whitespace-delimited token of a line. In an `ls -la` symlink line, the last token is the symlink target, not the directory path. To extract the PID from the path `/proc/<PID>/fd`, the PID must be parsed from the directory path itself, for example from `$pid` in the loop. Since PID `66` was already confirmed in Step 22, subsequent steps reference it directly.

***Screenshot Placeholder: Terminal showing the failed cat command with the awk NF extraction producing the nonsensical path /proc/socket:[2932112603]/cmdline and the No such file or directory error***

---

### Step 24 - Read Process Name from `/proc/66/comm`

With PID `66` already confirmed in Step 22, read the short process name directly from its `comm` file.

```bash
[root@stapp02 docker]# cat /proc/66/comm
```

**Output:**

```
ttyd
```

The process is named `ttyd`.

***Screenshot Placeholder: Terminal showing cat /proc/66/comm returning ttyd***

---

### Step 25 - Read Full Command Line from `/proc/66/cmdline`

Read the complete invocation command for PID `66`. In `/proc/<PID>/cmdline`, arguments are separated by null bytes (`\0`). The `tr '\0' ' '` converts these to spaces for readability.

```bash
[root@stapp02 docker]# cat /proc/66/cmdline | tr '\0' ' '
```

**Output:**

```
/usr/bin/ttyd -p 8080 --ping-interval 30 -t fontSize=16 -t theme={"foreground":"#eff0eb","background":"#282a36","cursor":"#adadad","black":"#282a36","red":"#ff5c57","green":"#5af78e","yellow":"#f3f99d","blue":"#57c7ff","magenta":"#ff6ac1","cyan":"#9aedfe","white":"#f1f1f0","brightBlack":"#686868","brightRed":"#ff5c57","brightGreen":"#5af78e","brightYellow":"#f3f99d","brightBlue":"#57c7ff","brightMagenta":"#ff6ac1","brightCyan":"#37e6e8","brightWhite":"#eff0eb"} bash -c sudo su - steve
```

**Key arguments decoded:**

| Argument | Value | Meaning |
|---|---|---|
| Binary | `/usr/bin/ttyd` | Web-based terminal multiplexer daemon |
| `-p 8080` | Port 8080 | `ttyd` is explicitly and intentionally bound to port 8080 |
| `--ping-interval 30` | 30 seconds | WebSocket keepalive ping interval |
| `-t fontSize=16` | 16 pixels | Browser terminal font size |
| `-t theme={...}` | Dracula color theme | Full JSON theme passed as a terminal option |
| `bash -c sudo su - steve` | Shell command | The shell session `ttyd` wraps and serves via the browser |

> **Critical finding:** This is the Stratos DC lab platform's own browser-based terminal service. `ttyd` is started at PID `66` early in boot and is hardcoded to port `8080`. It is the process that provides the web terminal used to interact with this server through the lab UI. Terminating it would disconnect the browser session. It is a reserved platform service and must not be killed.

***Screenshot Placeholder: Terminal showing cat /proc/66/cmdline piped to tr showing the full ttyd invocation with -p 8080 clearly visible***

---

### Step 26 - Confirm Process Status via `/proc/66/status`

Read the process status to confirm name, lifecycle state, and PID as final documentation of the port conflict root cause.

```bash
[root@stapp02 docker]# cat /proc/66/status | grep -E "^Name|^State|^Pid"
```

**Output:**

```
Name:   ttyd
State:  S (sleeping)
Pid:    66
```

**Summary of findings:**

* **Name:** `ttyd` - confirmed
* **State:** `S (sleeping)` - the process is alive, idle, and waiting for WebSocket connections
* **PID:** `66` - confirmed

Port `8080` on host `stapp02` is permanently occupied by the `ttyd` platform terminal service (PID 66). The container cannot be mapped to host port `8080` in this environment without terminating `ttyd`. As this is a platform service outside the scope of the task, no further action is taken. The primary deliverable, building `nautilus-image`, was completed successfully in Step 11.

***Screenshot Placeholder: Terminal showing cat /proc/66/status grepped output with Name ttyd, State S sleeping, and Pid 66***

---

### Step 27 - Exit Root Shell and Disconnect from stapp02

Cleanly exit the root shell and close the SSH connection.

```bash
[root@stapp02 docker]# exit
logout
[steve@stapp02 docker]$ exit
logout
Connection to stapp02 closed.
thor@jump-host ~$
```

**Session teardown sequence:**

1. First `exit` terminates the `sudo su -` root shell and returns to `steve`'s shell
2. Second `exit` closes the SSH session to `stapp02`
3. The terminal returns to `thor@jump-host ~$`, confirming full disconnection

***Screenshot Placeholder: Terminal showing the two exit commands, the two logout messages, the Connection to stapp02 closed message, and the return to the jump-host prompt***

---

## Error Catalogue

A complete, ordered record of every error and failure encountered during this session.

| # | Step | Command | Error | Root Cause | Resolution |
|---|---|---|---|---|---|
| 1 | 13 | `docker run -d --name test-container -p 8080:8080 nautilus-image` | `Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use` | Port 8080 was pre-occupied by `ttyd` (PID 66) before Docker attempted to bind it | Identified `ttyd` as an out-of-scope platform service; primary deliverable (image build) was already complete; container was removed |
| 2 | 15 | `ss -tlnp \| grep 8080` | `-bash: ss: command not found` | `iproute2` package not installed in the minimal environment | Pivoted to `netstat` |
| 3 | 17 | `netstat -tlnp \| grep 8080` | `-bash: netstat: command not found` | `net-tools` package not installed | Pivoted to `fuser` |
| 4 | 18 | `fuser 8080/tcp` | `-bash: fuser: command not found` | `psmisc` package not installed | Pivoted to `/proc/net/tcp` raw filesystem investigation |
| 5 | 20 | Broad `ls -la /proc/*/fd \| grep $(inode)` | Produced extensive noisy output with false partial matches; no clean signal | The inode number `2932112603` matched partial numeric sequences in unrelated fields across the full combined dump of all process fd listings | Isolated inode cleanly first (Step 21), then used a controlled per-PID loop (Step 22) |
| 6 | 23 | `cat /proc/$(ls ... \| awk '{print $NF}' \| cut -d'/' -f3)/cmdline` | `cat: '/proc/socket:[2932112603]/cmdline': No such file or directory` | `awk '{print $NF}'` extracted the symlink target (`socket:[2932112603]`) not the PID from the directory path; `cut -d'/' -f3` on a string with no `/` returned the entire string unchanged | Used the already confirmed PID `66` directly in Steps 24-26 |

---

## Validation Matrix

| # | Validation Step | Command | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|
| 1 | Target server confirmed | `hostname` | `stapp02` | `stapp02` | Passed |
| 2 | Operating user confirmed | `whoami` | `steve` | `steve` | Passed |
| 3 | Build context files present | `ls -la /opt/docker` | Dockerfile, certs/, html/ | All present | Passed |
| 4 | Broken directives identified | `cat /opt/docker/Dockerfile` | `IMAGE` and `ADD sed` visible | Both confirmed broken | Confirmed |
| 5 | Root shell obtained | `sudo su -` | `[root@stapp02 ~]#` | Root shell active | Passed |
| 6 | `IMAGE` corrected to `FROM` | `sed -i 's/^IMAGE /FROM /'` | Silent success | No output, no error | Passed |
| 7 | `ADD sed` corrected to `RUN sed` (x4) | `sed -i 's/^ADD sed/RUN sed/g'` | Silent success | No output, no error | Passed |
| 8 | Fixed Dockerfile verified | `cat -n /opt/docker/Dockerfile` | All 15 lines valid | All valid, no broken directives | Passed |
| 9 | Docker image build completes | `docker build -t nautilus-image /opt/docker/` | All 8 build stages DONE | All 8 DONE, image written | Passed |
| 10 | Image present in local registry | `docker images \| grep nautilus-image` | `nautilus-image latest 166MB` | Confirmed | Passed |
| 11 | Test container run | `docker run -d -p 8080:8080 nautilus-image` | Container running | Port conflict - FAILED | Failed (out of scope) |
| 12 | Failed container cleaned up | `docker rm test-container` | Container removed | `test-container` echoed | Passed |
| 13 | `ss` port lookup | `ss -tlnp \| grep 8080` | Process on port 8080 | `command not found` | Tool absent |
| 14 | `netstat` port lookup | `netstat -tlnp \| grep 8080` | Process on port 8080 | `command not found` | Tool absent |
| 15 | `fuser` port lookup | `fuser 8080/tcp` | Process on port 8080 | `command not found` | Tool absent |
| 16 | `/proc/net/tcp` hex match | `grep -r "1F90" /proc/net/tcp` | Entry with inode | Entry found, inode `2932112603` | Passed |
| 17 | Socket inode isolated | `awk '{print $10}'` | `2932112603` | `2932112603` | Passed |
| 18 | PID matched to inode | `for pid in /proc/[0-9]*/fd` loop | PID owning socket | PID `66` | Passed |
| 19 | Dynamic path construction | `awk '{print $NF}' \| cut -d'/'` | PID from path | `socket:[2932112603]` - wrong field | Failed |
| 20 | Process name confirmed | `cat /proc/66/comm` | Process name | `ttyd` | Passed |
| 21 | Full cmdline confirmed | `cat /proc/66/cmdline \| tr '\0' ' '` | Full invocation with `-p 8080` | `ttyd -p 8080 ...` confirmed | Passed |
| 22 | Process state confirmed | `cat /proc/66/status` | Name, State, PID | `ttyd`, `S sleeping`, `66` | Passed |

---

## Best Practices

### Dockerfile Authoring and Validation

* **Lint every Dockerfile before committing.** Run `docker run --rm -i hadolint/hadolint < Dockerfile` in your CI pipeline. `hadolint` catches invalid directives, deprecated instructions, and security issues before any build attempt. This single gate eliminates the `IMAGE`/`ADD` class of error entirely.
* **Know the 18 valid Dockerfile instructions by name.** The complete set is: `FROM`, `RUN`, `CMD`, `LABEL`, `EXPOSE`, `ENV`, `ADD`, `COPY`, `ENTRYPOINT`, `VOLUME`, `USER`, `WORKDIR`, `ARG`, `ONBUILD`, `STOPSIGNAL`, `HEALTHCHECK`, `SHELL`, `MAINTAINER`. No other keywords are valid in a Dockerfile.
* **`RUN` executes shell commands. `COPY` and `ADD` copy files.** These are distinct concerns. `ADD` does not and cannot run shell commands. If you find yourself writing `ADD <something that looks like a command>`, you mean `RUN`.
* **Anchor `sed` patterns with `^`** when targeting line-starting keywords to prevent unintended partial-match substitutions elsewhere in a config file.
* **Use `2>&1 | tee /tmp/build.log`** on every `docker build` in an operational environment to capture build output for post-incident analysis without suppressing live streaming output.
* **Consolidate multiple `RUN sed` instructions into one** using `&&` to reduce image layer count and overall image size. Each `RUN` creates a separate layer. Four separate `sed` calls create four layers that could be one.

### Port Management

* **Audit host port bindings before mapping containers.** Use `ss -tlnp` or `netstat -tlnp` where available. In minimal environments, use `grep "$(printf '%04X\n' <port)" /proc/net/tcp`.
* **Know the hex equivalents of commonly used ports:** `80` = `0050`, `443` = `01BB`, `3000` = `0BB8`, `8080` = `1F90`, `8443` = `20FB`, `9090` = `2382`.
* **Maintain a port inventory for your environment.** Document which ports are reserved by platform services (`ttyd`, monitoring agents, sidecars) so engineers do not attempt container bindings on occupied ports.

### Diagnostic Methodology in Minimal Environments

* **Attempt standard tools first, gracefully fall back.** Try `ss`, then `netstat`, then `fuser`, then `/proc`. Document each failure. This is not wasted effort - it confirms what is and is not available on the host.
* **The full `/proc/net/tcp` PID investigation pattern:**
  1. Convert port to 4-digit hex: `printf '%04X\n' 8080` returns `1F90`
  2. Find socket inode: `grep "1F90" /proc/net/tcp | awk '{print $10}'`
  3. Match inode to PID: `for pid in /proc/[0-9]*/fd; do ls -la $pid 2>/dev/null | grep "<inode>" && echo "PID: $(echo $pid | cut -d'/' -f3)"; done`
  4. Confirm process: `cat /proc/<PID>/comm` and `cat /proc/<PID>/cmdline | tr '\0' ' '`

### Privilege Management

* **Read as the application user. Write as root. Exit root immediately after.** This two-step model limits the blast radius of any accidental command executed in the elevated shell.
* **Use `sudo su -` with the `-` flag** to get a full root login environment, including a correct PATH. Using `sudo bash` or `sudo su` without `-` can produce an incomplete environment.

---

## Lessons Learned

### 1. Malformed Dockerfile Keywords Cause Parse-Time Build Failures Before Any Layer Executes

`IMAGE` is not a Dockerfile instruction and `ADD <shell-command>` is a misuse of the `ADD` directive. Docker's build parser fails immediately on encountering an unknown instruction keyword. A Dockerfile linter integrated into the CI pipeline catches this class of mistake in under one second, long before any file is copied to a server.

### 2. Three Standard Port Diagnostic Tools Were All Absent

`ss`, `netstat`, and `fuser` were all missing from `stapp02`. This is not unusual for container-optimized or hardened minimal Linux images where package footprint is deliberately minimized. Engineers must treat `/proc/net/tcp` socket inode resolution as a first-class skill, not a fallback of last resort.

### 3. Broad Grep Across Full Proc Listings Produces Unusable Noise

Running `ls -la /proc/*/fd | grep <inode>` as a single pipeline dumps the file descriptor listings of every process on the system into one stream. The inode number can partially match other numeric sequences in unrelated fields across that stream. The correct pattern is a per-PID loop that checks each process's file descriptor directory individually and stops at the first match.

### 4. `awk '{print $NF}'` Extracts the Last Token, Not the Path Component

In a line like `lrwx------ 1 root root 64 Mar 23 02:28 11 -> socket:[2932112603]`, `$NF` returns `socket:[2932112603]` (the symlink target), not the PID. When constructing a `/proc/<PID>/cmdline` path, the PID must come from the directory path string `/proc/<PID>/fd`, not from the `ls` output's content fields. The correct pattern is to use the loop variable directly (`$pid` in the loop), not to re-parse the `ls` output.

### 5. `ttyd` Permanently Occupies Port 8080 in the Stratos DC Platform

The Stratos DC lab environment uses `ttyd` as its browser-based terminal daemon, started at PID `66` at boot time, hardcoded to port `8080`. Any container mapped to host port `8080` in this environment will fail to bind. Engineers must map to an alternate host port (e.g., `-p 8081:8080`) or consult the environment's port inventory before deploying.

### 6. Docker Allocates a Container Before Binding Ports

When `docker run` fails with a port binding error, the container has already been created and registered in Docker's metadata store. It exists in a stopped state in `docker ps -a`. Always follow a failed `docker run` with `docker rm <container-name>` before retrying, or use `--rm` on the run command to auto-remove on failure.

### 7. Scope Discipline Is a Professional Obligation

The task required fixing the Dockerfile and building the image. Both were achieved. The port conflict with `ttyd` was investigated, root-caused, and documented, but `ttyd` was not terminated because doing so was outside the defined scope and would have disrupted the lab platform for all users on this server. Knowing the boundary of a task and stopping at it is as important as resolving what is inside it.

---

## References

* [Dockerfile Reference - Official Docker Documentation](https://docs.docker.com/engine/reference/builder/)
* [Docker `FROM` Instruction](https://docs.docker.com/engine/reference/builder/#from)
* [Docker `RUN` Instruction](https://docs.docker.com/engine/reference/builder/#run)
* [Docker `ADD` vs `COPY` Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#add-or-copy)
* [Apache HTTPD 2.4 SSL/TLS Configuration Guide](https://httpd.apache.org/docs/2.4/ssl/ssl_howto.html)
* [Linux Kernel `/proc/net/tcp` Format Reference](https://www.kernel.org/doc/html/latest/networking/proc_net_tcp.html)
* [Linux `proc(5)` Man Page](https://man7.org/linux/man-pages/man5/proc.5.html)
* [ttyd - Share Terminal over the Web via WebSocket](https://github.com/tsl0922/ttyd)
* [hadolint - Dockerfile Linter](https://github.com/hadolint/hadolint)
* [iproute2 `ss` Command Reference](https://man7.org/linux/man-pages/man8/ss.8.html)

---

## Author

**Nautilus DevOps Team**
Infrastructure: Stratos DC
Server: Application Server 2 (`stapp02`)
Performed by: `steve` (escalated to `root` for file modification)
Accessed via: Jump Host (`thor@jump-host`)
Date: March 23, 2026

---

> **Disclaimer:** This runbook documents a specific incident resolution within the Nautilus DevOps lab environment at Stratos DC. Infrastructure details including IP addresses, usernames, and passwords are environment-specific lab values. Never store production credentials in documentation or version control systems.








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


