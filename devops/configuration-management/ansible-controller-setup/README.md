# Ansible Controller Provisioning on Jump Host via pip3

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Objectives](#objectives)
- [Implementation](#implementation)
  - [Step 1: Upgrade pip3](#step-1-upgrade-pip3)
  - [Step 2: Attempt Ansible Installation (PATH Failure)](#step-2-attempt-ansible-installation-path-failure)
  - [Step 3: Install Ansible via Absolute pip3 Path](#step-3-install-ansible-via-absolute-pip3-path)
  - [Step 4: Verify Ansible Binary Location](#step-4-verify-ansible-binary-location)
  - [Step 5: Validate Ansible Installation](#step-5-validate-ansible-installation)
- [Final Outcome](#final-outcome)
- [Errors and Resolutions](#errors-and-resolutions)
- [Key Decisions](#key-decisions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

This document details the end-to-end process of provisioning the **Nautilus DevOps Jump Host** as an **Ansible Controller** by installing Ansible version **4.7.0** via `pip3`. It covers dependency setup, PATH conflict resolution, binary verification, and global accessibility validation.

Ansible was selected as the configuration management tool for this environment due to its agentless architecture, low barrier to entry, and broad community support. The Jump Host serves as the centralized control plane from which all managed nodes are orchestrated.

---

## Problem Statement

The Nautilus DevOps team required a centralized Ansible Controller to begin automating infrastructure configuration across managed nodes. The Jump Host was designated as the control node. The primary constraints were:

- Ansible must be installed at a **specific version (4.7.0)** using `pip3` exclusively.
- The Ansible binary must be **globally accessible** to all system users, not scoped to a virtual environment.
- The environment had a pre-existing pip version (24.0) that required upgrading before reliable package installation.
- Post-upgrade, the `pip3` binary was installed into `/usr/local/bin`, a directory absent from `sudo`'s default `PATH`, causing command resolution failures on first use.

---

## Architecture Context

```
+----------------------------+
|        Jump Host           |
|  (Ansible Controller)      |
|                            |
|  Python 3.9.18             |
|  pip 26.0.1 (upgraded)     |
|  Ansible 4.7.0             |
|  ansible-core 2.11.12      |
|  Jinja2 3.1.6              |
|  /usr/local/bin/ansible    |
+----------------------------+
           |
           | SSH / Playbook Execution
           |
+----------+-------+---------+
|          |       |         |
| Node A   | Node B| Node C  |
| (managed hosts)            |
+----------------------------+
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Operating System | Red Hat / CentOS-based Linux |
| Python Version | 3.9.18 |
| pip (pre-upgrade) | 24.0 |
| Sudo Access | Required for system-wide installation |
| Network Access | Required for pip package downloads |
| User | `thor` (with sudo privileges) |

---

## Objectives

- Upgrade pip3 on the Jump Host to the latest stable release
- Install Ansible version 4.7.0 system-wide using pip3
- Diagnose and resolve the pip3 PATH resolution failure under sudo
- Verify that the Ansible binary is globally accessible to all users
- Confirm the correct Ansible version and runtime configuration

---

## Implementation

### Step 1: Upgrade pip3

Before installing any packages, the existing pip installation was upgraded to ensure compatibility with modern package metadata and wheel formats. Running with an outdated pip version can result in failed dependency resolution or incorrect package builds.

**Command:**

```bash
sudo pip3 install --upgrade pip
```

**What happened:**

- pip detected an existing installation at version 24.0 in `/usr/local/lib/python3.9/site-packages`
- pip 26.0.1 was downloaded (1.8 MB at 22.4 MB/s) and installed in its place
- A PATH warning was emitted indicating `/usr/local/bin` is not in the system PATH

**Warning observed (expected):**

```
WARNING: The scripts pip, pip3 and pip3.9 are installed in '/usr/local/bin'
which is not on PATH.
```

This warning is expected in this environment configuration. It does not indicate a failure. The implication is addressed in Step 2.

**Screenshot: pip3 upgrade to version 26.0.1**

<img width="1038" height="485" alt="pip3 upgrade output showing pip 26.0.1 successfully installed" src="https://github.com/user-attachments/assets/35ba1ee9-058c-40a1-9711-6b7fc1e9b67b" />

---

### Step 2: Attempt Ansible Installation (PATH Failure)

With pip3 upgraded, the standard installation command was attempted using the `pip3` alias.

**Command:**

```bash
sudo pip3 install ansible==4.7.0
```

**Error:**

```
sudo: pip3: command not found
```

**Root Cause Analysis:**

When pip was upgraded in Step 1, the new binary was placed in `/usr/local/bin`. This directory is part of the standard user `PATH` but is **excluded from sudo's secure PATH** (`/sbin:/bin:/usr/sbin:/usr/bin`) by default on Red Hat-based systems.

As a result, `sudo pip3` resolves against the restricted sudo PATH and fails to locate the binary even though it exists at `/usr/local/bin/pip3`.

**Screenshot: sudo pip3 command not found error and switch to absolute path**

<img width="1030" height="858" alt="pip3 command not found error under sudo, followed by absolute path workaround" src="https://github.com/user-attachments/assets/5a235ad7-c556-462b-9925-be48e3be8f21" />

---

### Step 3: Install Ansible via Absolute pip3 Path

To bypass the sudo PATH restriction, the installation was retried using the **fully qualified path** to the pip3 binary.

**Command:**

```bash
sudo /usr/local/bin/pip3 install ansible==4.7.0
```

**What happened:**

- Ansible 4.7.0 package (36 MB) downloaded and extracted
- `ansible-core 2.11.12` resolved and installed as the core dependency
- Additional dependencies resolved and installed:
  - `jinja2 3.1.6`
  - `MarkupSafe 3.0.3`
  - `packaging 26.0`
  - `resolvelib 0.5.4`
- Wheel files built for `ansible` and `ansible-core` from source (pyproject.toml)
- Pre-existing system packages satisfied without reinstallation:
  - `PyYAML 5.4.1`
  - `cryptography 36.0.1`
  - `cffi 1.14.5`
  - `pycparser 2.20`
  - `ply 3.11`

**Screenshot: Ansible dependencies being resolved and downloaded**

<img width="1030" height="858" alt="Ansible 4.7.0 and ansible-core 2.11.12 being downloaded via absolute pip3 path" src="https://github.com/user-attachments/assets/5a235ad7-c556-462b-9925-be48e3be8f21" />

**Screenshot: Wheel builds completing and all packages installed successfully**

<img width="1035" height="862" alt="Wheel builds for ansible and ansible-core completed, all packages installed" src="https://github.com/user-attachments/assets/c874d292-26ba-426f-82e4-3fc4eb3de4d1" />

---

### Step 4: Verify Ansible Binary Location

After installation, the Ansible binary was inspected to confirm it was placed in a system-wide executable location and carries the correct permissions for global access.

**Command:**

```bash
ls -l /usr/local/bin/ansible
```

**Output:**

```
-rwxr-xr-x 1 root root 6437 Feb 14 06:04 /usr/local/bin/ansible
```

**Interpretation:**

| Field | Value | Meaning |
|---|---|---|
| Permissions | `-rwxr-xr-x` | Owner (root) can read/write/execute; group and others can execute |
| Owner | `root` | System-level ownership confirms global install |
| Size | `6437 bytes` | Lightweight Python wrapper script |
| Path | `/usr/local/bin/ansible` | Available in the standard user PATH |

The `r-x` bits for group and others confirm that **all users on the system can execute Ansible** without requiring sudo or environment modification.

**Screenshot: Ansible binary ownership and permission verification**

<img width="1038" height="869" alt="ls -l output confirming Ansible binary owned by root with world-execute permissions" src="https://github.com/user-attachments/assets/7ce4deb1-6a7e-42ca-8290-eb81c2c05ee1" />

---

### Step 5: Validate Ansible Installation

The final validation confirmed the installed Ansible version, runtime configuration, and all key paths.

**Command:**

```bash
ansible --version
```

**Output:**

```
ansible [core 2.11.12]
  config file = None
  configured module search path = ['/home/thor/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/thor/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.18 (main, Jan 24 2024, 00:00:00) [GCC 11.4.1 20231218 (Red Hat 11.4.1-3)]
  jinja version = 3.1.6
  libyaml = True
```

**Validation Checklist:**

| Check | Expected | Result |
|---|---|---|
| Ansible version | 4.7.0 | Confirmed via `ansible [core 2.11.12]` (4.7.0 bundles this core) |
| Executable path | `/usr/local/bin/ansible` | Confirmed |
| Python runtime | 3.9.x | Python 3.9.18 confirmed |
| Jinja2 version | 3.x | Jinja2 3.1.6 confirmed |
| libyaml | True | Optimized YAML parsing active |
| Config file | None (acceptable) | No `/etc/ansible/ansible.cfg` present; defaults apply |

**Screenshot: ansible --version output confirming successful installation and runtime configuration**

<img width="1035" height="856" alt="ansible --version output showing Ansible 4.7.0 with core 2.11.12 and Python 3.9.18" src="https://github.com/user-attachments/assets/f4716738-959e-436e-81ba-764aa67ee9a6" />

---

## Final Outcome

| Outcome | Status |
|---|---|
| pip3 upgraded to 26.0.1 | Complete |
| Ansible 4.7.0 installed system-wide | Complete |
| pip3 PATH conflict resolved via absolute path | Complete |
| Ansible binary globally executable | Complete |
| Ansible version and runtime validated | Complete |
| Jump Host designated as Ansible Controller | Complete |

---

## Errors and Resolutions

| Error | Cause | Resolution |
|---|---|---|
| `sudo: pip3: command not found` | pip3 upgraded binary placed in `/usr/local/bin`, which is excluded from sudo's secure PATH | Used absolute path `sudo /usr/local/bin/pip3` to bypass PATH restriction |
| `WARNING: Running pip as the 'root' user...` | pip3 invoked with sudo instead of in a virtual environment | Acceptable for system-wide installation; no virtual environment required in this context |
| `WARNING: The scripts pip, pip3 and pip3.9 are installed in '/usr/local/bin' which is not on PATH` | sudo PATH does not include `/usr/local/bin` | Informational only; no action required for this use case |

---

## Key Decisions

| Decision | Rationale |
|---|---|
| pip3 over package manager (yum/dnf) | pip3 allows pinning to an exact version (4.7.0), which package repositories may not carry |
| System-wide install over virtual environment | All users on the Jump Host must be able to execute Ansible without environment activation |
| Absolute path for pip3 invocation | Workaround for sudo's secure PATH excluding `/usr/local/bin` on RHEL-based systems |
| Ansible 4.7.0 with ansible-core 2.11.12 | Version pinning ensures reproducibility and compatibility with existing playbooks and collections |

---

## Best Practices

- **Pin package versions** for infrastructure tooling. Using `ansible==4.7.0` ensures consistent behavior across environments and prevents unintended upgrades during re-provisioning.
- **Verify binary permissions** after pip-based system installs. Pip does not always guarantee world-executable permissions, especially in constrained environments.
- **Audit the sudo PATH** before relying on shell aliases or symlinks post-upgrade. When a binary is upgraded via pip under sudo, the new binary may land in a location outside sudo's resolution scope.
- **Avoid running pip as root in production environments** where possible. Use virtual environments or a dedicated service account. For this use case, system-wide installation was a deliberate and justified exception.
- **Confirm `libyaml = True`** in `ansible --version` output. libyaml provides significantly faster YAML parsing via C bindings and should always be active in production controller environments.
- **Create an `/etc/ansible/ansible.cfg`** as a follow-up step to define inventory paths, remote user defaults, SSH private key locations, and logging configuration before running any playbooks.

---

## Lessons Learned

- **sudo does not inherit the full user PATH.** On RHEL and CentOS systems, sudo's `secure_path` is explicitly defined in `/etc/sudoers` and typically excludes `/usr/local/bin`. Any binary installed there post-baseline will not resolve under sudo unless the PATH is updated or the absolute path is used explicitly.
- **pip upgrades under sudo can silently shift binary locations.** The pre-existing pip 24.0 was in a resolvable location, but after upgrading, pip 26.0.1 was placed in `/usr/local/bin`. This is a common failure pattern when upgrading pip in environments with restricted sudo paths.
- **`ansible --version` is a comprehensive validation tool.** It surfaces the executable path, Python version, Jinja2 version, collection and plugin search paths, and config file state in a single output, making it the most reliable post-install validation command.
- **Ansible 4.x and ansible-core versioning is distinct.** Ansible 4.7.0 ships with ansible-core 2.11.x. Understanding this two-tier versioning model is important when troubleshooting compatibility issues with collections or roles that pin against `ansible-core` directly.

---

## Tags

`ansible` `configuration-management` `devops` `automation` `pip3` `linux` `jump-host` `rhel` `ansible-core` `infrastructure`



























# Ansible Controller Setup on Jump Host (pip3)

## LAB OVERVIEW
- The Nautilus DevOps team selected Ansible as the configuration
management tool due to its simplicity and minimal prerequisites.
- The Jump Host is designated as the Ansible Controller.
- This task installs Ansible version 4.7.0 using pip3 only and ensures
the Ansible binary is globally accessible to all users.

## OBJECTIVES

- Upgrade pip3 on the Jump Host
- Install Ansible version 4.7.0 using pip3
- Resolve pip3 PATH issue
- Verify Ansible is globally executable
- Confirm correct Ansible version

## HIGH-LEVEL LOGIC

- CONNECT to Jump Host
- UPGRADE pip3
- ATTEMPT Ansible installation using pip3
- IF pip3 not found in PATH:
  -  USE absolute pip3 path
- INSTALL Ansible 4.7.0
- VERIFY Ansible binary location
- CONFIRM Ansible runs globally
- VALIDATE Ansible version

## IMPLEMENTATION STEPS

## STEP 1: UPGRADE PIP3

- COMMAND:

`sudo pip3 install --upgrade pip`

- RESULT:
- pip upgraded
- warning about /usr/local/bin not in PATH (acceptable)

## SCREENSHOT:
<img width="1038" height="485" alt="image" src="https://github.com/user-attachments/assets/35ba1ee9-058c-40a1-9711-6b7fc1e9b67b" />

## STEP 2: ATTEMPT ANSIBLE INSTALLATION (FAILURE)
COMMAND:
`sudo pip3 install ansible==4.7.0`

ERROR:
`pip3 command not found`

CAUSE:
- pip3 installed in /usr/local/bin
- sudo PATH does not include this directory

SCREENSHOT:
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/5a235ad7-c556-462b-9925-be48e3be8f21" />

## STEP 3: INSTALL ANSIBLE USING ABSOLUTE PIP3 PATH
- COMMAND:
`sudo /usr/local/bin/pip3 install ansible==4.7.0`

ACTION:
- Downloads Ansible 4.7.0
- Installs ansible-core 2.11.12
- Installs required dependencies

SCREENSHOT:
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/5a235ad7-c556-462b-9925-be48e3be8f21" />
<img width="1035" height="862" alt="image" src="https://github.com/user-attachments/assets/c874d292-26ba-426f-82e4-3fc4eb3de4d1" />

## STEP 4: VERIFY ANSIBLE BINARY LOCATION
- COMMAND:
`ls -l /usr/local/bin/ansible`

- EXPECTED RESULT:
`Executable file owned by root`
- Permissions allow execution by all users

SCREENSHOT:
<img width="1038" height="869" alt="image" src="https://github.com/user-attachments/assets/7ce4deb1-6a7e-42ca-8290-eb81c2c05ee1" />

## STEP 5: VERIFY ANSIBLE INSTALLATION
- COMMAND:
`ansible --version`

EXPECTED RESULT:
- Ansible version 4.7.0
- Executable location: /usr/local/bin/ansible
- Python version 3.9.18

SCREENSHOT:
<img width="1035" height="856" alt="image" src="https://github.com/user-attachments/assets/f4716738-959e-436e-81ba-764aa67ee9a6" />

## FINAL OUTCOME

- Ansible version 4.7.0 installed successfully
- Ansible binary available globally
- Jump Host configured as Ansible Controller
- All users can run Ansible commands

## TAGS

`ansible`
`configuration-management`
`devops`
`automation`
`pip3`
`linux`
