# Docker Container Service Configuration: Apache2 on Custom Port

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=apache&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu_18.04-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Environment Overview](#environment-overview)
- [Architecture Diagram](#architecture-diagram)
- [Prerequisites](#prerequisites)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Step 1: SSH Into the Application Server](#step-1-ssh-into-the-application-server)
  - [Step 2: Verify the Running Container](#step-2-verify-the-running-container)
  - [Step 3: Enter the Container Shell](#step-3-enter-the-container-shell)
  - [Step 4: Update Package Lists](#step-4-update-package-lists)
  - [Step 5: Install Apache2](#step-5-install-apache2)
  - [Step 6: Reconfigure Apache to Listen on Port 8083](#step-6-reconfigure-apache-to-listen-on-port-8083)
  - [Step 7: Update the Virtual Host Configuration](#step-7-update-the-virtual-host-configuration)
  - [Step 8: Verify Configuration Changes](#step-8-verify-configuration-changes)
  - [Step 9: Start the Apache2 Service](#step-9-start-the-apache2-service)
  - [Step 10: Validate the Service is Running](#step-10-validate-the-service-is-running)
  - [Step 11: Perform an End-to-End Connectivity Test](#step-11-perform-an-end-to-end-connectivity-test)
  - [Step 12: Exit and Confirm Container Uptime](#step-12-exit-and-confirm-container-uptime)
- [Verification Summary](#verification-summary)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting Reference](#troubleshooting-reference)

---

## Problem Statement

A Nautilus DevOps team member was in the middle of configuring a running Docker container (`kkloud`) on **App Server 3** within the **Stratos Datacenter** before going on PTO. The following tasks were left incomplete and required urgent resolution:

| # | Requirement | Status at Handover |
|---|-------------|--------------------|
| 1 | Install `apache2` inside the `kkloud` container using `apt` | Incomplete |
| 2 | Configure Apache to listen on port **8083** instead of the default port 80, without binding to a specific IP or hostname | Incomplete |
| 3 | Ensure the Apache service is running inside the container and the container itself remains in a running state | Incomplete |

> **Key Constraint:** Apache must listen on all interfaces (localhost, 127.0.0.1, container IP, etc.), not on any specific IP or hostname. No `Listen` directive should reference an IP address.

---

## Environment Overview

| Property | Value |
|----------|-------|
| Jump Host | `jump-host` |
| Target Server | `stapp03` (App Server 3) |
| Server IP | `10.244.195.52` |
| Target User | `banner` |
| Container Name | `kkloud` |
| Base Image | `ubuntu:18.04` |
| Web Server | Apache2 v2.4.29 |
| Target Port | `8083` |
| Datacenter | Stratos Datacenter |

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                     Stratos Datacenter                   │
│                                                          │
│   ┌────────────┐       SSH        ┌──────────────────┐   │
│   │ Jump Host  │ ───────────────► │  stapp03         │   │
│   │            │                  │  (App Server 3)  │   │
│   └────────────┘                  │                  │   │
│                                   │  ┌────────────┐  │   │
│                                   │  │  Docker    │  │   │
│                                   │  │  ┌───────┐ │  │   │
│                                   │  │  │kkloud │ │  │   │
│                                   │  │  │Ubuntu │ │  │   │
│                                   │  │  │18.04  │ │  │   │
│                                   │  │  │Apache2│ │  │   │
│                                   │  │  │:8083  │ │  │   │
│                                   │  │  └───────┘ │  │   │
│                                   │  └────────────┘  │   │
│                                   └──────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

---

## Prerequisites

Before beginning, ensure the following conditions are met:

- SSH access to the jump host is established and credentials are available
- The `banner` user on `stapp03` has `sudo` privileges
- Docker is installed and the daemon is active on `stapp03`
- The container `kkloud` (based on `ubuntu:18.04`) is already running
- Network access from the jump host to `stapp03` on port 22 is open

---

## Resolution Walkthrough

### Step 1: SSH Into the Application Server

From the jump host, establish an SSH session into App Server 3 using the `banner` user account.

```bash
ssh banner@stapp03
```

When prompted about host authenticity, accept by typing `yes`. This permanently adds the host fingerprint to the known hosts file.

```
The authenticity of host 'stapp03 (10.244.195.52)' can't be established.
ED25519 key fingerprint is SHA256:aUJsvcsZb7NNupGt3HPdOQIflM8WjbbZJg+0E5uvfxY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
```

> **SCREENSHOT**

<img width="1030" height="421" alt="image" src="https://github.com/user-attachments/assets/4c45703f-d00a-4be2-a9d0-626e57306ea1" />

> *Caption: Successful SSH connection from jump host to stapp03 showing the ED25519 key fingerprint acceptance prompt and the banner@stapp03 shell prompt.*

---

### Step 2: Verify the Running Container

Before entering the container, confirm that `kkloud` is running and healthy. Use `sudo docker ps` to inspect active containers.

```bash
sudo docker ps
```

**Expected output:**

```
CONTAINER ID   IMAGE          COMMAND       CREATED         STATUS         PORTS     NAMES
38927837a6a7   ubuntu:18.04   "/bin/bash"   4 minutes ago   Up 4 minutes             kkloud
```

Confirm the following before proceeding:

- `STATUS` shows `Up` (not `Exited` or `Restarting`)
- `NAMES` column shows `kkloud`
- `IMAGE` is `ubuntu:18.04`

> **SCREENSHOT**

<img width="1027" height="583" alt="image" src="https://github.com/user-attachments/assets/af183c28-43df-4351-bff9-356c9f8de191" />

> *Caption: Output of `sudo docker ps` on stapp03 confirming the kkloud container is actively running with ubuntu:18.04 as the base image.*

---

### Step 3: Enter the Container Shell

Use `docker exec` to open an interactive bash shell inside the running `kkloud` container. The `-it` flags allocate a pseudo-TTY and keep stdin open.

```bash
sudo docker exec -it kkloud bash
```

Your prompt will change from `[banner@stapp03 ~]$` to `root@<container-id>:/#`, confirming you are now operating inside the container as root.

> **SCREENSHOT**

<img width="1030" height="580" alt="image" src="https://github.com/user-attachments/assets/d427cdb1-df0d-421a-86b1-45cc5f9805ed" />

> *Caption: Terminal prompt transitioning from banner@stapp03 to root@38927837a6a7 after executing docker exec, confirming successful container entry.*

---

### Step 4: Update Package Lists

Before installing any package, refresh the `apt` package index to ensure the latest package metadata is retrieved from the Ubuntu repositories.

```bash
apt-get update -y
```

**Expected output (abbreviated):**

```
Hit:1 http://archive.ubuntu.com/ubuntu bionic InRelease
Hit:2 http://archive.ubuntu.com/ubuntu bionic-updates InRelease
Hit:3 http://archive.ubuntu.com/ubuntu bionic-backports InRelease
Hit:4 http://security.ubuntu.com/ubuntu bionic-security InRelease
Reading package lists... Done
```

All four repository hits (`bionic`, `bionic-updates`, `bionic-backports`, `bionic-security`) must succeed. A `Done` status confirms the package index is ready.

> **SCREENSHOT**

<img width="1030" height="654" alt="image" src="https://github.com/user-attachments/assets/8587326b-4c26-4fbb-a30b-b5e3f35e879b" />

> *Caption: apt-get update output inside the kkloud container showing all four Ubuntu 18.04 (Bionic) repository hits resolving successfully.*

---

### Step 5: Install Apache2

Install the `apache2` web server and all its dependencies non-interactively using the `-y` flag to auto-accept prompts.

```bash
apt-get install -y apache2
```

This command resolves and installs 24 packages totaling approximately 88.7 MB of disk space, including core dependencies such as `libapr1`, `libexpat1`, `libicu60`, `libxml2`, `perl`, `ssl-cert`, and `mime-support`.

**Key packages installed:**

| Package | Purpose |
|---------|---------|
| `apache2` | Core web server binary |
| `apache2-bin` | Apache2 compiled modules |
| `apache2-data` | Default web content and icons |
| `apache2-utils` | Utility tools (htpasswd, etc.) |
| `libicu60` | Unicode and locale support |
| `ssl-cert` | Self-signed certificate tooling |
| `perl` | Required for configuration helpers |

Upon completion, Apache2 automatically enables default modules and the `000-default` site. The service start attempt is blocked by `policy-rc.d` inside the container (expected behavior in containerized environments).

> **SCREENSHOTS**

<img width="1028" height="865" alt="image" src="https://github.com/user-attachments/assets/55557316-d79e-4756-88c7-a15fabc763ce" />
<img width="1035" height="858" alt="image" src="https://github.com/user-attachments/assets/75637c82-fd21-4ef3-a321-d1fd9e57738a" />
<img width="1033" height="861" alt="image" src="https://github.com/user-attachments/assets/e797dc05-9941-43ef-a7e0-541ea0c4d3f7" />

> *Caption: Complete apt-get install apache2 output inside kkloud container showing all 24 packages fetched, unpacked, and configured, ending with Apache module enablement messages.*

---

### Step 6: Reconfigure Apache to Listen on Port 8083

The default Apache2 installation listens on port 80. The requirement mandates port **8083** with no IP restriction. Modify `ports.conf` using an in-place `sed` substitution.

```bash
sed -i 's/Listen 80/Listen 8083/' /etc/apache2/ports.conf
```

**What this does:**
- `-i` performs an in-place edit on the file
- The substitution replaces the literal string `Listen 80` with `Listen 8083`
- This directive instructs Apache to accept connections on port 8083 on **all network interfaces**, satisfying the no-IP-binding requirement

> **SCREENSHOT**

<img width="1040" height="613" alt="image" src="https://github.com/user-attachments/assets/1fbdd482-5b06-4882-b1d9-3a2c68c6bcb0" />

> *Caption: Terminal showing the sed command execution modifying /etc/apache2/ports.conf, with no error output confirming the silent, successful in-place substitution.*

---

### Step 7: Update the Virtual Host Configuration

The default virtual host file (`000-default.conf`) contains a `<VirtualHost *:80>` directive that must also be updated to port 8083 to match `ports.conf`.

```bash
sed -i 's/<VirtualHost \*:80>/<VirtualHost *:8083>/' /etc/apache2/sites-enabled/000-default.conf
```

**Why this step is critical:**

Apache2 will fail to start (or throw a configuration mismatch warning) if `ports.conf` declares a port that no virtual host is configured to serve. Both files must be in sync.

> **SCREENSHOT**

<img width="1040" height="613" alt="image" src="https://github.com/user-attachments/assets/1fbdd482-5b06-4882-b1d9-3a2c68c6bcb0" />

> *Caption: sed command updating the VirtualHost directive in 000-default.conf from port 80 to port 8083 to ensure consistency with ports.conf.*

---

### Step 8: Verify Configuration Changes

Before starting the service, explicitly validate that both configuration changes were applied correctly.

**Verify `ports.conf`:**

```bash
grep "Listen" /etc/apache2/ports.conf
```

**Expected output:**

```
Listen 8083
        Listen 443
        Listen 443
```

**Verify `000-default.conf`:**

```bash
grep "VirtualHost" /etc/apache2/sites-enabled/000-default.conf
```

**Expected output:**

```
<VirtualHost *:8083>
</VirtualHost>
```

Both outputs confirm that:
- Port 80 has been fully replaced with 8083 in the listening directive
- SSL listeners on port 443 remain untouched (correct)
- The default virtual host now correctly maps to port 8083

> **SCREENSHOT**



> *Caption: grep outputs for both ports.conf and 000-default.conf confirming Listen 8083 and VirtualHost *:8083 are correctly set, with the 443 SSL entries remaining intact.*

---

### Step 9: Start the Apache2 Service

With configuration validated, start the Apache2 web server using the service command.

```bash
service apache2 start
```

**Expected output:**

```
 * Starting Apache httpd web server apache2
   AH00558: apache2: Could not reliably determine the server's fully qualified domain name,
   using 172.12.0.2. Set the 'ServerName' directive globally to suppress this message
 *
```

> **Note on AH00558:** This is a non-critical warning. Apache cannot resolve a fully qualified domain name for the container because no `ServerName` directive is set globally. The service starts and operates correctly. This warning can be suppressed by adding `ServerName localhost` to `/etc/apache2/apache2.conf`, but it does not impact functionality.

> **SCREENSHOT PLACEHOLDER**
> *Caption: service apache2 start output inside kkloud showing the AH00558 ServerName warning (non-critical) and the successful start confirmation with no errors.*

---

### Step 10: Validate the Service is Running

Confirm the Apache2 process is active and healthy.

```bash
service apache2 status
```

**Expected output:**

```
 * apache2 is running
```

> **SCREENSHOT PLACEHOLDER**
> *Caption: service apache2 status output showing "apache2 is running" confirming the web server process is active inside the kkloud container.*

---

### Step 11: Perform an End-to-End Connectivity Test

With the service running, validate that Apache2 is serving HTTP traffic on port 8083 using `curl`.

```bash
curl http://127.0.0.1:8083
```

**Expected output (abbreviated):**

```html
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" ...>
<html xmlns="http://www.w3.org/1999/xhtml">
  ...
  <title>Apache2 Ubuntu Default Page: It works</title>
  ...
    <div class="section_header section_header_red">
      <div id="about"></div>
      It works!
    </div>
  ...
</html>
```

The presence of the `Apache2 Ubuntu Default Page` with the `It works!` section header confirms:

- Apache2 is actively listening on port 8083
- The web server is correctly serving HTTP responses
- No firewall or binding restrictions are blocking loopback access
- The default document root at `/var/www/html/index.html` is being served correctly

> **Note:** The `ss` command (`ss -tlnp | grep 8083`) is not available in the minimal `ubuntu:18.04` image. The `curl` test serves as the definitive connectivity validation.

> **SCREENSHOT PLACEHOLDER**
> *Caption: curl http://127.0.0.1:8083 output showing the full Apache2 Ubuntu Default Page HTML response with "It works!" confirming Apache2 is correctly serving on port 8083.*

---

### Step 12: Exit and Confirm Container Uptime

Exit the container shell and verify from the host that the `kkloud` container is still in a running state. The container must not be in a stopped or exited state after configuration.

```bash
exit
```

Back on `stapp03`, run:

```bash
sudo docker ps
```

**Expected output:**

```
CONTAINER ID   IMAGE          COMMAND       CREATED          STATUS          PORTS     NAMES
38927837a6a7   ubuntu:18.04   "/bin/bash"   22 minutes ago   Up 22 minutes             kkloud
```

The `STATUS` column showing `Up 22 minutes` confirms the container remained running throughout the entire configuration process and Apache2 startup.

> **SCREENSHOT PLACEHOLDER**
> *Caption: sudo docker ps output from stapp03 after exiting the container, showing kkloud still in "Up" status with the elapsed uptime, confirming the container was not stopped or restarted.*

---

## Verification Summary

| Check | Command | Expected Result | Status |
|-------|---------|-----------------|--------|
| Container running | `sudo docker ps` | `kkloud` shows `Up` | PASS |
| Apache installed | `apt-get install -y apache2` | 24 packages installed | PASS |
| Port in ports.conf | `grep "Listen" /etc/apache2/ports.conf` | `Listen 8083` | PASS |
| Port in VirtualHost | `grep "VirtualHost" .../000-default.conf` | `<VirtualHost *:8083>` | PASS |
| Apache service up | `service apache2 status` | `apache2 is running` | PASS |
| HTTP connectivity | `curl http://127.0.0.1:8083` | Apache default page returned | PASS |
| Container still up (post-config) | `sudo docker ps` | `kkloud` shows `Up` | PASS |

---

## Best Practices

### Container Management

- **Always verify container state before entering.** Run `docker ps` to confirm the container is `Up` before executing `docker exec`. Attempting to exec into a stopped container will result in an error.
- **Use `docker exec` over `docker attach` for service configuration.** `exec` spawns a new process, whereas `attach` connects to PID 1. Exiting an attached session can unintentionally stop the container.
- **Do not use `docker run` when a container already exists.** Use `docker start` to resume a stopped container and `docker exec` to enter it.

### Apache2 Configuration

- **Always update both `ports.conf` and the VirtualHost directive together.** A mismatch between the listening port and the virtual host port is a common source of `AH00544` (Address already in use) or `AH00558` escalation errors.
- **Use `Listen <port>` without an IP address** when the requirement is to accept connections on all interfaces. Using `Listen 0.0.0.0:8083` is functionally equivalent but unnecessarily explicit. Using a specific IP (e.g., `Listen 172.12.0.2:8083`) would restrict binding and violate the task requirement.
- **Always run `service apache2 configtest`** (or `apache2ctl configtest`) before starting the service in production environments to catch syntax errors without a disruptive restart.
- **Suppress the `AH00558` warning** in persistent environments by adding `ServerName localhost` to `/etc/apache2/apache2.conf`. While non-critical, it keeps logs clean and is considered production hygiene.

### Security

- **Avoid running unnecessary services as root inside containers.** Where possible, create a dedicated service user. For containerized environments, consider using the `www-data` user that Apache creates by default.
- **Do not expose container ports publicly without explicit need.** The task used `curl` on loopback (`127.0.0.1`) for validation. Publishing ports via `-p 8083:8083` in `docker run` should only be done when external access is required.
- **Audit installed packages.** The `apt-get install apache2` command installs 24 packages. In hardened environments, review the dependency tree and remove packages not required for the service.

### Operational Hygiene

- **Document configuration changes with inline comments** in config files (e.g., `# Changed from 80 to 8083 per Jira DEVOPS-1234`) so future engineers understand the intent without archaeology.
- **Validate changes before restarting services.** Using `grep` to confirm the `sed` substitution succeeded before running `service apache2 start` prevents debugging a service that failed due to a silently incorrect config.
- **Use `curl` for HTTP validation in minimal containers** where tools like `ss`, `netstat`, or `nmap` may not be installed. `curl http://127.0.0.1:<port>` is the most portable HTTP connectivity test available.

---

## Lessons Learned

### 1. Two Files, One Port: The Dual-Config Rule

Apache2 on Ubuntu/Debian splits its port configuration across two files: `ports.conf` (the listener) and the virtual host config (the handler). Updating only one file is the single most common cause of Apache failing to start or reverting to default behavior. **Both must be updated atomically.**

### 2. `policy-rc.d` Blocks Auto-Start in Containers

During `apt-get install apache2`, the system attempted to auto-start Apache and was blocked with `invoke-rc.d: policy-rc.d denied execution of start`. This is by design in Docker containers running Ubuntu/Debian, where `policy-rc.d` prevents services from auto-starting on package install. This is expected and correct behavior. The manual `service apache2 start` step is always required in this environment.

### 3. Minimal Images Lack Standard Networking Tools

The `ubuntu:18.04` base image does not ship with `iproute2` (which provides `ss`) or `net-tools` (which provides `netstat`). Engineers must anticipate this and rely on `curl` for connectivity validation or explicitly install the required tools (`apt-get install -y iproute2`) when socket-level visibility is needed. Never assume standard Linux tooling is available inside a Docker container.

### 4. `sed -i` is Powerful but Silent on No-Match

The `sed -i 's/old/new/'` command performs an in-place substitution but does NOT error or warn if the pattern is not found. If `Listen 80` had already been changed or was typed differently (e.g., `Listen  80` with two spaces), the command would succeed with no output and no change. Always follow `sed` substitutions with a `grep` verification step to confirm the change landed as intended.

### 5. Container Uptime is Not Guaranteed Post-Configuration

Running service daemons inside a container does not inherently stop the container. The container's lifecycle is tied to PID 1 (in this case, `/bin/bash`). Because the container was started with an interactive shell as PID 1, it remained running. However, in containers where PID 1 is the service itself, restarting the service could terminate PID 1 and stop the container. Always understand what PID 1 is before modifying services in a container.

### 6. `AH00558` is a Warning, Not a Failure

The `ServerName` warning is one of the most frequently misread Apache messages by engineers new to containerized deployments. It does not prevent Apache from starting or serving requests. It is purely cosmetic but should be addressed (`ServerName localhost`) in long-running environments to avoid cluttering error logs with noise that could mask real issues.

### 7. Handover Documentation Saves Hours

This entire task was necessitated by an incomplete handover. A structured handover document noting what had been done, what remained, and what the target state was would have reduced resolution time significantly. Operational work should always be paired with state documentation, even for short-duration tasks.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| `docker exec` fails with "no such container" | Container is stopped | Run `sudo docker start kkloud` first |
| `apt-get update` hangs or fails | No network connectivity inside container | Check Docker network settings; verify DNS resolution |
| Apache fails to start, port conflict | Another process using 8083 | Run `ss -tlnp` on the host to identify the conflicting process |
| `curl` returns "Connection refused" | Apache not running or wrong port | Run `service apache2 status`; recheck `ports.conf` |
| `sed` command produces no change | Pattern not found in file | Inspect file manually with `cat`; confirm exact whitespace and content |
| Container exits after `exit` command | PID 1 was the shell session | Do not `exit` from an `attach` session; use `ctrl+p ctrl+q` to detach |
| `AH00558` persists after ServerName fix | Config file not reloaded | Run `service apache2 reload` after editing `apache2.conf` |

---








<img width="1037" height="312" alt="image" src="https://github.com/user-attachments/assets/75ccba17-05fa-4af2-970c-6d73df17e4d5" />
<img width="1036" height="331" alt="image" src="https://github.com/user-attachments/assets/ee70f85a-10e2-4ec8-a914-d8804487d31a" />
<img width="1037" height="368" alt="image" src="https://github.com/user-attachments/assets/c27c6f0a-a087-4337-8319-ffb423b1f765" />
<img width="1033" height="368" alt="image" src="https://github.com/user-attachments/assets/803f3a4b-d197-401b-a92a-d952e4fd8373" />
<img width="1030" height="871" alt="image" src="https://github.com/user-attachments/assets/2860c874-d394-488c-a739-fef6ef947feb" />
<img width="1030" height="865" alt="image" src="https://github.com/user-attachments/assets/fcad79dd-cdbb-4708-8447-4fffbdc40b7d" />
<img width="1033" height="859" alt="image" src="https://github.com/user-attachments/assets/b38f5120-9480-4fe5-bce2-06782b589db7" />
<img width="1029" height="349" alt="image" src="https://github.com/user-attachments/assets/a508c092-e2aa-47d6-977d-39aef147da22" />
