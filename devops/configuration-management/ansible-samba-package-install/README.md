# Ansible Package Installation Across Multi-Node Infrastructure

Automated `logrotate` package deployment to all application servers in the Stratos Datacenter using Ansible ad-hoc connectivity validation and a structured YAML playbook executed from a centralized jump host.

---

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Implementation](#implementation)
  * [Step 1: Verify Ansible Installation](#step-1-verify-ansible-installation)
  * [Step 2: Validate SSH Connectivity to App Servers](#step-2-validate-ssh-connectivity-to-app-servers)
  * [Step 3: Create the Playbook Directory](#step-3-create-the-playbook-directory)
  * [Step 4: Create the Inventory File](#step-4-create-the-inventory-file)
  * [Step 5: Author the Ansible Playbook](#step-5-author-the-ansible-playbook)
  * [Step 6: Run a Syntax Check](#step-6-run-a-syntax-check)
  * [Step 7: Validate Connectivity with Ping Module](#step-7-validate-connectivity-with-ping-module)
  * [Step 8: Execute the Playbook](#step-8-execute-the-playbook)
* [Verification](#verification)
* [Best Practices](#best-practices)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)

---

## Overview

The Nautilus Application development team required the `logrotate` package to be installed across all application servers in the Stratos Datacenter before beginning application testing. Rather than performing this operation manually on each server, the DevOps team leveraged Ansible to automate the package installation from a single jump host, ensuring consistency, repeatability, and zero manual SSH interaction per node.

**Key objectives:**

* Create a structured inventory file at `/home/thor/playbook/inventory` on the jump host listing all target app servers
* Author a YAML playbook at `/home/thor/playbook/playbook.yml` that installs `logrotate` using the `yum` module
* Ensure the playbook is executable by user `thor` on the jump host with no additional arguments beyond `ansible-playbook -i inventory playbook.yml`

---

## Architecture

```
jump-host (thor)
     |
     |-- SSH --> stapp01 (tony)   [App Server 1]
     |-- SSH --> stapp02 (steve)  [App Server 2]
     |-- SSH --> stapp03 (banner) [App Server 3]
```

All playbook execution originates from the jump host. Ansible uses SSH with privilege escalation (`become: yes`) to install packages as root on each target node.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Ansible installed on jump host | `ansible [core 2.14.18]` |
| SSH access to all app servers | Verified manually before automation |
| Sudo privileges on all app servers | Required for `yum` package installation |
| Python 3 on all target nodes | Auto-discovered by Ansible as `/usr/bin/python3` |

---

## Implementation

### Step 1: Verify Ansible Installation

Before beginning, confirm the Ansible version and configuration on the jump host to establish the baseline environment.

```bash
ansible --version
```

**Output observed:**

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

> Screenshot: Ansible version output confirming core 2.14.18 on jump host

<img width="516" height="341" alt="image" src="https://github.com/user-attachments/assets/4a79ef0f-e84a-4f05-8027-679e45d27402" />

---

### Step 2: Validate SSH Connectivity to App Servers

Before writing any automation, manually validate SSH access to each application server to confirm credentials and reachability. This step prevents misleading playbook failures caused by connectivity issues unrelated to Ansible.

**Connect to stapp01:**

```bash
ssh tony@stapp01
```

**Connect to stapp02:**

```bash
ssh steve@stapp02
```

**Connect to stapp03:**

```bash
ssh banner@stapp03
```

Exit each session after confirming successful authentication:

```bash
exit
```

> Screenshot: Successful SSH login and logout for stapp01, stapp02, and stapp03

<img width="517" height="368" alt="image" src="https://github.com/user-attachments/assets/8aafd0b5-d587-4565-8442-c6a798355ddc" />

---

### Step 3: Create the Playbook Directory

Create the working directory structure to house the inventory and playbook files.

```bash
mkdir -p /home/thor/playbook
cd /home/thor/playbook
pwd
```

**Output observed:**

```
/home/thor/playbook
```

> Screenshot: Directory creation and confirmed working path at `/home/thor/playbook`

<img width="517" height="404" alt="image" src="https://github.com/user-attachments/assets/b93b06bd-06a0-437d-a7ea-ed8b11c55546" />

---

### Step 4: Create the Inventory File

Create the static inventory file at `/home/thor/playbook/inventory`. This file defines all three application servers as Ansible-managed hosts, along with their SSH credentials and privilege escalation configuration.

```bash
vi /home/thor/playbook/inventory
```

**Inventory file contents:**

```ini
stapp01 ansible_user=tony ansible_password=Ir0nM@n ansible_become=yes ansible_become_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_password=Am3ric@ ansible_become=yes ansible_become_pass=Am3ric@
stapp03 ansible_user=banner ansible_password=BigGr33n ansible_become=yes ansible_become_pass=BigGr33n
```

Verify the file contents after saving:

```bash
cat /home/thor/playbook/inventory
```

> Screenshots: `cat` output of the inventory file showing all three app server entries with credentials and become configuration

<img width="519" height="433" alt="image" src="https://github.com/user-attachments/assets/f684334d-5b82-41ce-bac2-f351e8233167" />
<img width="517" height="409" alt="image" src="https://github.com/user-attachments/assets/2f4cdd53-c2c9-4442-a192-e4cccc76b4b9" />
<img width="516" height="407" alt="image" src="https://github.com/user-attachments/assets/a09f9960-ae01-43a6-9bab-a602373deda7" />

**Inventory variable breakdown:**

| Variable | Purpose |
|---|---|
| `ansible_user` | SSH username for each app server |
| `ansible_password` | SSH password for authentication |
| `ansible_become` | Enables privilege escalation (`sudo`) |
| `ansible_become_pass` | Password used for `sudo` escalation |

---

### Step 5: Author the Ansible Playbook

Create the playbook at `/home/thor/playbook/playbook.yml`. The playbook targets all hosts in the inventory and uses the `yum` module to install `logrotate` in an idempotent `present` state.

```bash
vi /home/thor/playbook/playbook.yml
```

**Playbook contents:**

```yaml
---
- name: Install logrotate on all app servers
  hosts: all
  become: yes
  tasks:
    - name: Install logrotate package
      yum:
        name: logrotate
        state: present
```

Verify the file contents after saving:

```bash
cat /home/thor/playbook/playbook.yml
```

> Screenshots: `cat` output of the playbook confirming correct YAML structure, `hosts: all`, `become: yes`, and `yum` module usage

<img width="515" height="431" alt="image" src="https://github.com/user-attachments/assets/75f34007-2ae6-4611-9b7d-368658a42665" />
<img width="518" height="419" alt="image" src="https://github.com/user-attachments/assets/bfed3be4-3723-43de-945d-0b800b13d926" />
<img width="518" height="422" alt="image" src="https://github.com/user-attachments/assets/a1653075-2b67-4f30-acc1-40faf83e6659" />

**Playbook directive breakdown:**

| Directive | Value | Purpose |
|---|---|---|
| `hosts` | `all` | Targets every host defined in the inventory |
| `become` | `yes` | Elevates to root for package management |
| `yum.name` | `logrotate` | Package to install |
| `yum.state` | `present` | Ensures the package is installed; idempotent |

---

### Step 6: Run a Syntax Check

Before executing the playbook against live servers, validate the YAML syntax to catch any formatting or structural errors early.

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
```

**Output observed:**

```
playbook: playbook.yml
```

A clean return with the playbook filename and no errors confirms the YAML is syntactically valid.

> Screenshot: Syntax check output showing `playbook: playbook.yml` with no errors

<img width="517" height="416" alt="image" src="https://github.com/user-attachments/assets/2609d661-26f4-4962-b19e-4d87931157a0" />

---

### Step 7: Validate Connectivity with Ping Module

Run an Ansible ad-hoc ping against all inventory hosts to confirm that Ansible can authenticate, establish SSH connections, and discover Python on each managed node before running the full playbook.

```bash
ansible all -i inventory -m ping
```

**Output observed:**

```
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
stapp02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

All three nodes returned `SUCCESS` with `pong` and auto-discovered Python 3 at `/usr/bin/python3`.

> Screenshot: Ad-hoc ping output showing SUCCESS and pong response from stapp01, stapp02, and stapp03

<img width="516" height="425" alt="image" src="https://github.com/user-attachments/assets/b23ef89c-c63d-4741-aad5-02ee9ff41f63" />

---

### Step 8: Execute the Playbook

Run the playbook against the full inventory to install `logrotate` on all three application servers.

```bash
ansible-playbook -i inventory playbook.yml
```

**Output observed:**

```
PLAY [Install logrotate on all app servers] ************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [stapp01]
ok: [stapp03]
ok: [stapp02]

TASK [Install logrotate package] ***********************************************************************************************
changed: [stapp01]
changed: [stapp02]
changed: [stapp03]

PLAY RECAP *********************************************************************************************************************
stapp01                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

> Screenshot: Full playbook execution output confirming `changed=1` and `failed=0` across all three app servers

<img width="517" height="431" alt="image" src="https://github.com/user-attachments/assets/237ff90f-7b6a-4eba-8ad4-5e266cda47ca" />

**Play recap interpretation:**

| Field | Value | Meaning |
|---|---|---|
| `ok` | 2 | Two tasks ran successfully (Gathering Facts + Install) |
| `changed` | 1 | Package was not previously installed; state was changed |
| `unreachable` | 0 | All nodes were reachable via SSH |
| `failed` | 0 | No task failures on any node |

---

## Verification

To confirm idempotency, re-running the playbook on an already-provisioned inventory should return `changed=0` for the install task, demonstrating that Ansible correctly identifies the package as already present and takes no action.

```bash
ansible-playbook -i inventory playbook.yml
```

Expected recap after a second run:

```
stapp01                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

---

## Best Practices

* **Pre-automation SSH validation:** Manually verifying SSH connectivity to each node before writing inventory files prevents inventory credential issues from masking Ansible configuration problems during debugging.

* **Syntax check before execution:** Running `--syntax-check` as a mandatory pre-execution step catches YAML formatting errors without touching any managed hosts, preserving production server state during development.

* **Ad-hoc ping before playbook runs:** Using `ansible all -m ping` validates the full Ansible control path including authentication, SSH transport, and Python interpreter discovery before committing to a multi-task playbook execution.

* **`state: present` for idempotency:** Using `state: present` instead of `state: latest` ensures the playbook only makes a change when the package is absent. This prevents unintended upgrades during re-runs in environments where package versions must be pinned.

* **`become` at the play level:** Defining `become: yes` at the play level rather than the task level ensures all tasks in the play run with elevated privileges by default, reducing the risk of individual tasks silently failing due to missing permissions.

* **Flat inventory with per-host credentials:** For a small, heterogeneous node set with different credentials per host, inline inventory variables are appropriate. In larger environments, `host_vars/` directories or Ansible Vault-encrypted variable files should be used instead.

* **Directory isolation for playbook artifacts:** Placing both the inventory and playbook under a dedicated `/home/thor/playbook/` directory keeps automation artifacts organized and ensures the validation command `ansible-playbook -i inventory playbook.yml` resolves paths correctly from within that directory.

---

## Errors and Resolutions

### Typo in Terminal Session (`exitt`)

**Error encountered:**

```
[tony@stapp01 ~]$ exitt
-bash: exitt: command not found
```

**Root cause:** A typographical error was made when attempting to exit the SSH session on stapp01 after the initial connectivity check.

**Resolution:** The correct command `exit` was issued immediately after, closing the SSH session without any impact on the subsequent Ansible workflow.

---

## Lessons Learned

* **Validate before you automate.** Running a manual SSH test to each node before constructing the inventory revealed the correct usernames and confirmed password-based authentication was enabled. This eliminated a common category of inventory configuration errors before the Ansible layer was even introduced.

* **The ping module is a pre-flight checklist, not just a connectivity test.** Beyond confirming SSH reachability, `ansible -m ping` validates that Ansible can resolve the Python interpreter path on each node. A successful pong response means fact gathering, module execution, and privilege escalation are all structurally sound.

* **`changed` vs `ok` in play recaps carries operational meaning.** A `changed=1` result on the first run and `changed=0` on subsequent runs is the expected signature of a correctly written idempotent playbook. Monitoring this distinction in CI/CD pipelines can surface unexpected configuration drift across managed fleets.

* **Inline passwords in inventory files are acceptable for isolated training environments but must never be used in production.** The correct production approach is to use Ansible Vault to encrypt sensitive variables, or to configure SSH key-based authentication and manage sudo via `/etc/sudoers` with NOPASSWD entries scoped appropriately.

* **Execution order matters for reproducibility.** The sequence of directory creation, inventory authoring, playbook authoring, syntax check, ping validation, and final execution is not arbitrary. Each step builds confidence in the next, and collapsing this sequence risks introducing errors that are harder to isolate.
  
