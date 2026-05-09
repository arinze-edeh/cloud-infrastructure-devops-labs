# Ansible Inventory Configuration and Automated File Provisioning on App Server 3

---

## Table of Contents

* [Overview](#overview)
* [Architecture and Environment Context](#architecture-and-environment-context)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Inspect the Existing Inventory File](#step-1-inspect-the-existing-inventory-file)
  * [Step 2: Update the Inventory File with Group and Host Definition](#step-2-update-the-inventory-file-with-group-and-host-definition)
  * [Step 3: Validate Connectivity with an Ansible Ad-Hoc Ping](#step-3-validate-connectivity-with-an-ansible-ad-hoc-ping)
  * [Step 4: Author the Ansible Playbook](#step-4-author-the-ansible-playbook)
  * [Step 5: Syntax Check the Playbook](#step-5-syntax-check-the-playbook)
  * [Step 6: Execute the Playbook](#step-6-execute-the-playbook)
* [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Repository Structure](#repository-structure)

---

## Overview

This implementation demonstrates how to configure an Ansible inventory file and author a task-specific playbook from a shared jump host targeting a remote application server in the **Stratos DC** environment. The objective is to extend an incomplete team handoff by updating the static inventory to include the correct host group definition, verifying connectivity, and deploying a playbook that creates an empty file at `/tmp/file.txt` on **App Server 3** (`stapp03`).

Validation is performed by running the playbook using the exact command:

```bash
ansible-playbook -i inventory playbook.yml
```

No additional arguments are passed during validation, making accurate inventory configuration the critical dependency for successful execution.

---

## Architecture and Environment Context

| Component | Detail |
|---|---|
| Control Node | `jump-host` (accessed as user `thor`) |
| Target Host | `stapp03` (App Server 3, Stratos DC) |
| Remote User | `banner` |
| Inventory Path | `/home/thor/ansible/inventory` |
| Playbook Path | `/home/thor/ansible/playbook.yml` |
| Authentication | SSH password via `ansible_ssh_pass` |
| SSH Host Key Checking | Disabled via `ansible_ssh_common_args` |
| Working Directory | `/home/thor/ansible/` |

---

## Prerequisites

* Ansible is installed and available on the jump host
* SSH access to `stapp03` is reachable from the jump host
* Credentials for the `banner` user on `stapp03` are known
* The `/home/thor/ansible/` directory exists and is writable

---

## Implementation Guide

### Step 1: Inspect the Existing Inventory File

Before making any changes, inspect the current state of the inventory file to understand what the previous team member left in place.

```bash
cat /home/thor/ansible/inventory
```

**Output observed:**

```
stapp03 ansible_user=banner ansible_ssh_pass=$pwd ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

**Key findings from inspection:**

* `stapp03` is defined as a bare host entry with no group assignment
* The SSH password is set to the literal string `$pwd`, indicating a placeholder that was never replaced with the actual credential
* No Ansible host group is defined, meaning the inventory cannot be reliably targeted by group name in a playbook

Screenshot: `01-initial-inventory-cat.png`

---

### Step 2: Update the Inventory File with Group and Host Definition

Open the inventory file for editing and apply two changes: add the `[stapp03]` group header above the host entry, and replace the placeholder `$pwd` with the correct SSH password (`BigGr33n`).

```bash
vi /home/thor/ansible/inventory
```

**Final inventory file content after editing:**

```ini
[stapp03]
stapp03 ansible_user=banner ansible_ssh_pass=BigGr33n ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Verify the changes were saved correctly:

```bash
cat /home/thor/ansible/inventory
```

Screenshot: `02-updated-inventory-cat.png`

---

### Step 3: Validate Connectivity with an Ansible Ad-Hoc Ping

Before authoring the playbook, confirm that Ansible can successfully reach `stapp03` using the updated inventory. Change to the working directory first.

```bash
cd /home/thor/ansible
ansible stapp03 -i inventory -m ping
```

**Output observed:**

```
[WARNING]: Found both group and host with same name: stapp03
stapp03 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

The `SUCCESS` status with a `pong` response confirms that:

* Authentication is working correctly with the `banner` user and the `BigGr33n` credential
* The Python interpreter was successfully discovered on the target host
* The control node can reach `stapp03` over SSH without host key verification errors

The warning `Found both group and host with same name: stapp03` is expected behavior when both an inventory group and a host share the identical name. It does not affect functionality and is documented further in the [Errors Encountered and Resolutions](#errors-encountered-and-resolutions) section.

Screenshot: `03-ad-hoc-ping-success.png`

---

### Step 4: Author the Ansible Playbook

Create the playbook file at `/home/thor/ansible/playbook.yml`. The playbook targets the `stapp03` host group, escalates privileges with `become: yes`, and uses the `file` module to create an empty file at `/tmp/file.txt` using the `touch` state.

```bash
vi playbook.yml
```

**Playbook content:**

```yaml
---
- name: Create empty file on App Server 3
  hosts: stapp03
  become: yes
  tasks:
    - name: Create /tmp/file.txt
      file:
        path: /tmp/file.txt
        state: touch
```

Verify the playbook was written correctly:

```bash
cat playbook.yml
```

Screenshot: `04-playbook-cat.png`

**Design decisions in this playbook:**

* `become: yes` ensures the task runs with elevated privileges on the target host, which is required for operations in directories that may have restricted write permissions
* `state: touch` is the idiomatic Ansible approach to creating an empty file; it also updates the file's modification timestamp if the file already exists, making the task fully idempotent
* `hosts: stapp03` matches the group name defined in the inventory, ensuring the playbook targets the correct server without ambiguity

---

### Step 5: Syntax Check the Playbook

Before running the playbook, perform a syntax check to catch any YAML or Ansible structural errors.

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
```

**Output observed:**

```
[WARNING]: Found both group and host with same name: stapp03
playbook: playbook.yml
```

A clean syntax check result confirms the playbook YAML structure is valid and Ansible can parse it without errors. The warning is expected and non-blocking.

Screenshot: `05-syntax-check.png`

---

### Step 6: Execute the Playbook

Run the playbook using the exact validation command. No additional flags or arguments are used beyond what is specified.

```bash
ansible-playbook -i inventory playbook.yml
```

**Output observed:**

```
[WARNING]: Found both group and host with same name: stapp03

PLAY [Create empty file on App Server 3] ****************************************************

TASK [Gathering Facts] **********************************************************************
ok: [stapp03]

TASK [Create /tmp/file.txt] *****************************************************************
changed: [stapp03]

PLAY RECAP **********************************************************************************
stapp03                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

**Play recap interpretation:**

| Field | Value | Meaning |
|---|---|---|
| `ok` | 2 | Facts gathering and the file task both completed without error |
| `changed` | 1 | `/tmp/file.txt` was created on `stapp03` (state changed from absent to present) |
| `unreachable` | 0 | No connectivity failures |
| `failed` | 0 | No task failures |
| `skipped` | 0 | No tasks were skipped |

The playbook executed successfully end-to-end. The file `/tmp/file.txt` now exists on App Server 3.

Screenshot: `06-playbook-execution-success.png`

---

## Errors Encountered and Resolutions

### Warning: Found Both Group and Host with Same Name

**Warning message:**

```
[WARNING]: Found both group and host with same name: stapp03
```

**Root cause:**

In the inventory file, the host is named `stapp03` and it is placed under a group also named `[stapp03]`. Ansible detects this name collision and raises a warning to inform the operator that both a group and a standalone host share the same identifier.

**Impact:**

This is a non-fatal warning. Ansible resolves the ambiguity internally and the playbook executes correctly against the intended target.

**Resolution:**

No code change was required for the scope of this task. The warning can be eliminated in production environments by renaming the group to a more descriptive identifier such as `[app_servers]` or `[stratos_app]` and updating the `hosts:` field in the playbook accordingly. This naming separation is the recommended practice for maintainable inventories.

---

## Best Practices Applied

* **Connectivity verification before playbook execution:** An ad-hoc `ping` module test was run against the target host before authoring and executing the playbook. This validates authentication, reachability, and Python interpreter availability independently of the playbook itself, reducing ambiguity when diagnosing failures.

* **Syntax check before execution:** `--syntax-check` was used to validate the playbook structure before running it against live infrastructure. This is a low-cost safeguard that prevents avoidable execution failures caused by YAML formatting errors.

* **Idempotent file creation with `state: touch`:** Using the `file` module with `state: touch` ensures the task can be run multiple times without producing unintended side effects. If the file already exists, the task updates its timestamp and reports `ok` rather than failing.

* **Privilege escalation with `become: yes`:** Applying `become: yes` at the play level rather than the task level ensures consistent privilege handling across all tasks in the play, reducing the risk of partial privilege application in multi-task playbooks.

* **Disabling SSH host key checking for lab and ephemeral environments:** Using `ansible_ssh_common_args='-o StrictHostKeyChecking=no'` in the inventory prevents connection failures in environments where SSH host keys have not been pre-registered. This is appropriate for controlled lab and ephemeral infrastructure; in production, host keys should be pre-populated in `known_hosts` and this argument should be removed.

* **Static inventory with explicit credential parameters:** Defining `ansible_user` and `ansible_ssh_pass` directly in the inventory ensures portability and reproducibility for team handoff scenarios, making the configuration self-contained without relying on external variable files or Ansible Vault for this scope.

---

## Lessons Learned

* **Team handoffs require complete credential and structural validation.** The original inventory contained a literal `$pwd` placeholder instead of the actual SSH password. Treating every inherited configuration file as potentially incomplete and inspecting it before use is a critical discipline in shared infrastructure environments.

* **Matching group names to playbook `hosts:` directives is essential.** Without the `[stapp03]` group header, the playbook would have failed to match the target host through group-based targeting. Understanding the distinction between individual host targeting and group-based targeting in Ansible inventories prevents subtle misconfigurations that only surface at execution time.

* **Warnings require investigation, not dismissal.** The `Found both group and host with same name` warning could indicate a structural issue in a larger, more complex inventory. Treating all warnings as informational rather than ignorable builds operational discipline that scales to production environments where inventory files manage hundreds of hosts.

* **Ad-hoc module testing is an underutilized debugging primitive.** Isolating connectivity and authentication validation from playbook execution reduces the number of variables in flight when diagnosing a failure. The `ansible ... -m ping` pattern should be a reflex before every first playbook run against a new or unfamiliar host.

---

## Repository Structure

```
ansible/
├── inventory          # Static inventory file defining stapp03 host group and connection parameters
└── playbook.yml       # Playbook to create /tmp/file.txt on App Server 3
```

---

*Documented by Arinze Edeh | Cloud and DevOps Engineer*







<img width="516" height="307" alt="image" src="https://github.com/user-attachments/assets/45139d17-7560-4152-b2d6-8b68b7114e0a" />
<img width="512" height="403" alt="image" src="https://github.com/user-attachments/assets/e5d53d9d-4d7b-4bd9-9bc9-81e6c65a1b13" />
<img width="518" height="269" alt="image" src="https://github.com/user-attachments/assets/a649279c-bbe4-4b1e-ba1c-764cb7454d5b" />
<img width="517" height="350" alt="image" src="https://github.com/user-attachments/assets/7a4ec7d6-0360-42b1-bbfa-0a326006668a" />
<img width="516" height="329" alt="image" src="https://github.com/user-attachments/assets/24f422be-c0a5-41b2-a9c5-8efb0738e3e6" />
<img width="517" height="411" alt="image" src="https://github.com/user-attachments/assets/99a966e0-4157-4cac-a04a-110fe8f9f40b" />
<img width="518" height="380" alt="image" src="https://github.com/user-attachments/assets/001a3a0c-a95b-4a4f-bb57-15a29390d8a9" />
<img width="519" height="323" alt="image" src="https://github.com/user-attachments/assets/f0339c18-583a-4d8f-b5c5-ec10626dd3fe" />
<img width="514" height="369" alt="image" src="https://github.com/user-attachments/assets/2c2d1797-d647-446c-b3a4-7a458ee30a05" />
<img width="517" height="433" alt="image" src="https://github.com/user-attachments/assets/3f23e268-5028-4551-8715-ed1b0ed747d3" />
