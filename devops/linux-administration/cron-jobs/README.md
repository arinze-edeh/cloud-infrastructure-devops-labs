# Automated Cron Job Scheduling with cronie on CentOS Stream 9

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment](#environment)
- [Architecture and High-Level Logic](#architecture-and-high-level-logic)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: SSH into Each Application Server](#step-1-ssh-into-each-application-server)
  - [Step 2: Install and Configure the Cron Service](#step-2-install-and-configure-the-cron-service)
  - [Step 3: Schedule the Root Cron Job](#step-3-schedule-the-root-cron-job)
  - [Step 4: Verify the Cron Configuration](#step-4-verify-the-cron-configuration)
- [Validation and Outcome](#validation-and-outcome)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting](#troubleshooting)
- [Tags](#tags)

---

## Overview

This document describes the end-to-end process for deploying a validated cron job across multiple application servers in a multi-tier Linux environment. The work was performed by the Nautilus system administration team as a prerequisite to rolling out production automation scripts. A sample cron job was used to verify cron daemon functionality across all application servers before committing production workloads to the scheduler.

---

## Problem Statement

The automation team required confidence that the `crond` service was correctly installed, running, and capable of executing scheduled jobs on all application servers. Without this validation step, deploying production automation directly risks silent failures where jobs appear scheduled but never execute. This exercise establishes a repeatable, verified baseline for cron-based automation across the fleet.

---

## Environment

| Component | Details |
|---|---|
| Jump Host | `jumphost` |
| App Server 1 | `stapp01` (172.16.238.10), User: `tony` |
| App Server 2 | `stapp02` (172.16.238.11), User: `steve` |
| App Server 3 | `stapp03` (172.16.238.12), User: `banner` |
| Operating System | CentOS Stream 9 |
| Cron Package | `cronie` (crond daemon) |
| Access Method | SSH via jump host |

---

## Architecture and High-Level Logic

```
Jump Host (jumphost)
     |
     |-- SSH --> stapp01 (tony) --> Install cronie --> Start/Enable crond --> Add root crontab --> Verify
     |-- SSH --> stapp02 (steve) --> Install cronie --> Start/Enable crond --> Add root crontab --> Verify
     |-- SSH --> stapp03 (banner) --> Install cronie --> Start/Enable crond --> Add root crontab --> Verify
```

The following process was repeated identically on each of the three application servers:

1. Connect via SSH from the jump host
2. Install the `cronie` package using `yum`
3. Start and enable the `crond` daemon
4. Configure a root-level cron job via `crontab`
5. Verify the scheduled job is registered correctly

---

## Prerequisites

- SSH access from the jump host to each application server
- Sudo privileges on each application server
- Network connectivity to CentOS Stream 9 repositories (BaseOS, AppStream, EPEL)
- Basic familiarity with `systemctl` and `crontab`

---

## Implementation

> **Note:** All steps below were performed individually on **each** application server (`stapp01`, `stapp02`, `stapp03`). The commands and sequence are identical across all three servers.

---

### Step 1: SSH into Each Application Server

Initiate an SSH session from the jump host to each application server using its designated service account.

```bash
# Connect to stapp01
ssh tony@stapp01

# Connect to stapp02
ssh steve@stapp02

# Connect to stapp03
ssh banner@stapp03
```

On the first connection to each server, SSH will prompt to verify the host fingerprint. Confirm by typing `yes` to permanently add the host to `~/.ssh/known_hosts`.

**Screenshot: SSH into stapp01**

![SSH into stapp01](https://github.com/user-attachments/assets/19c263a8-0787-4e9e-981a-c0b45835cfe3)

*Initial SSH session from the jump host to stapp01 using the tony service account. The host fingerprint is verified and added to known_hosts.*

**Screenshot: SSH into stapp02**

![SSH into stapp02](https://github.com/user-attachments/assets/aa017631-f3cd-49fd-bd94-e4547f322de7)

*SSH session established to stapp02 using the steve service account.*

**Screenshot: SSH into stapp03**

![SSH into stapp03](https://github.com/user-attachments/assets/58f777cb-8792-4d6a-bea3-3e193bde2489)

*SSH session established to stapp03 using the banner service account.*

---

### Step 2: Install and Configure the Cron Service

#### 2a. Install the cronie Package

The `cronie` package provides the `crond` daemon and the `crontab` utility on CentOS/RHEL systems. Install it using `yum`:

```bash
sudo yum install -y cronie
```

> **Note:** On systems where `cronie` is already installed, `yum` will upgrade the package to the latest available version rather than performing a fresh install. Both `cronie` and `cronie-anacron` are upgraded as a unit.

**Screenshots: Installing cronie on stapp01**

![Installing cronie on stapp01](https://github.com/user-attachments/assets/2f546048-df6a-4433-bb28-4366f66b7506)
![Installing cronie on stapp02](https://github.com/user-attachments/assets/58da6179-0709-467f-96fc-4f061833f44d)

*`yum install -y cronie` on stapp01. The package manager detects the existing version and triggers an upgrade to 1.5.7-16.el9.*

**Screenshot: Installing cronie on stapp02**

![crond status on stapp01](https://github.com/user-attachments/assets/7e5bc2a2-ab8f-49e0-8086-f22f134d5741)

*Package upgrade completing on stapp02. Both `cronie` and `cronie-anacron` are upgraded successfully.*

**Screenshot: Installing cronie on stapp03**

![cronie installed on stapp02](https://github.com/user-attachments/assets/3c8748f4-1a76-4898-a98c-3a8ce8ba05f1)

*Package installation initiated on stapp03 with dependency resolution and repository metadata refresh.*

#### 2b. Start and Enable the crond Service

Once the package is installed, start the daemon and configure it to start automatically on system boot:

```bash
sudo systemctl start crond
sudo systemctl enable crond
```

Enabling the service ensures persistence across reboots, which is critical for production automation workloads.

**Screenshot: crond service started and verified on stapp01**

![Installing cronie on stapp03](https://github.com/user-attachments/assets/a715db10-e659-48f9-af15-884f9d19fa0d)

*`systemctl status crond` confirms the service is active (running) and enabled on stapp01. The unit file is loaded from `/usr/lib/systemd/system/crond.service`.*

**Screenshot: cronie upgrade completion on stapp02**

<img width="1010" height="718" alt="image" src="https://github.com/user-attachments/assets/4ca7c75b-ae7e-4cf7-a485-63b0e9095f5b" />

*Upgrade transaction completes successfully on stapp02. crond is then started and enabled in the same session.*

---

### Step 3: Schedule the Root Cron Job

The cron job must be added specifically to the **root user's** crontab. This is achieved using the `-u root` flag with `crontab -e`, which opens the root user's crontab for editing regardless of which user is running the command.

```bash
sudo crontab -e -u root
```

Add the following entry in the crontab editor:

```cron
*/5 * * * * echo hello > /tmp/cron_text
```

**Cron Expression Breakdown:**

| Field | Value | Meaning |
|---|---|---|
| Minute | `*/5` | Every 5 minutes |
| Hour | `*` | Every hour |
| Day of Month | `*` | Every day |
| Month | `*` | Every month |
| Day of Week | `*` | Every day of the week |
| Command | `echo hello > /tmp/cron_text` | Write "hello" to `/tmp/cron_text` |

> **Best Practice:** Using `>` (overwrite) rather than `>>` (append) ensures the output file remains small and does not grow unbounded over time, which is important for long-running validation jobs.

**Screenshot: Root crontab configured on stapp01**

![Root crontab stapp01](https://github.com/user-attachments/assets/067a0da0-3b97-45a2-b0f2-1830500208c4)

*The crontab editor opens for the root user on stapp01. A new crontab is created and the job entry `*/5 * * * * echo hello > /tmp/cron_text` is saved.*

**Screenshot: Root crontab configured on stapp02**

![Root crontab stapp02](https://github.com/user-attachments/assets/cafad6e7-1a35-4a41-9509-41c779dcb83c)

*Root crontab entry added on stapp02. The system confirms a new crontab is being installed for root.*

**Screenshot: Root crontab configured on stapp03**

![Root crontab stapp03](https://github.com/user-attachments/assets/9401e24a-6951-4b21-a11f-f31d47906e4b)

*Root crontab entry saved on stapp03, completing the scheduled job configuration across all three servers.*

---

### Step 4: Verify the Cron Configuration

After saving the crontab, confirm the job is correctly registered by listing the root user's crontab:

```bash
sudo crontab -l -u root
```

Expected output:

```
*/5 * * * * echo hello > /tmp/cron_text
```

**Screenshot: Crontab verification on stapp01**

![Crontab verification stapp01](https://github.com/user-attachments/assets/7577a943-68d8-46c1-a55b-445a6f01d243)

*`crontab -l -u root` on stapp01 confirms the job is correctly registered. The session is then exited cleanly.*

**Screenshot: Crontab verification on stapp02**

![Crontab verification stapp02](https://github.com/user-attachments/assets/a33c92bb-3fe2-4353-9e16-04dc3eda3b76)

*Root crontab on stapp02 confirmed with the expected `*/5 * * * * echo hello > /tmp/cron_text` entry.*

**Screenshot: Crontab verification on stapp03**

![Crontab verification stapp03](https://github.com/user-attachments/assets/4d9e2505-4efe-474f-af0d-fe9a95fcc839)

*Root crontab on stapp03 verified. Configuration is now consistent and confirmed across all three application servers.*

---

## Validation and Outcome

The following criteria were confirmed on all three application servers:

- **cronie installed:** `cronie-1.5.7-16.el9.x86_64` and `cronie-anacron-1.5.7-16.el9.x86_64` upgraded and present
- **crond running:** `systemctl status crond` reports `active (running)` with enabled preset
- **Root cron job registered:** `crontab -l -u root` returns the expected `*/5 * * * * echo hello > /tmp/cron_text` entry
- **Execution target:** `/tmp/cron_text` will be populated every 5 minutes by the root-owned cron process

**Servers completed:**

| Server | User | cronie Version | crond Status | Cron Job |
|---|---|---|---|---|
| stapp01 | tony | 1.5.7-16.el9 | Active (running), enabled | Confirmed |
| stapp02 | steve | 1.5.7-16.el9 | Active (running), enabled | Confirmed |
| stapp03 | banner | 1.5.7-16.el9 | Active (running), enabled | Confirmed |

---

## Operational Considerations

**Service persistence:** Always pair `systemctl start` with `systemctl enable`. Starting a service without enabling it means it will not survive a server reboot, which would cause scheduled jobs to silently stop running after the next maintenance window.

**Root crontab vs system crontab:** Using `sudo crontab -e -u root` edits the root user's personal crontab (`/var/spool/cron/root`). This is distinct from `/etc/cron.d/` drop-in files and `/etc/crontab`. For per-user automation, always use `crontab -u <user>` to avoid inadvertently editing the wrong crontab.

**Output redirection:** The `echo hello > /tmp/cron_text` command uses overwrite redirection. For production jobs that require audit trails, consider `>>` with log rotation via `logrotate`, or redirect to a dedicated logging service.

**Containerized environments:** The systemd-related warnings observed in `systemctl status crond` output (`Failed to reset devices.allow`, `Failed to set 'trusted.invocation_id' xattr`) are expected and benign in Docker-based environments where systemd runs with reduced capabilities. They do not affect cron job execution.

**Timezone awareness:** By default, `crond` uses the system timezone (`/etc/localtime`). For multi-region deployments, standardize all servers to UTC and document timezone assumptions in each crontab entry using the `CRON_TZ` variable if needed.

---

## Troubleshooting

**crond fails to start:**
- Check for conflicting packages: `rpm -qa | grep cron`
- Review logs: `journalctl -u crond --since "10 minutes ago"`

**Cron job not executing:**
- Verify crond is running: `systemctl is-active crond`
- Check for crontab syntax errors: `crontab -l -u root` must return a valid entry with no parse errors
- Confirm the output file path is writable by root: `ls -la /tmp/`
- Review cron execution logs: `grep CRON /var/log/cron` or `journalctl -u crond`

**Permission denied errors:**
- Confirm the executing user has `sudo` privileges
- Check `/etc/sudoers` or `/etc/sudoers.d/` for user policy

**Output file not updated:**
- Confirm `/tmp` is not mounted with `noexec` flag: `mount | grep /tmp`
- Validate the cron expression using an online cron parser before deployment

---

## Tags

`linux` `cron` `cronie` `crond` `crontab` `system-administration` `automation` `centos` `rhel` `systemd` `devops` `scheduler`

























# Linux Cron Job Scheduling (cronie)

## 📌 Lab Overview
The Nautilus system administration team is preparing to deploy
automation scripts across multiple application servers. Before
scheduling production jobs, a sample cron job was configured
to validate cron functionality across all servers.

---

## 🎯 Objectives
- Install the `cronie` package
- Start and enable the `crond` service
- Schedule a recurring cron job for the root user
- Verify cron execution output

---

## 🧱 Environment

- Jump Host: jumphost

- App Servers: `stapp01` `stapp02` `stapp03`

- OS: CentOS Stream 9

- Scheduler: cron (cronie package)

## 🧠 High-Level Logic
- CONNECT to jump host
- FOR each app server:
  -  INSTALL cron service
  -  START and ENABLE cron daemon
  -  CONFIGURE root cron job
  -  VERIFY cron execution

## 🛠️ Implementation Steps
⚠️ `The following steps were performed individually on each app server.`

## Step 1: SSH into the Server
- ssh tony@stapp01
- ssh steve@stapp02
- ssh banner@stapp03

📸 screenshots:
<img width="1034" height="666" alt="image" src="https://github.com/user-attachments/assets/19c263a8-0787-4e9e-981a-c0b45835cfe3" />
<img width="1039" height="866" alt="image" src="https://github.com/user-attachments/assets/aa017631-f3cd-49fd-bd94-e4547f322de7" />
<img width="1036" height="876" alt="image" src="https://github.com/user-attachments/assets/58f777cb-8792-4d6a-bea3-3e193bde2489" />

## Step 2: Install and Start Cron
- Install the Package
  -  sudo yum install -y cronie
- Start and Enable the Service
- `Start the crond service and enable it so it persists after a reboot`
  -  sudo systemctl start crond
  -  sudo systemctl enable crond

📸 screenshots:
<img width="1036" height="851" alt="image" src="https://github.com/user-attachments/assets/2f546048-df6a-4433-bb28-4366f66b7506" />
<img width="1034" height="853" alt="image" src="https://github.com/user-attachments/assets/58da6179-0709-467f-96fc-4f061833f44d" />
<img width="1042" height="862" alt="image" src="https://github.com/user-attachments/assets/a715db10-e659-48f9-af15-884f9d19fa0d" />
<img width="1036" height="869" alt="image" src="https://github.com/user-attachments/assets/7e5bc2a2-ab8f-49e0-8086-f22f134d5741" />
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/3c8748f4-1a76-4898-a98c-3a8ce8ba05f1" />

## Step 3: Add the Cron Job
- The task requires adding a cron job specifically for the root user.
- `sudo crontab -e -u root`
- Add the Cron Entry
- `*/5 * * * * echo hello > /tmp/cron_text`

📸 screenshots:
<img width="1035" height="882" alt="image" src="https://github.com/user-attachments/assets/067a0da0-3b97-45a2-b0f2-1830500208c4" />
<img width="1038" height="890" alt="image" src="https://github.com/user-attachments/assets/cafad6e7-1a35-4a41-9509-41c779dcb83c" />
<img width="1037" height="876" alt="image" src="https://github.com/user-attachments/assets/9401e24a-6951-4b21-a11f-f31d47906e4b" />

## Step 4: Verification
- `sudo crontab -l -u root`
- `exit`

📸 screenshots:
<img width="1026" height="859" alt="image" src="https://github.com/user-attachments/assets/7577a943-68d8-46c1-a55b-445a6f01d243" />
<img width="1039" height="871" alt="image" src="https://github.com/user-attachments/assets/a33c92bb-3fe2-4353-9e16-04dc3eda3b76" />
<img width="1027" height="857" alt="image" src="https://github.com/user-attachments/assets/4d9e2505-4efe-474f-af0d-fe9a95fcc839" />

## ✅ Final Outcome
- Cron service installed and running

- Root cron job scheduled every 5 minutes

- Output successfully written to /tmp/cron_text

- Configuration applied across all app servers

## 🏷️ Tags
`linux` `cron` `cronie` `system-administration` `automation`







