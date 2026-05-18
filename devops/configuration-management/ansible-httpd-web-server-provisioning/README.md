# Ansible: Automated Apache HTTP Server Deployment Across Multi-Node Infrastructure

## Table of Contents

* [Overview](#overview)
* [Infrastructure Details](#infrastructure-details)
* [Architecture and Solution Design](#architecture-and-solution-design)
* [Prerequisites](#prerequisites)
* [Project Structure](#project-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Inspect Working Directory and Inventory](#step-1-inspect-working-directory-and-inventory)
  * [Step 2: Verify Ansible Version and Runtime Configuration](#step-2-verify-ansible-version-and-runtime-configuration)
  * [Step 3: Change into Working Directory and Validate Connectivity](#step-3-change-into-working-directory-and-validate-connectivity)
  * [Step 4: Author the Ansible Playbook](#step-4-author-the-ansible-playbook)
  * [Step 5: Confirm Playbook on Disk](#step-5-confirm-playbook-on-disk)
  * [Step 6: Syntax Validation](#step-6-syntax-validation)
  * [Step 7: Execute the Playbook](#step-7-execute-the-playbook)
  * [Step 8: Post-Deployment Verification](#step-8-post-deployment-verification)
* [Playbook Reference](#playbook-reference)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Outcome Summary](#outcome-summary)

---

## Overview

This implementation automates the provisioning and configuration of the Apache HTTP Server (`httpd`) across all three application servers in the Stratos DC environment. Using Ansible from a centralized jump host, the playbook installs the web server package, starts and enables the service, deploys a custom `index.html` file with the correct content and ownership, and uses the `lineinfile` module to prepend an additional line at the top of the HTML file.

All tasks execute idempotently and in a defined order, with no manual SSH intervention required on any target node beyond final verification.

**Tooling:** Ansible Core 2.14.18 on Python 3.9

**Execution Host:** `jump-host` (thor)

**Target Hosts:** `stapp01`, `stapp02`, `stapp03`

---

## Infrastructure Details

| Server Name | Hostname | User | Purpose |
|---|---|---|---|
| Application Server 1 | stapp01 | tony | Hosts Nautilus Application 1 |
| Application Server 2 | stapp02 | steve | Hosts Nautilus Application 2 |
| Application Server 3 | stapp03 | banner | Hosts Nautilus Application 3 |
| Jump Host Server | jump-host | thor | Provides secure access to Stratos DC |

---

## Architecture and Solution Design

The solution follows a controller-to-node Ansible push model:

```
jump-host (thor)
    |
    |-- ansible -i inventory all -m ping     [connectivity check]
    |
    |-- ansible-playbook -i inventory playbook.yml
            |
            |-- stapp01 (tony) --> httpd installed, started, index.html deployed
            |-- stapp02 (steve) --> httpd installed, started, index.html deployed
            |-- stapp03 (banner) --> httpd installed, started, index.html deployed
```

The `index.html` final state across all nodes:

```
Welcome to xFusionCorp Industries!
This is a Nautilus sample file, created using Ansible!
```

The `lineinfile` module with `insertbefore: BOF` ensures the welcome line is prepended above any existing content, resulting in a deterministic two-line output.

---

## Prerequisites

* Ansible Core 2.14+ installed on the jump host
* SSH access to all application servers via password-based authentication
* `host_key_checking = False` configured in `ansible.cfg` to suppress SSH host key prompts in a trusted internal network
* All target hosts reachable by hostname from the jump host

---

## Project Structure

```
/home/thor/ansible/
├── ansible.cfg       # Ansible runtime configuration (host key checking disabled)
├── inventory         # Static inventory with per-host SSH credentials
└── playbook.yml      # Main playbook (authored during this implementation)
```

---

## Implementation Guide

### Step 1: Inspect Working Directory and Inventory

From the home directory on the jump host, the ansible working directory was listed to confirm the files present, followed by reading the inventory to understand the target hosts and their credentials.

```bash
ls -la /home/thor/ansible/
```

```
total 20
drwxr-xr-x 2 thor thor 4096 May 18 03:36 .
drwx------ 1 thor thor 4096 May 18 03:36 ..
-rw-r--r-- 1 thor thor   36 May 18 03:36 ansible.cfg
-rw-r--r-- 1 thor thor  219 May 18 03:36 inventory
```

```bash
cat /home/thor/ansible/inventory
```

```
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner
```

**Screenshot: Directory listing and inventory file contents on jump-host**

<img width="511" height="369" alt="image" src="https://github.com/user-attachments/assets/de1a8bcc-012b-42a3-ac2e-e0c75d940403" />

---

### Step 2: Verify Ansible Version and Runtime Configuration

Still from the home directory, the installed Ansible version was checked, then the `ansible.cfg` was read to confirm the runtime configuration in place.

```bash
ansible --version
```

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

```bash
cat /home/thor/ansible/ansible.cfg
```

```
[defaults]
host_key_checking = False
```

**Screenshot: Ansible version output and ansible.cfg contents**

<img width="511" height="386" alt="image" src="https://github.com/user-attachments/assets/aaa52578-aeba-4ce2-858c-dbd1a3d433c2" />

---

### Step 3: Change into Working Directory and Validate Connectivity

The working directory was changed to `/home/thor/ansible/`, then a live ping was issued against all inventory hosts to confirm end-to-end SSH connectivity before writing any playbook.

```bash
cd /home/thor/ansible/
ansible -i inventory all -m ping
```

```
stapp02 | SUCCESS => {
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
stapp03 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

All three application servers returned `pong`, confirming SSH connectivity and Python interpreter availability on every target node.

**Screenshot: Ansible ping output confirming connectivity to stapp01, stapp02, stapp03**

---

### Step 4: Author the Ansible Playbook

The playbook was written using a heredoc directly on the jump host into `/home/thor/ansible/playbook.yml`. The playbook targets all hosts in the inventory and runs with privilege escalation (`become: yes`) to execute package management and service control as root.

Four tasks were implemented in execution order:

1. **Install `httpd`** using the `package` module for OS-agnostic package management
2. **Start and enable `httpd`** using the `service` module, ensuring the web server is both immediately active and persistent across reboots
3. **Deploy `/var/www/html/index.html`** using the `copy` module with inline content, setting `owner`, `group`, and `mode` as specified
4. **Prepend the welcome line** using the `lineinfile` module with `insertbefore: BOF` to inject the line at the top of the file

```bash
cat > /home/thor/ansible/playbook.yml << 'EOF'
---
- name: Install and configure httpd on all app servers
  hosts: all
  become: yes

  tasks:

    - name: Install httpd web server
      package:
        name: httpd
        state: present

    - name: Start and enable httpd service
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Create /var/www/html/index.html with initial content
      copy:
        dest: /var/www/html/index.html
        content: "This is a Nautilus sample file, created using Ansible!"
        owner: apache
        group: apache
        mode: "0655"

    - name: Insert Welcome line at the top using lineinfile
      lineinfile:
        path: /var/www/html/index.html
        line: "Welcome to xFusionCorp Industries!"
        insertbefore: BOF
        owner: apache
        group: apache
        mode: "0655"
EOF
```

**Screenshot: Heredoc playbook write command on jump-host**

---

### Step 5: Confirm Playbook on Disk

Immediately after writing the playbook, its contents were read back to verify the heredoc wrote correctly with no truncation or formatting errors.

```bash
cat /home/thor/ansible/playbook.yml
```

**Screenshot: Full playbook content displayed via cat on jump-host**

---

### Step 6: Syntax Validation

Before executing the playbook against live infrastructure, a dry-run syntax check was performed to catch any YAML or Ansible structural errors.

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
```

```
playbook: playbook.yml
```

The clean output (returning only the playbook filename with no errors) confirmed the playbook was syntactically valid and safe to execute.

**Screenshot: Syntax check output confirming zero errors**

---

### Step 7: Execute the Playbook

With connectivity verified and syntax validated, the playbook was executed against all target hosts.

```bash
ansible-playbook -i inventory playbook.yml
```

```
PLAY [Install and configure httpd on all app servers] **************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [stapp03]
ok: [stapp02]
ok: [stapp01]

TASK [Install httpd web server] ************************************************************************************************
changed: [stapp03]
changed: [stapp01]
changed: [stapp02]

TASK [Start and enable httpd service] ******************************************************************************************
changed: [stapp03]
changed: [stapp02]
changed: [stapp01]

TASK [Create /var/www/html/index.html with initial content] ********************************************************************
changed: [stapp03]
changed: [stapp01]
changed: [stapp02]

TASK [Insert Welcome line at the top using lineinfile] *************************************************************************
changed: [stapp03]
changed: [stapp02]
changed: [stapp01]

PLAY RECAP *********************************************************************************************************************
stapp01                    : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp02                    : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp03                    : ok=5    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

All five tasks (including `Gathering Facts`) completed with `ok=5` and `changed=4` on every host. Zero failures, zero unreachable, zero skipped.

**Screenshot: Full playbook execution output with PLAY RECAP showing ok=5 changed=4 on all three hosts**

---

### Step 8: Post-Deployment Verification

SSH access to `stapp01` was used to manually verify the three key outcomes: file content, file ownership and permissions, and service state.

```bash
ssh tony@stapp01
```

```bash
cat /var/www/html/index.html && ls -la /var/www/html/index.html && systemctl is-active httpd
```

```
Welcome to xFusionCorp Industries!
This is a Nautilus sample file, created using Ansible!-rw-r-xr-x 1 apache apache 89 May 18 03:53 /var/www/html/index.html
active
```

Verification confirmed:

* The `index.html` file contains both lines in the correct order, with the welcome message at the top
* File ownership is `apache:apache`
* File permissions are `0655` (`-rw-r-xr-x`)
* The `httpd` service is in an `active` (running) state

**Screenshot: SSH session on stapp01 showing cat, ls -la, and systemctl is-active output**

---

## Playbook Reference

```yaml
---
- name: Install and configure httpd on all app servers
  hosts: all
  become: yes

  tasks:

    - name: Install httpd web server
      package:
        name: httpd
        state: present

    - name: Start and enable httpd service
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Create /var/www/html/index.html with initial content
      copy:
        dest: /var/www/html/index.html
        content: "This is a Nautilus sample file, created using Ansible!"
        owner: apache
        group: apache
        mode: "0655"

    - name: Insert Welcome line at the top using lineinfile
      lineinfile:
        path: /var/www/html/index.html
        line: "Welcome to xFusionCorp Industries!"
        insertbefore: BOF
        owner: apache
        group: apache
        mode: "0655"
```

---

## Best Practices Applied

* **`package` module over `yum`/`apt`:** Using the `package` module keeps the playbook OS-agnostic. If a future target runs Ubuntu or a different distribution, the same playbook executes without modification.

* **`become: yes` at play level:** Privilege escalation is declared at the play level rather than per-task, keeping the playbook DRY and applying consistent escalation across all tasks that require root.

* **`state: started` and `enabled: yes` on the service task:** Ensuring both the immediate running state and the boot persistence setting are explicitly declared prevents scenarios where the service is running but not enabled (or enabled but currently stopped) from passing silently.

* **`copy` module with inline `content`:** Using `content:` within the `copy` module eliminates the need for a template directory or separate file, making the playbook fully self-contained and portable.

* **`lineinfile` with `insertbefore: BOF`:** The `BOF` (Beginning of File) anchor is the correct and explicit way to prepend content with `lineinfile`. This avoids fragile line-number-based indexing and works correctly regardless of the file's initial state.

* **Ownership and mode set on both `copy` and `lineinfile` tasks:** Setting `owner`, `group`, and `mode` on both the creation and modification tasks ensures permissions are enforced regardless of execution order or whether one task creates the file fresh or operates on an existing file.

* **Connectivity pre-check before playbook execution:** Running `ansible -i inventory all -m ping` before the playbook catches inventory or SSH issues early, avoiding partial-run side effects on live servers.

* **Syntax validation before live execution:** `--syntax-check` acts as a fast, zero-impact gate that catches structural YAML errors before any connection is made to target hosts.

* **`host_key_checking = False` scoped to `ansible.cfg`:** Disabling host key checking at the config level rather than via environment variable or command flag ensures consistent behavior across all ad-hoc commands and playbook runs in this environment.

---

## Lessons Learned

* **`lineinfile` is idempotent but order-sensitive with `copy`:** The `copy` task must run before `lineinfile` because `lineinfile` requires the file to already exist at the target path. If the order were reversed, `lineinfile` would either fail or create an incomplete file. Establishing the file first with `copy` and then modifying it with `lineinfile` is the correct and deterministic sequence.

* **`insertbefore: BOF` does not add a newline separator automatically:** The resulting file has the inserted line immediately adjacent to the existing content on the next line. This is expected behavior. If a blank line between the two lines were required, a separate `lineinfile` task or a `blockinfile` approach would be needed. In this implementation the two-line output was the intended result.

* **File permission `0655` vs. `0644`:** The required permissions (`0655`) grant execute permission to group and others. This is atypical for HTML files where `0644` is the standard. Applying exactly what the task specification required while noting this deviation is the correct approach in an environment with defined compliance requirements.

* **Verifying output on at least one target node after playbook execution:** Playbook `PLAY RECAP` showing zero failures is a necessary but not sufficient final check. SSHing into a representative node and running `cat`, `ls -la`, and `systemctl is-active` adds a human-readable ground-truth layer that catches any edge cases where tasks report `ok` but produce incorrect state.

* **`discovered_interpreter_python` in `Gathering Facts`:** Ansible auto-discovering the Python interpreter path on each node confirms that the control node and target nodes are correctly aligned for module execution. If this were missing or falling back to Python 2, module behavior could differ across tasks.

---

## Outcome Summary

| Task | Result |
|---|---|
| `httpd` installed on all app servers | Confirmed (changed on stapp01, stapp02, stapp03) |
| `httpd` service started and enabled | Confirmed (active on stapp01, stapp02, stapp03) |
| `/var/www/html/index.html` created with correct content | Confirmed |
| Welcome line prepended using `lineinfile` | Confirmed |
| File ownership set to `apache:apache` | Confirmed |
| File permissions set to `0655` | Confirmed |
| Zero failures across all hosts | Confirmed (PLAY RECAP: failed=0 on all) |






<img width="511" height="312" alt="image" src="https://github.com/user-attachments/assets/0aa7ef92-a5a7-46ef-a250-0d4ba9fb1557" />

<img width="512" height="348" alt="image" src="https://github.com/user-attachments/assets/d6e3dbb2-b141-4c5b-b2b1-3f8e2f8f9292" />

<img width="511" height="319" alt="image" src="https://github.com/user-attachments/assets/b0b5387a-0c9a-47d4-8231-01c4ea558207" />
<img width="517" height="435" alt="image" src="https://github.com/user-attachments/assets/2501467f-22f1-4d59-83d4-b60bd0512c77" />
<img width="514" height="434" alt="image" src="https://github.com/user-attachments/assets/afdde6e9-9c37-4d3f-9af3-9370bbf4e769" />
<img width="514" height="401" alt="image" src="https://github.com/user-attachments/assets/5cb75ae0-b571-4b3b-94c6-3f87c1f58f95" />
<img width="516" height="393" alt="image" src="https://github.com/user-attachments/assets/41dd05a9-fcc4-4891-8647-199cbaf6cbe4" />
<img width="518" height="388" alt="image" src="https://github.com/user-attachments/assets/3a297613-b891-4750-b170-22938b419dab" />
<img width="518" height="250" alt="image" src="https://github.com/user-attachments/assets/6b6c96d9-3213-45c3-9845-6dbc52d214c4" />

