# Ansible File Distribution to Application Servers | Stratos DC

## Table of Contents

* [Overview](#overview)
* [Architecture and Context](#architecture-and-context)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
  * [Step 1 - Confirm Operating User and Hostname](#step-1---confirm-operating-user-and-hostname)
  * [Step 2 - Verify Ansible Toolchain](#step-2---verify-ansible-toolchain)
  * [Step 3 - Inspect Home Directory State](#step-3---inspect-home-directory-state)
  * [Step 4 - Confirm Source File Exists](#step-4---confirm-source-file-exists)
  * [Step 5 - Inspect the Ansible Working Directory](#step-5---inspect-the-ansible-working-directory)
  * [Step 6 - Review Global Ansible Configuration](#step-6---review-global-ansible-configuration)
  * [Step 7 - Create the Ansible Inventory](#step-7---create-the-ansible-inventory)
  * [Step 8 - Verify the Inventory File](#step-8---verify-the-inventory-file)
  * [Step 9 - Author the Ansible Playbook](#step-9---author-the-ansible-playbook)
  * [Step 10 - Verify the Playbook File](#step-10---verify-the-playbook-file)
  * [Step 11 - Validate Inventory Connectivity](#step-11---validate-inventory-connectivity)
  * [Step 12 - Execute the Playbook](#step-12---execute-the-playbook)
  * [Step 13 - Verify Remote File Placement](#step-13---verify-remote-file-placement)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [Errors and Resolutions](#errors-and-resolutions)

---

## Overview

This implementation documents the automated distribution of a static HTML file from a centralized jump host to all application servers in the Stratos Data Center using Ansible. The solution demonstrates a repeatable, agent-less file delivery pattern using SSH-based automation, an inventory-driven host model, and idempotent playbook design.

**Objective:** Copy `/usr/src/itadmin/index.html` from the jump host to `/opt/itadmin/index.html` on all three application servers (`stapp01`, `stapp02`, `stapp03`).

**Toolchain:** Ansible Core 2.14.18 | Python 3.9 | Jump Host (`jump-host`) | RHEL-based application servers

---

## Architecture and Context

```
jump-host (Control Node)
    |
    |-- /usr/src/itadmin/index.html      (source file)
    |-- /home/thor/ansible/inventory     (host definitions)
    |-- /home/thor/ansible/playbook.yml  (automation logic)
    |
    |-- SSH --> stapp01 (ansible_user: tony)
    |-- SSH --> stapp02 (ansible_user: steve)
    |-- SSH --> stapp03 (ansible_user: banner)
                |
                +-- /opt/itadmin/index.html  (destination)
```

All playbook execution originates from the jump host. The application servers are managed nodes; no Ansible installation is required on them. Privilege escalation (`become: true`) is used to write to `/opt/itadmin`, which requires root ownership.

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

### Step 1 - Confirm Operating User and Hostname

Establish the identity of the control node user and the hostname to confirm the correct execution context before any configuration work begins.

```bash
whoami
hostname
```

**Output:**

```
thor
jump-host
```

Execution is confirmed on `jump-host` as user `thor`, which is the designated control node user for this automation.

*Screenshot: Terminal output confirming user `thor` and hostname `jump-host`*

<img width="512" height="314" alt="image" src="https://github.com/user-attachments/assets/b69ee445-113f-4cba-96ba-d210d6fedad9" />

---

### Step 2 - Verify Ansible Toolchain

Confirm Ansible is installed, identify the version, and validate the Python interpreter and configuration file in use.

```bash
ansible --version
```

**Output:**

```
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

Ansible Core 2.14.18 is confirmed with Python 3.9.19 and the global config file at `/etc/ansible/ansible.cfg`. The `libyaml = True` flag confirms native YAML parsing is active, which improves playbook parse performance.

*Screenshot: ansible --version output on jump-host*

<img width="510" height="326" alt="image" src="https://github.com/user-attachments/assets/b72e87ce-0b55-4159-892f-b33a0f52567f" />

---

### Step 3 - Inspect Home Directory State

List the home directory contents to establish the current filesystem state and confirm the `ansible/` working directory is present.

```bash
ls -la
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
```

The `ansible/` directory exists under `/home/thor/`. All inventory and playbook artifacts will be created inside this directory.

*Screenshot: ls -la output of /home/thor showing the ansible working directory*

---

### Step 4 - Confirm Source File Exists

Verify the source HTML file is present on the jump host at the path the playbook will reference as `src`.

```bash
ls -la /usr/src/itadmin/index.html
```

**Output:**

```
-rw-r--r-- 1 root root 35 May 10 18:53 /usr/src/itadmin/index.html
```

The file is confirmed at 35 bytes, owned by root, and world-readable. The `copy` module will read this file from the control node filesystem and push it to each managed node.

*Screenshot: ls -la output confirming the source file exists at /usr/src/itadmin/index.html*

---

### Step 5 - Inspect the Ansible Working Directory

Confirm the current state of the `ansible/` working directory before creating any files inside it.

```bash
ls -la /home/thor/ansible/
```

**Output:**

```
total 12
drwxr-xr-x 2 thor thor 4096 May 10 18:53 .
drwx------ 1 thor thor 4096 May 10 18:57 ..
```

The directory is empty. No inventory or playbook files exist yet. Both artifacts will be created from scratch in the steps that follow.

*Screenshot: ls -la of /home/thor/ansible showing an empty directory*

---

### Step 6 - Review Global Ansible Configuration

Read the global Ansible configuration file to confirm host key checking behavior and understand what defaults are already applied to the control node environment.

```bash
cat /etc/ansible/ansible.cfg
```

**Output:**

```ini
[defaults]
host_key_checking = False
```

`host_key_checking = False` is pre-configured globally, removing the need for manual SSH fingerprint acceptance on first connection to each managed node. This is appropriate for a controlled internal network environment where all hosts are known and trusted.

*Screenshot: cat output of /etc/ansible/ansible.cfg showing host_key_checking = False*

---

### Step 7 - Create the Ansible Inventory

Author the inventory file at `/home/thor/ansible/inventory` using a heredoc. This file defines all three application servers as managed nodes with their per-host SSH and privilege escalation credentials.

```bash
cat > /home/thor/ansible/inventory << 'EOF'
stapp01 ansible_user=tony ansible_ssh_pass=Ir0nM@n ansible_become=true ansible_become_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_ssh_pass=Am3ric@ ansible_become=true ansible_become_pass=Am3ric@
stapp03 ansible_user=banner ansible_ssh_pass=BigGr33n ansible_become=true ansible_become_pass=BigGr33n
EOF
```

**Inventory Variable Reference:**

| Variable | Purpose |
|---|---|
| `ansible_user` | SSH login user on the managed node |
| `ansible_ssh_pass` | SSH password for authentication |
| `ansible_become` | Enables privilege escalation (sudo) |
| `ansible_become_pass` | Password supplied to the sudo prompt |

> The heredoc delimiter is quoted (`'EOF'`) to prevent shell variable interpolation inside the inventory content block. This is critical when credentials contain special characters such as `@`, `!`, or `$`.

*Screenshot: heredoc command written to terminal creating the inventory file*

---

### Step 8 - Verify the Inventory File

Read the written inventory file back to confirm its contents match the intended input exactly before proceeding to playbook creation.

```bash
cat /home/thor/ansible/inventory
```

**Output:**

```
stapp01 ansible_user=tony ansible_ssh_pass=Ir0nM@n ansible_become=true ansible_become_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_ssh_pass=Am3ric@ ansible_become=true ansible_become_pass=Am3ric@
stapp03 ansible_user=banner ansible_ssh_pass=BigGr33n ansible_become=true ansible_become_pass=BigGr33n
```

All three host entries are confirmed correct. Each line contains the full set of connection variables required for SSH authentication and privilege escalation on that node.

*Screenshot: cat output of /home/thor/ansible/inventory showing all three host entries*

---

### Step 9 - Author the Ansible Playbook

Create the playbook at `/home/thor/ansible/playbook.yml` using a heredoc. The playbook targets all inventory hosts, ensures the destination directory exists with correct permissions, then copies the source file.

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

**Playbook Design Notes:**

* `hosts: all` targets every host defined in the inventory without requiring a named group
* `become: true` at the play level applies privilege escalation uniformly to all tasks
* The `file` task with `state: directory` is idempotent - it creates the directory only if absent and makes no change if it already exists
* `mode: '0755'` sets standard directory permissions (owner read-write-execute, group and other read-execute)
* The `copy` module resolves `src` on the control node (jump host) and transfers the file to each managed node

*Screenshot: heredoc command written to terminal creating the playbook file*

---

### Step 10 - Verify the Playbook File

Read the written playbook back to confirm the YAML structure, task definitions, and indentation are correct before execution.

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

YAML structure is confirmed. Indentation, module names, and parameter values all match the intended design.

*Screenshot: cat output of /home/thor/ansible/playbook.yml showing full play and task definitions*

---

### Step 11 - Validate Inventory Connectivity

Change into the ansible working directory and run an ad-hoc ping against all managed nodes to confirm SSH connectivity and Python interpreter availability before committing to a full playbook run.

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

All three nodes returned `SUCCESS` with `pong`. Python 3 was auto-discovered at `/usr/bin/python3` on each server, confirming full Ansible readiness. The response order (stapp02, stapp03, stapp01) reflects parallel SSH execution, not the order hosts appear in the inventory.

*Screenshot: ansible ping ad-hoc output showing SUCCESS for all three application servers*

---

### Step 12 - Execute the Playbook

Run the playbook against all managed nodes using the inventory file. Execution is performed from within the `/home/thor/ansible/` directory established by the `cd` in the previous step.

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

**PLAY RECAP Interpretation:**

| Column | Value | Meaning |
|---|---|---|
| `ok` | 3 | Three tasks completed successfully per host (facts, directory check, file copy) |
| `changed` | 1 | The file copy task made a real change, confirming first-time delivery |
| `unreachable` | 0 | No connectivity failures across any host |
| `failed` | 0 | No task failures |

The `file` task returned `ok` (not `changed`) on all hosts, indicating `/opt/itadmin` was already present. The `copy` task returned `changed` on all hosts, confirming the file was newly placed during this execution.

*Screenshot: Full ansible-playbook output showing all tasks and PLAY RECAP with 0 failures across all three hosts*

---

### Step 13 - Verify Remote File Placement

Run an ad-hoc shell command against all managed nodes to independently confirm the file exists at the correct destination path with expected ownership, size, and permissions.

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

The file is confirmed present on all three application servers with consistent attributes:

| Attribute | Value | Significance |
|---|---|---|
| Owner | root | Consistent with `become: true` task execution |
| Size | 35 bytes | Matches the source file at `/usr/src/itadmin/index.html` |
| Timestamp | May 10 19:08 | Identical across all nodes, confirming concurrent delivery |
| Permissions | `-rw-r--r--` | World-readable, no execute bit, default `copy` module behavior |

*Screenshot: ansible shell ad-hoc output confirming index.html presence on stapp01, stapp02, and stapp03*

---

## Best Practices

* **Full baseline verification before any configuration work:** Confirming the operating user, hostname, Ansible version, home directory state, source file existence, working directory state, and global configuration before writing a single file ensures the environment is fully understood. This prevents silent assumption failures that surface only during execution.

* **Quoted heredoc delimiter for special character safety:** Using `<< 'EOF'` rather than `<< EOF` prevents the shell from interpreting variables or escape sequences inside the heredoc content block. This is critical when inventory values contain characters such as `@`, `!`, or `$`, which are common in password fields.

* **Immediate content verification after every file write:** Each file creation step was followed immediately by a `cat` of the written file to confirm on-disk contents match the intended input. This catches truncation, encoding issues, or unintended shell interpretation before they propagate to execution.

* **Connectivity pre-validation with ping before playbook execution:** Running `ansible -m ping` before `ansible-playbook` catches SSH, credential, or Python interpreter issues without committing to a full playbook run. This is a non-negotiable pre-flight step in production automation workflows.

* **Idempotent task ordering with directory creation preceding file copy:** The `file` module task runs before the `copy` task, ensuring the destination directory always exists before a file is written into it. This prevents task failure on any host where the directory has not been pre-created.

* **`become` scoped at the play level:** Applying `become: true` at the play level rather than the task level ensures consistent privilege escalation across all tasks, preventing partial failures caused by permission gaps on individual tasks.

* **Ad-hoc post-run verification as independent confirmation:** Using an out-of-band `shell` command to inspect remote files after playbook execution provides confirmation independent of Ansible's own change tracking, validating both file existence and file metadata simultaneously.

---

## Lessons Learned

* **`ok` on the directory task reveals pre-existing managed node state:** The `file` task returning `ok` instead of `changed` indicated `/opt/itadmin` was already present on all three servers before this run. In production this is meaningful signal. It indicates the managed nodes had been partially pre-configured, which is worth recording in change management history.

* **The `shell` module always reports `CHANGED` regardless of actual state modification:** The verification step using `-m shell -a "ls -la ..."` returned `CHANGED | rc=0` on all hosts. This is expected and documented behavior. The `shell` module cannot introspect whether its command modified system state, so it marks every execution as `CHANGED`. For idempotent remote inspection, the `stat` or `command` module is preferred. In a verification context, the `CHANGED` status is cosmetic only.

* **Source path in the `copy` module resolves on the control node, not the managed node:** The `src: /usr/src/itadmin/index.html` path is resolved on the jump host. This is a common point of confusion for engineers new to Ansible. If the source path does not exist on the control node the task fails during the push phase, not during SSH connectivity. Understanding this distinction is important when debugging copy failures in multi-tier architectures.

* **Parallel SSH execution produces non-sequential host output:** Both the ping and playbook outputs returned results in a varying host order (stapp02, stapp03, stapp01 rather than the inventory order stapp01, stapp02, stapp03). This is normal. Ansible connects to managed nodes in parallel by default (governed by the `forks` setting, defaulting to 5). Output order reflects which SSH connection resolves first, not inventory position.

---

## Errors and Resolutions

No errors were encountered during this implementation. All tasks completed with expected outcomes across every execution phase.

The following are proactive risk items relevant to this pattern:

| Scenario | Root Cause | Resolution |
|---|---|---|
| `UNREACHABLE` during ping | Incorrect `ansible_ssh_pass` or `ansible_user`, or firewall blocking port 22 | Verify credentials match the target accounts; confirm SSH reachability manually with `ssh user@host` |
| `Permission denied` on file copy | `ansible_become_pass` is incorrect or the account lacks sudo rights for the target path | Verify the become password and confirm the account has password-based sudo configured |
| `[Errno 2] No such file or directory` on source | `/usr/src/itadmin/index.html` does not exist on the control node at execution time | Confirm the source path on the jump host with `ls -la /usr/src/itadmin/index.html` before running the playbook |
| `copy` task returns `ok` instead of `changed` | File already exists on the managed node with an identical checksum | This is correct idempotent behavior. The file is already in the desired state. No action is needed. |
| `Missing sudo password` fatal error | `ansible_become_pass` not set for a specific host in the inventory | Add `ansible_become_pass` to the affected host entry in the inventory file |
























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







<img width="514" height="366" alt="image" src="https://github.com/user-attachments/assets/a112802e-2925-4736-860f-1778aa588d97" />
<img width="511" height="388" alt="image" src="https://github.com/user-attachments/assets/8e55df4c-2987-4eaa-b61f-d433f23dad39" />

<img width="509" height="365" alt="image" src="https://github.com/user-attachments/assets/022107ad-1691-4ab6-bd59-f158c4c54637" />
<img width="512" height="320" alt="image" src="https://github.com/user-attachments/assets/8dfec186-6291-48c5-88d8-ca21ff1d5005" />
<img width="508" height="399" alt="image" src="https://github.com/user-attachments/assets/c616fad9-c51a-4153-ba93-09ac81bf8c41" />
<img width="511" height="425" alt="image" src="https://github.com/user-attachments/assets/b5597036-f293-439c-8f0c-f8d2e44f4f71" />
<img width="514" height="425" alt="image" src="https://github.com/user-attachments/assets/5b8426ee-a6f2-42ee-99bd-aa46701c78f4" />
<img width="512" height="410" alt="image" src="https://github.com/user-attachments/assets/561cd050-626a-4388-9efb-a5aa2cfa5cd4" />
