# Ansible Inventory Configuration for Multi-Server Playbook Execution

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Inspect the Existing Playbook Directory](#step-1-inspect-the-existing-playbook-directory)
  - [Step 2: Review the Ansible Configuration File](#step-2-review-the-ansible-configuration-file)
  - [Step 3: Review the Playbook](#step-3-review-the-playbook)
  - [Step 4: Create the INI Inventory File](#step-4-create-the-ini-inventory-file)
  - [Step 5: Verify the Inventory File Contents](#step-5-verify-the-inventory-file-contents)
  - [Step 6: Execute the Playbook Against the Inventory](#step-6-execute-the-playbook-against-the-inventory)
- [Execution Output](#execution-output)
- [Best Practices Applied](#best-practices-applied)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)

---

## Overview

This project demonstrates the creation of an INI-format Ansible inventory file on a jump host to enable playbook execution against a target application server (`stapp02`) in the Stratos DC environment. The inventory defines the necessary connection variables for SSH authentication and privilege escalation, allowing a pre-existing Apache HTTPD installation playbook to run successfully without modification.

---

## Problem Statement

The Nautilus DevOps team maintains a collection of Ansible playbooks under `/home/thor/playbook/` on the jump host. A playbook targeting all hosts (`hosts: all`) was ready for execution against **App Server 2** (`stapp02`) in Stratos DC, but no inventory file existed to tell Ansible where to connect or how to authenticate.

Without a properly constructed inventory file, `ansible-playbook` has no host resolution context, no SSH credentials, and no privilege escalation configuration. The task required creating a compliant INI inventory file at `/home/thor/playbook/inventory` that matched the Stratos DC naming convention (e.g., `stapp02` for App Server 2) and included all variables necessary for a fully automated, non-interactive execution.

---

## Architecture and Design Intent

```
Jump Host (thor@jump-host)
        |
        |  SSH (ansible_ssh_pass)
        v
  stapp02 (App Server 2, Stratos DC)
        |
        |  Privilege Escalation (ansible_become / ansible_become_pass)
        v
  root (httpd install + service start)
```

The jump host acts as the Ansible control node. The inventory file bridges control node awareness to the managed node by embedding:

* The **hostname** (`stapp02`) aligned to the Stratos DC wiki naming convention
* The **connection target** via `ansible_host`
* The **SSH user and password** for remote authentication
* The **become configuration** for root-level task execution

The playbook itself remained untouched. Only the inventory layer was added to satisfy execution requirements.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Control Node | Jump host with Ansible installed |
| Managed Node | `stapp02` (App Server 2, Stratos DC) |
| Playbook Location | `/home/thor/playbook/playbook.yml` |
| Ansible Config | `/home/thor/playbook/ansible.cfg` |
| SSH Access | Password-based authentication to `stapp02` |
| Privilege Escalation | `sudo` available for the `steve` user on `stapp02` |

---

## Implementation Guide

### Step 1: Inspect the Existing Playbook Directory

Begin by confirming the contents of the playbook directory to understand what files already exist and what is missing.

```bash
ls -la /home/thor/playbook/
```

**Output:**

```
total 20
drwxr-xr-x 2 thor thor 4096 May  3 06:32 .
drwx------ 1 thor thor 4096 May  3 06:32 ..
-rw-r--r-- 1 thor thor   36 May  3 06:32 ansible.cfg
-rw-r--r-- 1 thor thor  250 May  3 06:32 playbook.yml
```

> Screenshot: Terminal output showing the playbook directory listing with `ansible.cfg` and `playbook.yml` present and no inventory file.

<img width="1024" height="647" alt="image" src="https://github.com/user-attachments/assets/a9d9dcfb-5cd7-46b4-8ae6-22926f5c8c98" />

The directory contains only `ansible.cfg` and `playbook.yml`. No inventory file is present. This confirms the inventory must be created before playbook execution can proceed.

---

### Step 2: Review the Ansible Configuration File

Inspect `ansible.cfg` to understand the baseline Ansible behavior configured for this environment.

```bash
cat /home/thor/playbook/ansible.cfg
```

**Output:**

```ini
[defaults]
host_key_checking = False
```

> Screenshot: Terminal output of `ansible.cfg` showing `host_key_checking = False`.

<img width="1020" height="715" alt="image" src="https://github.com/user-attachments/assets/896efc8e-900b-481e-acbc-f3248402990d" />

`host_key_checking = False` disables SSH host key verification. This is expected in controlled lab-style environments where servers are provisioned dynamically and known host fingerprints are not pre-distributed. In production environments, host key checking should remain enabled.

---

### Step 3: Review the Playbook

Inspect `playbook.yml` to understand the tasks the inventory must support and confirm the host target.

```bash
cat /home/thor/playbook/playbook.yml
```

**Output:**

```yaml
---
- hosts: all
  become: yes
  become_user: root
  tasks:
    - name: Install httpd package
      yum:
        name: httpd
        state: installed

    - name: Start service httpd
      service:
        name: httpd
        state: started
```

> Screenshot: Terminal output of `playbook.yml` showing the two tasks: HTTPD package installation and service start.

<img width="1020" height="715" alt="image" src="https://github.com/user-attachments/assets/896efc8e-900b-481e-acbc-f3248402990d" />

Key observations from the playbook:

* `hosts: all` targets every host defined in the provided inventory
* `become: yes` and `become_user: root` require privilege escalation credentials in the inventory
* The playbook uses `yum`, confirming the target OS is RHEL/CentOS-based
* No variables are passed via the CLI; all connection context must come from the inventory

---

### Step 4: Create the INI Inventory File

Create the inventory file at `/home/thor/playbook/inventory` using a heredoc to write all required connection variables in a single atomic operation.

```bash
cat > /home/thor/playbook/inventory << 'EOF'
stapp02 ansible_host=stapp02 ansible_user=steve ansible_ssh_pass=Am3ric@ ansible_become=yes ansible_become_pass=Am3ric@
EOF
```

> Screenshot: Terminal showing the heredoc command executed with no errors, returning to the prompt cleanly.

**Inventory variable breakdown:**

| Variable | Value | Purpose |
|---|---|---|
| `stapp02` | (hostname) | INI inventory hostname; matches Stratos DC naming convention |
| `ansible_host` | `stapp02` | Actual connection target (resolved via DNS or `/etc/hosts`) |
| `ansible_user` | `steve` | SSH user for App Server 2 |
| `ansible_ssh_pass` | `Am3ric@` | SSH password for the `steve` user |
| `ansible_become` | `yes` | Enables privilege escalation |
| `ansible_become_pass` | `Am3ric@` | Password for `sudo` escalation to root |

The hostname `stapp02` follows the Stratos DC wiki naming convention where App Server 1 is `stapp01` and App Server 2 is `stapp02`.

---

### Step 5: Verify the Inventory File Contents

Confirm the inventory file was written correctly before executing the playbook.

```bash
cat /home/thor/playbook/inventory
```

**Output:**

```
stapp02 ansible_host=stapp02 ansible_user=steve ansible_ssh_pass=Am3ric@ ansible_become=yes ansible_become_pass=Am3ric@
```

> Screenshot: Terminal output confirming the full inventory line is written correctly with all six variables present.

The inventory file content matches the intended configuration exactly. All variables are on a single line with no formatting errors, extraneous whitespace, or missing values.

---

### Step 6: Execute the Playbook Against the Inventory

Change into the playbook directory and run `ansible-playbook` using the newly created inventory file.

```bash
cd /home/thor/playbook/ && ansible-playbook -i inventory playbook.yml
```

> Screenshot: Full terminal output of the `ansible-playbook` run showing PLAY, TASK, and PLAY RECAP sections with all tasks reporting `ok` or `changed` and zero failures.

---

## Execution Output

```
PLAY [all] *************************************************************

TASK [Gathering Facts] *************************************************
ok: [stapp02]

TASK [Install httpd package] *******************************************
changed: [stapp02]

TASK [Start service httpd] *********************************************
changed: [stapp02]

PLAY RECAP *************************************************************
stapp02 : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

**Result interpretation:**

| Task | Status | Meaning |
|---|---|---|
| Gathering Facts | `ok` | Ansible connected to `stapp02` and collected system facts successfully |
| Install httpd package | `changed` | The `httpd` package was not previously installed and was installed during this run |
| Start service httpd | `changed` | The `httpd` service was not running and was started during this run |
| PLAY RECAP | `failed=0` | All tasks completed without error; zero unreachable hosts |

A `failed=0` and `unreachable=0` result confirms the inventory file, credentials, and privilege escalation configuration are all correct.

---

## Best Practices Applied

* **INI inventory format for simplicity:** INI-format inventories are the most portable and human-readable format for single-host or small-scope Ansible configurations. They require no additional parsing dependencies and are immediately interpretable by any team member.

* **Inline host variables for self-contained inventories:** Placing all connection variables on the host line within the inventory file keeps the execution context fully contained. This eliminates the need for external variable files for straightforward connectivity requirements.

* **Heredoc for atomic file creation:** Using `cat > file << 'EOF'` ensures the file is written in a single operation with no intermediate state. Single-quoting the EOF delimiter (`'EOF'`) prevents shell variable expansion within the heredoc body, which is critical when the content includes special characters such as `@` in passwords.

* **Verify before execute:** Explicitly reading the file back with `cat` after creation confirms the write operation succeeded and the content is exactly as intended before triggering a playbook run.

* **Naming convention compliance:** Using `stapp02` as the inventory hostname (rather than an IP address or arbitrary alias) aligns with the Stratos DC wiki naming convention. This ensures consistency across team documentation, monitoring systems, and future playbook references.

* **`become` declared at the inventory level:** Placing `ansible_become=yes` and `ansible_become_pass` in the inventory rather than the playbook keeps privilege escalation configuration decoupled from playbook logic. The playbook itself declares the intent (`become: yes`); the inventory provides the credentials. This separation supports reusability of the playbook across different environments with different escalation configurations.

---

## Errors and Resolutions

No execution errors were encountered during this implementation. The playbook completed with `failed=0` and `unreachable=0` on the first run.

The following scenarios represent common failure modes in this class of implementation and the corresponding resolutions:

| Scenario | Root Cause | Resolution |
|---|---|---|
| `UNREACHABLE` for `stapp02` | Incorrect `ansible_host` value or DNS not resolving | Verify the hostname resolves from the jump host using `ping stapp02` or use the IP address as `ansible_host` |
| `Permission denied` during SSH | Wrong `ansible_user` or `ansible_ssh_pass` | Confirm the correct username and password for the target server from the environment reference |
| `sudo: incorrect password` | Wrong `ansible_become_pass` | Confirm the escalation password matches the `sudo` password for the remote user |
| `No hosts matched` | Inventory file path incorrect in the `-i` flag | Ensure the path passed to `-i` matches the actual inventory file location |
| Special characters in password corrupting inventory | Shell expansion in heredoc | Always single-quote the heredoc delimiter (`'EOF'`) to suppress shell interpretation |

---

## Lessons Learned

* **The inventory is the control plane for connectivity.** A playbook with `hosts: all` is entirely dependent on what the inventory defines. An empty or missing inventory produces zero-host execution. Building the inventory first, then verifying its content, is a non-negotiable step before playbook execution.

* **Single-quoting heredoc delimiters prevents silent corruption.** When writing files containing special characters (passwords with `@`, `!`, `$`, etc.) via heredoc, using `'EOF'` instead of `EOF` prevents the shell from expanding embedded characters. Without this, passwords containing `$` or `!` can be silently altered, producing authentication failures that are difficult to trace.

* **`host_key_checking = False` has scope implications.** Disabling host key checking in `ansible.cfg` applies globally to all connections made from that control node. While appropriate for ephemeral or controlled environments, this setting should be treated as a scoped configuration and not carried into production Ansible control nodes where SSH fingerprint validation is a meaningful security boundary.

* **INI inventory variable syntax is whitespace-sensitive.** All variables on an INI host line must be space-separated key-value pairs with no extraneous whitespace around the `=` sign. Formatting errors here do not always produce obvious error messages and can result in variables being silently ignored.

* **`changed` versus `ok` in PLAY RECAP reflects idempotency state.** A `changed` result on a package install or service start task indicates the resource was not in the desired state before the run. Re-running the playbook against the same host after a successful first run would produce `ok` for both tasks, demonstrating Ansible's idempotency. This distinction is important for auditing playbook impact in production environments.








<img width="1110" height="766" alt="image" src="https://github.com/user-attachments/assets/82df68d0-b37b-4372-942d-708595978b22" />
<img width="1098" height="780" alt="image" src="https://github.com/user-attachments/assets/b6c038e8-4d79-4256-b06f-d7d3c4fb9772" />
<img width="1110" height="866" alt="image" src="https://github.com/user-attachments/assets/685c1252-e9a6-4788-a9b3-965effb191f0" />






