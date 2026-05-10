# Ansible File Distribution to Application Servers | Stratos DC

## Table of Contents

* [Overview](#overview)
* [Architecture and Context](#architecture-and-context)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
  * [Step 1 - Verify Environment and Toolchain](#step-1---verify-environment-and-toolchain)
  * [Step 2 - Inspect the Source File](#step-2---inspect-the-source-file)
  * [Step 3 - Create the Ansible Inventory](#step-3---create-the-ansible-inventory)
  * [Step 4 - Author the Ansible Playbook](#step-4---author-the-ansible-playbook)
  * [Step 5 - Validate Inventory Connectivity](#step-5---validate-inventory-connectivity)
  * [Step 6 - Execute the Playbook](#step-6---execute-the-playbook)
  * [Step 7 - Verify Remote File Placement](#step-7---verify-remote-file-placement)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [Errors and Resolutions](#errors-and-resolutions)

---

## Overview

This implementation documents the automated distribution of a static HTML file from a centralized jump host to all application servers in the Stratos Data Center using Ansible. The solution demonstrates a repeatable, agent-less file delivery pattern using SSH-based automation, an inventory-driven host model, and idempotent playbook design.

**Objective:** Copy `/usr/src/itadmin/index.html` from the jump host to `/opt/itadmin/index.html` on all three application servers (`stapp01`, `stapp02`, `stapp03`).

**Toolchain:** Ansible Core 2.14.18 | Python 3.9 | Jump Host (jump-host) | RHEL-based application servers

---

## Architecture and Context

```
jump-host (Control Node)
    |
    |-- /usr/src/itadmin/index.html  (source file)
    |-- /home/thor/ansible/inventory  (host definitions)
    |-- /home/thor/ansible/playbook.yml  (automation logic)
    |
    |-- SSH --> stapp01 (ansible_user: tony)
    |-- SSH --> stapp02 (ansible_user: steve)
    |-- SSH --> stapp03 (ansible_user: banner)
                |
                +-- /opt/itadmin/index.html  (destination)
```

All playbook execution originates from the jump host. The application servers are managed nodes; no Ansible is installed or required on them. Privilege escalation (`become: true`) is used to write to `/opt/itadmin`, which requires root ownership.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Ansible Core | 2.14.x or later installed on the jump host |
| SSH Access | Jump host has SSH reachability to all application servers |
| Source File | `/usr/src/itadmin/index.html` exists on the jump host |
| Ansible Config | `host_key_checking = False` set in `/etc/ansible/ansible.cfg` |
| Privilege Escalation | Application server accounts have `sudo` capability |

---

## Implementation

### Step 1 - Verify Environment and Toolchain

Confirm the operating user, hostname, and Ansible version to establish a known baseline before any configuration is authored.

```bash
whoami
hostname
ansible --version
```

**Output:**

```
thor
jump-host

ansible [core 2.14.18]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/thor/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.9/site-packages/ansible
  ansible collection location = /home/thor/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.9.19 (main, Jun 11 2024, 00:00:00) [GCC 11.4.1 20231218 (Red Hat 11.4.1-3)] (/usr/bin/python3)
  jinja version = 3.1.2
  libyaml = True
```

Inspect the home directory and the pre-existing ansible working directory to confirm the starting state:

```bash
ls -la
ls -la /home/thor/ansible/
```

**Output:**

```
total 44
drwx------ 1 thor thor 4096 May 10 18:57 .
drwxr-xr-x 1 root root 4096 Mar  5 10:51 ..
drwxr-xr-x 3 thor thor 4096 May 10 18:57 .ansible
-rw-r--r-- 1 thor thor   18 Feb 15  2024 .bash_logout
-rw-r--r-- 1 thor thor  141 Feb 15  2024 .bash_profile
-rw-r--r-- 1 thor thor  581 Mar  5 10:52 .bashrc
drwxr-xr-x 1 thor thor 4096 Mar  5 10:52 .config
drwxr-xr-x 2 thor thor 4096 May 10 18:53 ansible

total 12
drwxr-xr-x 2 thor thor 4096 May 10 18:53 .
drwx------ 1 thor thor 4096 May 10 18:57 ..
```

The `ansible/` directory exists but contains no files yet. All artifacts will be created within this directory.

Review the global Ansible configuration to confirm host key checking behavior:

```bash
cat /etc/ansible/ansible.cfg
```

**Output:**

```ini
[defaults]
host_key_checking = False
```

> `host_key_checking = False` is pre-configured, removing the need for manual SSH key acceptance on first connection to each managed node. This is appropriate for a controlled internal network environment.

*Screenshots: Terminal output confirming ansible version, home directory listing, and ansible.cfg contents*

<img width="508" height="383" alt="image" src="https://github.com/user-attachments/assets/3746c70f-3d6c-4bd6-9736-1480ddd6590c" />
<img width="512" height="376" alt="image" src="https://github.com/user-attachments/assets/d07fa687-d2c2-4a74-a791-55c9cbefeb31" />

---

### Step 2 - Inspect the Source File

Confirm the source file exists on the jump host at the path expected by the playbook.

```bash
ls -la /usr/src/itadmin/index.html
```

**Output:**

```
-rw-r--r-- 1 root root 35 May 10 18:53 /usr/src/itadmin/index.html
```

The file is owned by root and readable by all. The `copy` module in the Ansible playbook will read this file as the control node's local file and push it to each managed node.

*Screenshot: ls -la output confirming source file existence and permissions*

<img width="509" height="318" alt="image" src="https://github.com/user-attachments/assets/bcfaf4e8-f1c9-42ad-bafe-014d750176e2" />

---

### Step 3 - Create the Ansible Inventory

Author the inventory file at `/home/thor/ansible/inventory`. This file defines all three application servers as managed nodes with their per-host connection variables.

```bash
cat > /home/thor/ansible/inventory << 'EOF'
stapp01 ansible_user=tony ansible_ssh_pass=Ir0nM@n ansible_become=true ansible_become_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_ssh_pass=Am3ric@ ansible_become=true ansible_become_pass=Am3ric@
stapp03 ansible_user=banner ansible_ssh_pass=BigGr33n ansible_become=true ansible_become_pass=BigGr33n
EOF
```

Verify the file was written correctly:

```bash
cat /home/thor/ansible/inventory
```

**Output:**

```
stapp01 ansible_user=tony ansible_ssh_pass=Ir0nM@n ansible_become=true ansible_become_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_ssh_pass=Am3ric@ ansible_become=true ansible_become_pass=Am3ric@
stapp03 ansible_user=banner ansible_ssh_pass=BigGr33n ansible_become=true ansible_become_pass=BigGr33n
```

**Inventory Variable Reference:**

| Variable | Purpose |
|---|---|
| `ansible_user` | SSH login user on the managed node |
| `ansible_ssh_pass` | SSH password for authentication |
| `ansible_become` | Enables privilege escalation (sudo) |
| `ansible_become_pass` | Password used for sudo escalation |

> Each host entry is self-contained. This flat inventory format is well-suited for small, static environments. For larger environments, group variables (`group_vars/`) and Ansible Vault encryption of sensitive credentials are strongly preferred.

*Screenshot: cat output of the completed inventory file*

---

### Step 4 - Author the Ansible Playbook

Create the playbook at `/home/thor/ansible/playbook.yml`. The playbook targets all hosts defined in the inventory, ensures the destination directory exists with appropriate permissions, then copies the source file.

```bash
cat > /home/thor/ansible/playbook.yml << 'EOF'
---
- name: Copy index.html to all application servers
  hosts: all
  become: true
  tasks:
    - name: Ensure /opt/itadmin directory exists
      file:
        path: /opt/itadmin
        state: directory
        mode: '0755'

    - name: Copy index.html to /opt/itadmin
      copy:
        src: /usr/src/itadmin/index.html
        dest: /opt/itadmin/index.html
EOF
```

Verify the playbook was written correctly:

```bash
cat /home/thor/ansible/playbook.yml
```

**Output:**

```yaml
---
- name: Copy index.html to all application servers
  hosts: all
  become: true
  tasks:
    - name: Ensure /opt/itadmin directory exists
      file:
        path: /opt/itadmin
        state: directory
        mode: '0755'

    - name: Copy index.html to /opt/itadmin
      copy:
        src: /usr/src/itadmin/index.html
        dest: /opt/itadmin/index.html
```

**Playbook Design Notes:**

* `hosts: all` targets every host in the inventory without requiring a named group
* `become: true` at the play level ensures all tasks run with elevated privileges
* The `file` task with `state: directory` is idempotent - it creates the directory only if absent, and makes no change if it already exists
* `mode: '0755'` sets standard directory permissions (owner read-write-execute, group and other read-execute)
* The `copy` module reads the source file from the Ansible control node (jump host) and pushes it to the destination path on each managed node

*Screenshot: cat output of the completed playbook.yml*

---

### Step 5 - Validate Inventory Connectivity

Before executing the playbook, use the `ping` module to confirm SSH connectivity and Python interpreter availability on all three managed nodes.

```bash
cd /home/thor/ansible && ansible -i inventory all -m ping
```

**Output:**

```
stapp02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
stapp03 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
stapp01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

All three nodes returned `SUCCESS` with `pong`. The Python interpreter was auto-discovered at `/usr/bin/python3` on each server, confirming full Ansible readiness.

*Screenshot: ansible ping output showing SUCCESS for stapp01, stapp02, and stapp03*

---

### Step 6 - Execute the Playbook

Run the playbook using the inventory file. The command is issued from within the `/home/thor/ansible/` directory.

```bash
ansible-playbook -i inventory playbook.yml
```

**Output:**

```
PLAY [Copy index.html to all application servers] **************************************************************

TASK [Gathering Facts] *****************************************************************************************
ok: [stapp03]
ok: [stapp01]
ok: [stapp02]

TASK [Ensure /opt/itadmin directory exists] ********************************************************************
ok: [stapp03]
ok: [stapp02]
ok: [stapp01]

TASK [Copy index.html to /opt/itadmin] *************************************************************************
changed: [stapp03]
changed: [stapp02]
changed: [stapp01]

PLAY RECAP *****************************************************************************************************
stapp01                    : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

**Recap Interpretation:**

| Column | Value | Meaning |
|---|---|---|
| `ok` | 3 | Three tasks ran successfully (facts gather, directory check, file copy) |
| `changed` | 1 | The file copy task made a real change on each host |
| `unreachable` | 0 | No connectivity failures |
| `failed` | 0 | No task failures |

The `file` task returned `ok` (not `changed`) because the `/opt/itadmin` directory already existed on all three servers. The `copy` task returned `changed` because the file had not previously been placed there, confirming a first-time delivery.

*Screenshot: Full ansible-playbook execution output showing PLAY RECAP with 0 failures across all three hosts*

---

### Step 7 - Verify Remote File Placement

Run an ad-hoc shell command against all managed nodes to confirm the file was placed at the correct path with expected ownership and timestamps.

```bash
ansible -i inventory all -m shell -a "ls -la /opt/itadmin/index.html"
```

**Output:**

```
stapp02 | CHANGED | rc=0 >>
-rw-r--r-- 1 root root 35 May 10 19:08 /opt/itadmin/index.html
stapp03 | CHANGED | rc=0 >>
-rw-r--r-- 1 root root 35 May 10 19:08 /opt/itadmin/index.html
stapp01 | CHANGED | rc=0 >>
-rw-r--r-- 1 root root 35 May 10 19:08 /opt/itadmin/index.html
```

The file is confirmed present on all three application servers. Key observations:

* **Owner:** root (consistent with `become: true` execution)
* **Size:** 35 bytes (matches the source file at `/usr/src/itadmin/index.html`)
* **Timestamp:** `May 10 19:08` (consistent across all nodes, confirming simultaneous delivery)
* **Permissions:** `-rw-r--r--` (world-readable, no execute bit)

*Screenshot: ansible shell ad-hoc output confirming index.html on stapp01, stapp02, and stapp03*

---

## Best Practices

* **Idempotent task ordering:** The `file` module task precedes the `copy` task, ensuring the destination directory always exists before a file is written to it. This prevents race conditions on first runs.

* **Connectivity pre-validation:** Executing `ansible -m ping` before `ansible-playbook` catches SSH, credential, or interpreter issues before committing to a full playbook run. This is a non-negotiable pre-flight step in production workflows.

* **Heredoc-based file creation:** Using `cat > file << 'EOF'` with a quoted delimiter ensures no shell variable interpolation occurs inside the content block. This is safer than unquoted heredocs when writing YAML or INI files containing special characters.

* **Immediate content verification:** Each file creation step was followed by a `cat` of the written file to confirm the contents on disk match the intended input. This eliminates silent write errors or encoding issues.

* **`become` at play level:** Setting `become: true` at the play level rather than the task level applies consistent privilege escalation to all tasks in the play, reducing the risk of partial failures due to missing permissions on individual tasks.

* **Ad-hoc post-run verification:** Using an ad-hoc `shell` command to inspect the remote files after playbook execution provides independent, out-of-band confirmation separate from Ansible's own change tracking.

---

## Lessons Learned

* **`ok` vs `changed` in the recap reveals directory pre-existence:** The `file` task returning `ok` instead of `changed` indicated `/opt/itadmin` was already present on all three servers. This is valuable signal in production - it means the managed nodes had been partially pre-configured, which is worth tracking.

* **`shell` module marks `CHANGED` even for read-only commands:** The ad-hoc verification step using `-m shell -a "ls -la ..."` returned `CHANGED | rc=0` for all hosts. This is expected behavior: the `shell` module always marks tasks as changed because Ansible cannot determine whether a shell command modified state. For idempotent checks, prefer `-m command` or `-m stat`. In ad-hoc verification contexts, the `CHANGED` status is cosmetic and does not indicate any file was modified.

* **Flat inventory with inline credentials is appropriate only for controlled environments:** Storing `ansible_ssh_pass` and `ansible_become_pass` in plaintext inventory is acceptable in isolated training and internal lab environments. In production, credentials must be stored in Ansible Vault or retrieved from a secrets manager. The inventory structure demonstrated here would be unchanged; only the credential storage mechanism would differ.

* **Source path in `copy` module resolves on the control node, not the managed node:** The `src: /usr/src/itadmin/index.html` path is resolved on the jump host (control node). This is a common point of confusion for engineers new to Ansible. If the source path does not exist on the control node, the task fails during the push phase, not during connectivity. Using `fetch` and `synchronize` modules follows different resolution logic.

---

## Errors and Resolutions

No errors were encountered during this implementation. All tasks completed with expected outcomes across all execution phases.

The following are proactive risk items relevant to this pattern:

| Scenario | Root Cause | Resolution |
|---|---|---|
| `UNREACHABLE` during ping | Incorrect `ansible_ssh_pass` or `ansible_user` in inventory, or firewall blocking SSH port 22 | Verify credentials match the target system accounts; confirm SSH connectivity with `ssh user@host` manually |
| `Permission denied` on file copy | `ansible_become_pass` is incorrect or the account lacks sudo rights | Verify the become password and confirm the user has passwordless or password-based sudo for the target path |
| `[Errno 2] No such file or directory` on source | Source file `/usr/src/itadmin/index.html` does not exist on the control node | Confirm the file path on the jump host with `ls -la /usr/src/itadmin/index.html` before running the playbook |
| `copy` task returns `ok` instead of `changed` | File already exists on managed node with identical content (checksum match) | This is correct idempotent behavior. No action needed. The file is already in the desired state. |
| `fatal: [hostX]: FAILED! => {"msg": "Missing sudo password"}` | `ansible_become_pass` not set in inventory for that host | Add `ansible_become_pass` to the affected host entry in the inventory file |







<img width="512" height="314" alt="image" src="https://github.com/user-attachments/assets/b69ee445-113f-4cba-96ba-d210d6fedad9" />
<img width="510" height="326" alt="image" src="https://github.com/user-attachments/assets/b72e87ce-0b55-4159-892f-b33a0f52567f" />
<img width="514" height="366" alt="image" src="https://github.com/user-attachments/assets/a112802e-2925-4736-860f-1778aa588d97" />
<img width="511" height="388" alt="image" src="https://github.com/user-attachments/assets/8e55df4c-2987-4eaa-b61f-d433f23dad39" />

<img width="509" height="365" alt="image" src="https://github.com/user-attachments/assets/022107ad-1691-4ab6-bd59-f158c4c54637" />
<img width="512" height="320" alt="image" src="https://github.com/user-attachments/assets/8dfec186-6291-48c5-88d8-ca21ff1d5005" />
<img width="508" height="399" alt="image" src="https://github.com/user-attachments/assets/c616fad9-c51a-4153-ba93-09ac81bf8c41" />
<img width="511" height="425" alt="image" src="https://github.com/user-attachments/assets/b5597036-f293-439c-8f0c-f8d2e44f4f71" />
<img width="514" height="425" alt="image" src="https://github.com/user-attachments/assets/5b8426ee-a6f2-42ee-99bd-aa46701c78f4" />
<img width="512" height="410" alt="image" src="https://github.com/user-attachments/assets/561cd050-626a-4388-9efb-a5aa2cfa5cd4" />
