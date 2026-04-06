# Permanently Disabling SELinux on a Linux Application Server

> **Environment:** CentOS Stream 9 | **Target Host:** stapp01 (172.16.238.10) | **Access Method:** SSH via Jump Host

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Connect to App Server via SSH](#step-1-connect-to-app-server-via-ssh)
  - [Step 2: Install Required SELinux Packages](#step-2-install-required-selinux-packages)
  - [Step 3: Modify the SELinux Configuration File](#step-3-modify-the-selinux-configuration-file)
  - [Step 4: No Immediate Reboot Required](#step-4-no-immediate-reboot-required)
  - [Step 5: Verify SELinux Status](#step-5-verify-selinux-status)
- [Validation and Final Outcome](#validation-and-final-outcome)
- [Operational Considerations](#operational-considerations)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Troubleshooting](#troubleshooting)
- [Tags](#tags)

---

## Overview

This document provides a production-grade, step-by-step walkthrough for **permanently disabling SELinux** on a Linux application server (CentOS Stream 9) as part of a controlled security audit engagement. The process covers package installation, persistent configuration changes, and status verification without requiring an immediate system reboot.

---

## Problem Statement

xFusionCorp Industries initiated an internal security audit across their application server fleet. As part of the initial testing phase, the security team requires SELinux to be **permanently disabled** on **App Server 1 (stapp01)**. This allows testers to establish a clean baseline before evaluating the impact of SELinux policies in subsequent phases.

**Requirements:**
- SELinux must be permanently disabled (persists across reboots)
- The server must **not** be rebooted immediately
- The configuration change must be verifiable without a reboot

---

## Architecture Context

```
[thor@jumphost] --SSH--> [tony@stapp01 | 172.16.238.10]
                              CentOS Stream 9
                              SELinux Target: disabled
```

Access to the application server is performed through a designated jump host. The operator connects as user `tony` with sudo privileges on `stapp01`.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Jump host access | SSH access as `thor` on the jump host |
| Target user | `tony` on `stapp01` with sudo privileges |
| Network access | stapp01 reachable at 172.16.238.10 |
| Internet access | Required for yum package downloads |
| SELinux packages | May not be pre-installed; must be verified |

---

## Implementation Steps

### Step 1: Connect to App Server via SSH

From the jump host, initiate an SSH session to the application server using the designated user account.

```bash
ssh tony@stapp01
```

**What happens here:**
- The SSH client presents the host's ED25519 fingerprint for first-time connection verification
- The operator confirms the fingerprint and the host is added to `~/.ssh/known_hosts`
- Upon successful authentication, an interactive shell session is established as `tony@stapp01`

> **Security Note:** Always verify the host fingerprint on first connection against a known-good source to prevent man-in-the-middle attacks. The fingerprint shown here is `SHA256:+G+DqH1pprgI6E+OTKXDNp9WFXr4zB7WTes9YmKfTgM`.

**Screenshot: Establishing the SSH connection from jumphost to stapp01**

![SSH connection from jumphost to stapp01](https://github.com/user-attachments/assets/d4340256-8c8b-426d-b807-6570015e8a36)

---

### Step 2: Install Required SELinux Packages

Before modifying the SELinux configuration, ensure the necessary policy and utility packages are present on the system. These packages are required for the configuration file to be recognized and properly processed at boot time.

```bash
sudo yum install -y selinux-policy selinux-policy-targeted policycoreutils
```

**Packages installed:**

| Package | Type | Purpose |
|---|---|---|
| `policycoreutils` | x86_64 | Core SELinux policy utilities |
| `selinux-policy` | noarch | Base SELinux policy framework |
| `selinux-policy-targeted` | noarch | Targeted protection policy |
| `diffutils` | x86_64 | Dependency: file comparison utilities |
| `libselinux-utils` | x86_64 | Dependency: SELinux utility library |
| `rpm-plugin-selinux` | x86_64 | Dependency: RPM SELinux plugin |

**Total download:** 7.8 MB | **Installed size:** 21 MB

> **Operational Note:** On minimal CentOS installations, SELinux packages are frequently absent. Always confirm package presence before attempting configuration changes to avoid silent failures.

**Screenshot: Initiating the yum package installation**

![yum install selinux packages](https://github.com/user-attachments/assets/0d5526fa-ae42-423b-9b77-8af470abf278)

**Screenshot: Package download and transaction completion**

![yum transaction complete](https://github.com/user-attachments/assets/b1072b6b-359f-4fb4-b969-ab8d223a0f29)

---

### Step 3: Modify the SELinux Configuration File

Open the SELinux configuration file using a privileged text editor and set the `SELINUX` directive to `disabled`. This is the **authoritative, persistent configuration** that governs SELinux state across reboots.

```bash
sudo vi /etc/selinux/config
```

**Locate and update the following line:**

```
# Before
SELINUX=enforcing

# After
SELINUX=disabled
```

**Ensure the final configuration reads:**

```ini
SELINUX=disabled
SELINUXTYPE=targeted
```

**Key configuration values:**

| Directive | Value | Description |
|---|---|---|
| `SELINUX=enforcing` | Active enforcement | SELinux policy is enforced; violations are blocked |
| `SELINUX=permissive` | Audit mode | Violations are logged but not blocked |
| `SELINUX=disabled` | **Target state** | No SELinux policy is loaded at boot |
| `SELINUXTYPE=targeted` | Policy scope | Targeted processes are selectively protected |

> **Critical Note:** On RHEL 9 and CentOS Stream 9, setting `SELINUX=disabled` in `/etc/selinux/config` takes effect **only after a full system reboot**. The current running session will still reflect the previous SELinux state. If a fully disabled system is required before a scheduled reboot, pass `selinux=0` to the kernel via `grubby`.

**Screenshot: /etc/selinux/config with SELINUX set to disabled**

![selinux config file disabled](https://github.com/user-attachments/assets/26cb4d73-0391-4c5f-9390-b418e9caee07)

---

### Step 4: No Immediate Reboot Required

Per the engagement requirements, **the server is not rebooted at this stage.** The configuration change written to `/etc/selinux/config` is persistent and will be applied automatically on the next scheduled system restart.

**Why this is safe:**
- The `/etc/selinux/config` file is the definitive source of truth for SELinux boot state
- No active workloads are disrupted by deferring the reboot
- The change is idempotent and will survive subsequent package updates

> **Best Practice:** Document the planned reboot window and communicate it to all relevant stakeholders before proceeding. In production environments, coordinate reboot scheduling with the change management team.

**Screenshot: Configuration file saved, no reboot executed**

![no reboot performed](https://github.com/user-attachments/assets/26cb4d73-0391-4c5f-9390-b418e9caee07)

---

### Step 5: Verify SELinux Status

Confirm that the current SELinux runtime status reflects the expected state and that the configuration change has been saved correctly.

```bash
sestatus
```

**Expected output:**

```
SELinux status:                 disabled
```

> **Interpretation:** The `sestatus` command queries the kernel for the live SELinux enforcement state. A status of `disabled` confirms either that SELinux was already disabled at boot, or that the system has been rebooted since the configuration change was applied. If the system has **not yet been rebooted**, `sestatus` will reflect the pre-change state (e.g., `permissive` or `enforcing`), while the config file already contains `SELINUX=disabled`. This is expected behavior.

**Screenshot: sestatus confirming SELinux is disabled**

![sestatus output disabled](https://github.com/user-attachments/assets/6d417c93-378a-4dc2-9d1d-237124c85035)

---

## Validation and Final Outcome

| Objective | Status |
|---|---|
| SSH access to stapp01 established | **COMPLETE** |
| SELinux packages installed successfully | **COMPLETE** |
| `/etc/selinux/config` updated to `SELINUX=disabled` | **COMPLETE** |
| System reboot deferred (not performed) | **COMPLETE** |
| `sestatus` confirms disabled state | **COMPLETE** |

**The system is now configured to boot with SELinux permanently disabled**, satisfying all security audit requirements for this engagement phase.

---

## Operational Considerations

- **Change Management:** Record this change in your organization's CMDB or ticketing system. Disabling SELinux is a significant security posture change and must be tracked.
- **Rollback Path:** To re-enable SELinux, set `SELINUX=enforcing` (or `permissive`) in `/etc/selinux/config` and reboot. Note that file relabeling may be required; create a `.autorelabel` file in `/` before rebooting to trigger automatic relabeling.
- **Audit Logging:** With SELinux disabled, AVC denial logs will no longer be generated. Ensure alternative audit mechanisms (auditd, syslog) are in place.
- **Scope Limitation:** This change applies only to `stapp01`. Other servers in the fleet retain their current SELinux state unless explicitly modified.
- **Post-Reboot Verification:** After the next scheduled reboot, run `sestatus` again to confirm the disabled state persists as expected.

---

## Risks and Edge Cases

**Risk: Inadvertently widening attack surface**
Disabling SELinux removes mandatory access controls. Ensure compensating controls (firewall rules, application-level sandboxing, network segmentation) are in place.

**Risk: Reboot timing conflict**
If the server reboots unexpectedly (kernel panic, hardware issue, automated patching) before the change window, SELinux will be disabled on restart without a controlled handoff. Monitor the system during the interim period.

**Edge Case: RHEL 9+ behavior difference**
On RHEL 9 and later, `SELINUX=disabled` in the config file no longer fully disables SELinux during the boot process by default. If full kernel-level disabling is required, pass `selinux=0` to the kernel using:

```bash
sudo grubby --update-kernel ALL --args selinux=0
```

To revert:

```bash
sudo grubby --update-kernel ALL --remove-args selinux
```

**Edge Case: Missing config file**
If `/etc/selinux/config` does not exist (absent on minimal installs without the policy packages), the file will only be created after installing `selinux-policy`. This is why Step 2 (package installation) must precede Step 3 (configuration).

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `sestatus: command not found` | `policycoreutils` not installed | Run Step 2 to install SELinux packages |
| `sestatus` shows `enforcing` after config change | System not yet rebooted | Expected; reboot during next maintenance window |
| `/etc/selinux/config` not found | Packages not installed or minimal OS image | Install `selinux-policy` package first |
| `yum install` fails | No network access or repo misconfiguration | Verify network connectivity and yum repo configuration |
| Permission denied on config file | Running without sudo | Use `sudo vi /etc/selinux/config` |

---

## Tags

`linux` `selinux` `security` `system-administration` `devops` `centos` `rhel` `hardening` `audit`

































# Disable SELinux on App Server 1

## 📌 Lab Overview
- Following a security audit, xFusionCorp Industries initiated SELinux
testing on their application servers. For initial testing, SELinux
must be permanently disabled on App Server 1.

- This lab demonstrates how to safely disable SELinux without rebooting
the server immediately.

---

## 🎯 Objectives
- Install required SELinux packages
- Permanently disable SELinux via configuration
- Avoid immediate reboot
- Ensure SELinux is disabled after scheduled reboot

---

## 🧠 High-Level Logic

- CONNECT to App Server 1
- INSTALL required SELinux packages

- OPEN SELinux configuration file
- SET SELINUX=disabled
- SAVE configuration

- DO NOT reboot system
- CONFIRM configuration is applied

## 🛠️ Implementation Steps

## Step 1: Connect to App Server
- `ssh tony@stapp01`

📸 screenshot:
<img width="1031" height="556" alt="image" src="https://github.com/user-attachments/assets/d4340256-8c8b-426d-b807-6570015e8a36" />

## Step 2: Install SELinux Packages
- `sudo yum install -y selinux-policy selinux-policy-targeted policycoreutils`

📸 screenshot:
<img width="1033" height="856" alt="image" src="https://github.com/user-attachments/assets/0d5526fa-ae42-423b-9b77-8af470abf278" />
<img width="1030" height="864" alt="image" src="https://github.com/user-attachments/assets/b1072b6b-359f-4fb4-b969-ab8d223a0f29" />

## Step 3: Modify SELinux Configuration
- `sudo vi /etc/selinux/config`
- Set:

  -  SELINUX=disabled

📸 screenshot:
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/26cb4d73-0391-4c5f-9390-b418e9caee07" />

## Step 4: No Reboot Required
- Server reboot is not performed

📸 screenshot:
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/26cb4d73-0391-4c5f-9390-b418e9caee07" />

## Step 5: Status Check (Informational)
- `sestatus`

📸 screenshot:
<img width="1031" height="859" alt="image" src="https://github.com/user-attachments/assets/6d417c93-378a-4dc2-9d1d-237124c85035" />

## ✅ Final Outcome
- SELinux packages installed

- SELinux permanently disabled

- No immediate reboot performed

- System compliant with security audit requirements

## 🏷️ Tags
`linux` `selinux` `security` `system-administration` `devops`





