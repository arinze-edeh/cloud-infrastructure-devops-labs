# Ansible Package Management: Automated httpd Installation and Service Management Across App Servers

## Table of Contents

- [Overview](#overview)
- [Architecture and Environment](#architecture-and-environment)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Inspect the Existing Ansible Directory](#step-1-inspect-the-existing-ansible-directory)
  - [Step 2: Review the Ansible Configuration File](#step-2-review-the-ansible-configuration-file)
  - [Step 3: Review the Inventory File](#step-3-review-the-inventory-file)
  - [Step 4: Author the Ansible Playbook](#step-4-author-the-ansible-playbook)
  - [Step 5: Update the Inventory with Privilege Escalation Credentials](#step-5-update-the-inventory-with-privilege-escalation-credentials)
  - [Step 6: Validate Playbook Syntax](#step-6-validate-playbook-syntax)
  - [Step 7: Execute the Playbook](#step-7-execute-the-playbook)
- [Playbook Reference](#playbook-reference)
- [Inventory Reference](#inventory-reference)
- [Execution Output](#execution-output)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project automates the installation and lifecycle management of the `httpd` (Apache HTTP Server) package across a fleet of application servers using Ansible. The playbook is authored on a centralized jump host and executed against three remote app servers via the existing inventory, requiring zero manual intervention on individual hosts.

The automation covers the full service lifecycle: package installation, service startup, and boot persistence, ensuring `httpd` is running and enabled on every target host after a single playbook run.

---

## Architecture and Environment

| Component | Details |
|-----------|---------|
| Control Node | `jump-host` (user: `thor`) |
| Target Hosts | `stapp01`, `stapp02`, `stapp03` |
| Inventory Path | `/home/thor/ansible/inventory` |
| Playbook Path | `/home/thor/ansible/playbook.yml` |
| Ansible Config | `/home/thor/ansible/ansible.cfg` |
| Package Managed | `httpd` (Apache HTTP Server) |
| Privilege Escalation | `become: yes` with per-host `ansible_become_pass` |

---

## Prerequisites

* Ansible installed and accessible on the jump host
* SSH connectivity from the jump host to all app servers
* Valid credentials for each app server user defined in inventory
* The `/home/thor/ansible/` directory must exist and contain the inventory and `ansible.cfg` files

---

## Project Structure

```
/home/thor/ansible/
├── ansible.cfg        # Ansible runtime configuration
├── inventory          # Static inventory with host connection and escalation credentials
└── playbook.yml       # Playbook to install, start, and enable httpd
```

---

## Implementation Guide

### Step 1: Inspect the Existing Ansible Directory

Begin by listing the contents of the Ansible working directory to confirm which files are already present before authoring anything new.

```bash
ls -la /home/thor/ansible/
```

**Output observed:**

```
total 20
drwxr-xr-x 2 thor thor 4096 May 16 05:14 .
drwx------ 1 thor thor 4096 May 16 05:14 ..
-rw-r--r-- 1 thor thor   36 May 16 05:14 ansible.cfg
-rw-r--r-- 1 thor thor  219 May 16 05:14 inventory
```

The `ansible.cfg` and `inventory` files are pre-provisioned. The `playbook.yml` does not yet exist and must be created.

*Screenshot: Directory listing confirming ansible.cfg and inventory are present*

<img width="506" height="317" alt="image" src="https://github.com/user-attachments/assets/d188976b-6027-4b79-b21c-a40f19adecaf" />

---

### Step 2: Review the Ansible Configuration File

Inspect `ansible.cfg` to understand the baseline runtime behavior configured for this environment.

```bash
cat /home/thor/ansible/ansible.cfg
```

**Output observed:**

```ini
[defaults]
host_key_checking = False
```

`host_key_checking = False` disables SSH host key verification. This is appropriate for ephemeral or controlled sandbox environments where host fingerprints are not pre-registered in `known_hosts`. In production environments this setting must be replaced with proper host key management.

*Screenshot: ansible.cfg content showing host_key_checking disabled*

<img width="508" height="319" alt="image" src="https://github.com/user-attachments/assets/383ae703-d670-456d-9e02-28d6a8c6d394" />

---

### Step 3: Review the Inventory File

Inspect the existing inventory to understand the target hosts, connection parameters, and user accounts defined for each app server.

```bash
cat /home/thor/ansible/inventory
```

**Initial inventory content observed:**

```ini
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner
```

Three app servers are defined with individual SSH usernames and passwords. At this stage, no `ansible_become_pass` directive is present, which will be required for privilege escalation when running tasks as root. This gap is addressed in Step 5.

*Screenshot: Initial inventory file contents*

---

### Step 4: Author the Ansible Playbook

Create the playbook at `/home/thor/ansible/playbook.yml`. The playbook must target all hosts in the inventory, install `httpd`, and ensure the service is started and enabled to survive reboots.

```bash
vi /home/thor/ansible/playbook.yml
```

Write the following content:

```yaml
---
- name: Install and enable httpd on all app servers
  hosts: all
  become: yes
  tasks:
    - name: Install httpd package
      package:
        name: httpd
        state: present
    - name: Start and enable httpd service
      service:
        name: httpd
        state: started
        enabled: yes
```

**Design decisions:**

* `hosts: all` targets every host group in the inventory without requiring a group name argument at runtime, satisfying the validation requirement of running `ansible-playbook -i inventory playbook.yml` with no extra arguments.
* `become: yes` at the play level applies privilege escalation to every task, as both package installation and service management require root permissions.
* The `package` module is used instead of `yum` or `dnf` to keep the playbook distribution-agnostic.
* The `service` module handles both the immediate start (`state: started`) and boot persistence (`enabled: yes`) in a single task.

*Screenshot: playbook.yml open in vi with the completed content*

---

### Step 5: Update the Inventory with Privilege Escalation Credentials

The initial inventory lacks `ansible_become_pass`, which Ansible requires to escalate from the SSH user to root on each app server. Edit the inventory to add this directive for all three hosts.

```bash
vi /home/thor/ansible/inventory
```

**Updated inventory content:**

```ini
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony ansible_become_pass=Ir0nM@n
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve ansible_become_pass=Am3ric@
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner ansible_become_pass=BigGr33n
```

Each `ansible_become_pass` value matches the respective SSH password for that host, reflecting the environment's configuration where the SSH user password also serves as the sudo password.

Verify the final inventory state:

```bash
cat /home/thor/ansible/inventory
```

*Screenshot: Updated inventory file with ansible_become_pass added for all three hosts*

---

### Step 6: Validate Playbook Syntax

Before executing, perform a syntax check to catch any YAML formatting or structural errors in the playbook without making any changes to the target hosts.

```bash
cd /home/thor/ansible/
ansible-playbook -i inventory playbook.yml --syntax-check
```

**Output observed:**

```
playbook: playbook.yml
```

A clean output with only the playbook name returned confirms the playbook passes syntax validation with no errors.

*Screenshot: Syntax check output showing playbook.yml with no errors*

---

### Step 7: Execute the Playbook

Run the playbook using the exact command that validation will use, ensuring it works without any additional arguments beyond the inventory flag.

```bash
ansible-playbook -i inventory playbook.yml
```

*Screenshot: Full playbook execution output in the terminal*

---

## Playbook Reference

```yaml
---
- name: Install and enable httpd on all app servers
  hosts: all
  become: yes
  tasks:
    - name: Install httpd package
      package:
        name: httpd
        state: present
    - name: Start and enable httpd service
      service:
        name: httpd
        state: started
        enabled: yes
```

---

## Inventory Reference

```ini
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony ansible_become_pass=Ir0nM@n
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve ansible_become_pass=Am3ric@
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner ansible_become_pass=BigGr33n
```

---

## Execution Output

The playbook completed successfully across all three app servers with zero failures.

```
PLAY [Install and enable httpd on all app servers] *****************************************************

TASK [Gathering Facts] *********************************************************************************
ok: [stapp01]
ok: [stapp02]
ok: [stapp03]

TASK [Install httpd package] ***************************************************************************
changed: [stapp01]
changed: [stapp02]
changed: [stapp03]

TASK [Start and enable httpd service] ******************************************************************
changed: [stapp01]
changed: [stapp02]
changed: [stapp03]

PLAY RECAP *********************************************************************************************
stapp01                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

**Result interpretation:**

* `ok=3` on every host confirms the fact-gathering task and both configuration tasks were reached and processed.
* `changed=2` on every host confirms `httpd` was installed and the service was started and enabled, reflecting real state changes applied to previously unconfigured servers.
* `failed=0`, `unreachable=0` across all hosts confirms full success with no connectivity or execution errors.

---

## Best Practices Applied

* **Idempotent task design:** Both the `package` and `service` modules are inherently idempotent. Re-running the playbook on already-configured hosts produces `ok` instead of `changed`, with no unintended side effects.

* **Distribution-agnostic module selection:** Using the `package` module instead of `yum` or `dnf` allows the playbook to function on hosts running different Linux distributions without modification.

* **Play-level privilege escalation:** Setting `become: yes` at the play level rather than per-task ensures consistent escalation behavior and reduces the risk of individual tasks silently running without root permissions if the directive were accidentally omitted from a task.

* **Single-command execution compliance:** The playbook is structured so that validation runs correctly using `ansible-playbook -i inventory playbook.yml` with no additional flags or arguments, as required by the environment's automated validation process.

* **Syntax validation before execution:** Running `--syntax-check` before the live execution pass catches structural issues early without touching any target host state.

* **Per-host `ansible_become_pass` in inventory:** Rather than using a vault or a global `become` password prompt, `ansible_become_pass` is defined per host in the inventory, allowing the playbook to run fully unattended and without interactive credential prompts.

* **Combined service state and enablement in a single task:** Using `state: started` and `enabled: yes` together in one `service` task ensures both immediate availability and boot persistence are enforced atomically, preventing a state where `httpd` starts but is not configured to survive a reboot.

---

## Lessons Learned

**`ansible_become_pass` is required even when SSH credentials are known.**
The initial inventory contained SSH connection credentials (`ansible_ssh_pass`, `ansible_user`) but omitted `ansible_become_pass`. Without this, any task that calls `become: yes` would fail with a sudo password prompt, blocking unattended execution. In environments where the same password serves both SSH and sudo purposes, the value is simply duplicated across both directives in the inventory.

**`hosts: all` eliminates the need for group arguments at runtime.**
Using `hosts: all` in the play definition means the playbook runs against every host in the inventory without requiring a `--limit` flag or group name in the execution command. This is critical when the validation mechanism runs a fixed command with no extra arguments.

**Syntax checks are a low-cost safety gate.**
Running `--syntax-check` before live execution costs negligible time and provides an early signal that the YAML structure and module references are valid. It does not test connectivity or idempotency, but it prevents the frustrating experience of watching a playbook fail mid-run due to a formatting error that could have been caught before a single SSH connection was made.

**The `package` module is preferable to distribution-specific modules for cross-host portability.**
In environments where target hosts may run different distributions (RHEL, CentOS, Ubuntu), using `package` instead of `yum` or `apt` avoids the need for conditional logic or separate plays per OS family, keeping the playbook clean and maintainable.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|--------|-------------|-----------|
| `Missing sudo password` error during play | `ansible_become_pass` not set in inventory | Add `ansible_become_pass` for each host in the inventory file |
| `Host key verification failed` | `host_key_checking` not disabled | Set `host_key_checking = False` in `ansible.cfg` or pre-register host keys |
| `Authentication failure` on SSH | Wrong `ansible_ssh_pass` or `ansible_user` | Verify credentials in the inventory against the actual host configuration |
| `Package not found` error | `httpd` not available in the host's configured repositories | Confirm the package manager repository configuration on the target host |
| `changed=0` when `httpd` was already installed | Idempotency working as expected | No action required; `package` module detected the package is already present |
| Playbook runs but service is not accessible | Firewall blocking port 80/443 | Open the required ports using `firewalld` or `iptables` on the target hosts |











<img width="508" height="336" alt="image" src="https://github.com/user-attachments/assets/f0084738-7129-4127-bd41-3782f777a8ae" />
<img width="511" height="419" alt="image" src="https://github.com/user-attachments/assets/cbf27550-400f-4f65-8516-c80b78d8ef8d" />
<img width="506" height="313" alt="image" src="https://github.com/user-attachments/assets/a7b4cb07-8c53-4517-87c1-bd2ba9f953dd" />
<img width="530" height="421" alt="image" src="https://github.com/user-attachments/assets/2c3865f2-a0ab-4d89-a916-6f26e91a24af" />
<img width="532" height="353" alt="image" src="https://github.com/user-attachments/assets/777999f8-0a97-45e4-9391-91df67392cf6" />
<img width="530" height="422" alt="image" src="https://github.com/user-attachments/assets/67001fd6-ea92-4bc6-9042-5a82d1eb8933" />
<img width="529" height="419" alt="image" src="https://github.com/user-attachments/assets/c94de91e-f479-413f-8564-76337845e76b" />
<img width="530" height="332" alt="image" src="https://github.com/user-attachments/assets/51987884-87d8-464b-955e-499b2ca81ea5" />
<img width="530" height="407" alt="image" src="https://github.com/user-attachments/assets/201e7022-be88-4eff-9ee4-da855e830815" />


