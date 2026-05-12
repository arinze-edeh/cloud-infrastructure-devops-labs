# Ansible File Creation and Permission Management Across Remote Hosts

> Automating file provisioning, permission enforcement, and ownership assignment on distributed application servers using Ansible's `file` module in a multi-host Stratos DC environment.

---

## Table of Contents

- [Overview](#overview)
- [Architecture and Environment](#architecture-and-environment)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Create the Ansible Inventory](#step-1-create-the-ansible-inventory)
  - [Step 2: Configure Ansible Defaults](#step-2-configure-ansible-defaults)
  - [Step 3: Validate Connectivity with Ping Module](#step-3-validate-connectivity-with-ping-module)
  - [Step 4: Author the Playbook](#step-4-author-the-playbook)
  - [Step 5: Syntax Check](#step-5-syntax-check)
  - [Step 6: Dry Run with Check Mode](#step-6-dry-run-with-check-mode)
  - [Step 7: Execute the Playbook](#step-7-execute-the-playbook)
  - [Step 8: Verify File Creation and Permissions on Each Host](#step-8-verify-file-creation-and-permissions-on-each-host)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Project Structure](#project-structure)

---

## Overview

This project demonstrates a structured Ansible approach to creating a managed file (`/usr/src/appdata.txt`) across all application servers in the Stratos Datacenter. The task enforces consistent file permissions (`0655`), and assigns the correct user and group ownership per host using dynamic variable resolution at runtime.

The implementation validates the idempotency of the `file` module and follows a gated execution workflow: inventory validation, connectivity verification, syntax check, dry run, and live execution with post-deployment SSH verification.

**Key outcomes delivered:**

* Blank file `/usr/src/appdata.txt` created on all three application servers
* Permissions set to `0655` on every target host
* User and group ownership assigned dynamically: `tony` on `stapp01`, `steve` on `stapp02`, and `banner` on `stapp03`
* All operations executed via a single playbook invocation with no additional arguments required

---

## Architecture and Environment

| Component | Detail |
|---|---|
| Control Node | `jump-host` (thor user) |
| Target Hosts | `stapp01`, `stapp02`, `stapp03` |
| Inventory Group | `app_servers` |
| Working Directory | `~/playbook` |
| Managed File Path | `/usr/src/appdata.txt` |
| File Permissions | `0655` |
| Privilege Escalation | `sudo` via `ansible_become` |
| Ansible Module Used | `file` |
| Python Interpreter | `/usr/bin/python3` (auto-discovered) |

**Host-to-User Mapping:**

| Host | SSH User | Sudo Password |
|---|---|---|
| `stapp01` | `tony` | `Ir0nM@n` |
| `stapp02` | `steve` | `Am3ric@` |
| `stapp03` | `banner` | `BigGr33n` |

---

## Prerequisites

* Ansible installed and available on the control node (`jump-host`)
* SSH connectivity from `jump-host` to all three application servers
* Target users (`tony`, `steve`, `banner`) exist on their respective hosts and have `sudo` privileges
* Working directory `~/playbook` exists or is created before execution

---

## Implementation Guide

### Step 1: Create the Ansible Inventory

Navigate to the working directory and create the inventory file defining all three application servers under a single group `[app_servers]`. Each entry specifies the SSH user, SSH password, privilege escalation flag, and the escalation password for `sudo` operations.

```bash
cat > inventory << 'EOF'
[app_servers]
stapp01 ansible_user=tony ansible_password=Ir0nM@n ansible_become=true ansible_become_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_password=Am3ric@ ansible_become=true ansible_become_pass=Am3ric@
stapp03 ansible_user=banner ansible_password=BigGr33n ansible_become=true ansible_become_pass=BigGr33n
EOF
```

Confirm the file was written correctly:

```bash
cat inventory
```

**Expected output:**

```
[app_servers]
stapp01 ansible_user=tony ansible_password=Ir0nM@n ansible_become=true ansible_become_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_password=Am3ric@ ansible_become=true ansible_become_pass=Am3ric@
stapp03 ansible_user=banner ansible_password=BigGr33n ansible_become=true ansible_become_pass=BigGr33n
```

*Screenshot: Terminal output of `cat inventory` confirming all three host entries are correctly defined with their respective credentials and become parameters.*

---

### Step 2: Configure Ansible Defaults

Create an `ansible.cfg` file in the working directory to disable SSH host key checking. This is required in dynamic lab and development environments where hosts may not have pre-established known-host entries.

```bash
cat > ansible.cfg << 'EOF'
[defaults]
host_key_checking = False
EOF
```

*Screenshot: Terminal output confirming creation of `ansible.cfg` with `host_key_checking = False`.*

---

### Step 3: Validate Connectivity with Ping Module

Before writing a single task, validate that Ansible can reach all managed hosts using the built-in `ping` module. This confirms inventory correctness, credential validity, and network reachability in a single command.

```bash
ansible all -i inventory -m ping
```

**Expected output:**

```
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
stapp03 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

All three hosts returned `SUCCESS` with `pong`, confirming connectivity and Python interpreter availability on each target.

*Screenshot: Full terminal output of the `ansible all -i inventory -m ping` command showing `SUCCESS` for `stapp01`, `stapp02`, and `stapp03`.*

---

### Step 4: Author the Playbook

Write the playbook that targets the `app_servers` group and creates `/usr/src/appdata.txt` with the `file` module. The `ansible_user` variable is used dynamically to assign ownership, ensuring each host receives the correct user and group mapping without hardcoding.

```bash
cat > playbook.yml << 'EOF'
---
- name: Create appdata.txt on all app servers
  hosts: app_servers
  become: true

  tasks:

    - name: Create blank file /usr/src/appdata.txt
      file:
        path: /usr/src/appdata.txt
        state: touch
        mode: "0655"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
EOF
```

Confirm the playbook was written correctly:

```bash
cat playbook.yml
```

**Expected output:**

```yaml
---
- name: Create appdata.txt on all app servers
  hosts: app_servers
  become: true

  tasks:

    - name: Create blank file /usr/src/appdata.txt
      file:
        path: /usr/src/appdata.txt
        state: touch
        mode: "0655"
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
```

**Design decision:** Using `{{ ansible_user }}` as both `owner` and `group` allows a single playbook definition to correctly map ownership to `tony`, `steve`, and `banner` without conditional branching. The `ansible_user` inventory variable is already defined per host, so it resolves at runtime with zero additional configuration.

*Screenshot: Terminal output of `cat playbook.yml` confirming playbook content with correct indentation, module parameters, and Jinja2 variable references.*

---

### Step 5: Syntax Check

Run a syntax validation pass against the playbook before any execution. This catches YAML indentation errors, module name typos, and structural issues without making any changes to target hosts.

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
```

**Expected output:**

```
playbook: playbook.yml
```

A clean output with only the playbook filename confirms no syntax errors were detected.

*Screenshot: Terminal output of `--syntax-check` showing `playbook: playbook.yml` with no error messages.*

---

### Step 6: Dry Run with Check Mode

Execute the playbook in check mode (`--check`) to simulate what changes would occur without applying them to the remote hosts. This is the final gate before a live run and allows validation of task logic against real inventory.

```bash
ansible-playbook -i inventory playbook.yml --check
```

**Expected output:**

```
PLAY [Create appdata.txt on all app servers] *****

TASK [Gathering Facts] ***************************
ok: [stapp01]
ok: [stapp02]
ok: [stapp03]

TASK [Create blank file /usr/src/appdata.txt] ****
changed: [stapp01]
changed: [stapp02]
changed: [stapp03]

PLAY RECAP ***************************************
stapp01 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

The check mode output confirms that the task will result in `changed=1` on all three hosts, indicating the file does not yet exist and will be created upon live execution.

*Screenshot: Full terminal output of `ansible-playbook --check` showing `changed: [stapp01]`, `changed: [stapp02]`, and `changed: [stapp03]` in the task summary, and a clean PLAY RECAP with zero failures.*

---

### Step 7: Execute the Playbook

With connectivity, syntax, and dry-run validation all passing, execute the playbook against the live inventory.

```bash
ansible-playbook -i inventory playbook.yml
```

**Expected output:**

```
PLAY [Create appdata.txt on all app servers] *****

TASK [Gathering Facts] ***************************
ok: [stapp02]
ok: [stapp01]
ok: [stapp03]

TASK [Create blank file /usr/src/appdata.txt] ****
changed: [stapp02]
changed: [stapp01]
changed: [stapp03]

PLAY RECAP ***************************************
stapp01 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03 : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

All three hosts reported `changed=1` with zero failures, zero unreachable, and zero skipped tasks. The playbook completed successfully across the entire `app_servers` group.

*Screenshot: Full terminal output of the live `ansible-playbook -i inventory playbook.yml` execution showing `changed` status on all three hosts and a clean PLAY RECAP.*

---

### Step 8: Verify File Creation and Permissions on Each Host

After playbook execution, SSH into each application server individually and inspect the file using `ls -l` to confirm the path, permissions, owner, and group are all correct.

**Verify on stapp01 (tony):**

```bash
ssh tony@stapp01 "ls -l /usr/src/appdata.txt"
```

**Expected output:**

```
-rw-r-xr-x 1 tony tony 0 May 12 05:44 /usr/src/appdata.txt
```

*Screenshot: Terminal output of the SSH command to `stapp01` showing `-rw-r-xr-x 1 tony tony 0` confirming correct permissions and ownership.*

---

**Verify on stapp02 (steve):**

```bash
ssh steve@stapp02 "ls -l /usr/src/appdata.txt"
```

**Expected output:**

```
-rw-r-xr-x 1 steve steve 0 May 12 05:44 /usr/src/appdata.txt
```

*Screenshot: Terminal output of the SSH command to `stapp02` showing `-rw-r-xr-x 1 steve steve 0` confirming correct permissions and ownership.*

---

**Verify on stapp03 (banner):**

```bash
ssh banner@stapp03 "ls -l /usr/src/appdata.txt"
```

**Expected output:**

```
-rw-r-xr-x 1 banner banner 0 May 12 05:44 /usr/src/appdata.txt
```

*Screenshot: Terminal output of the SSH command to `stapp03` showing `-rw-r-xr-x 1 banner banner 0` confirming correct permissions and ownership.*

---

All three verification checks confirmed:

* File path: `/usr/src/appdata.txt`
* Permissions: `-rw-r-xr-x` (octal `0655`)
* Owner and group resolved correctly per host: `tony`, `steve`, `banner`
* File size: `0` (blank file, as intended)

---

## Best Practices Applied

**Gated execution workflow:** The implementation followed a structured pre-execution validation pipeline: inventory verification via `cat`, connectivity check via `ping`, syntax validation via `--syntax-check`, and change simulation via `--check` before the live run. This prevents unexpected changes in production environments.

**Dynamic variable resolution for ownership:** Rather than defining separate tasks or using `when` conditionals for each host, `{{ ansible_user }}` was used to resolve the correct owner and group at runtime. This keeps the playbook DRY (Do Not Repeat Yourself) and eliminates host-specific branching.

**`ansible.cfg` scoped to the project directory:** Placing `ansible.cfg` inside the `~/playbook` working directory scopes configuration to the project and avoids contaminating global Ansible settings. This is critical in environments with multiple projects using different configuration profiles.

**`host_key_checking = False` for ephemeral environments:** Disabling host key checking in environments where SSH keys are not pre-distributed prevents connection failures from unknown host fingerprints. In production, this setting should be replaced with proper known-hosts management or SSH key distribution.

**Post-execution SSH verification:** Rather than relying solely on Ansible's task output, direct SSH-based file inspection was used to independently confirm the outcome on each host. This closes the loop between reported task results and actual remote state.

**`become: true` at the play level:** Applying privilege escalation at the play level rather than per task ensures consistent privilege context across all tasks in the play and reduces the risk of tasks silently running without escalation.

---

## Lessons Learned

**`ansible_user` is immediately available as an ownership variable.** Many engineers default to defining separate `host_vars` or group variables to manage per-host ownership. Using `ansible_user` directly eliminates that overhead when the SSH user and the intended file owner are the same entity, which is the common case in single-user host configurations.

**Check mode is not optional.** Running `--check` before `--apply` on file system tasks in shared environments prevents race conditions and unintended overwrites. The check mode output showed `changed=1` accurately for all hosts, confirming the task would execute correctly before touching production paths.

**Octal permissions must be quoted in YAML.** The `mode` parameter in Ansible's `file` module must be passed as a string (`"0655"`) rather than an unquoted integer. YAML may interpret unquoted leading-zero values incorrectly, stripping the leading zero and resulting in wrong permissions (`655` evaluated as decimal `429`, not octal `0655`). Quoting is mandatory.

**Task response ordering varies between runs.** In the gather-facts phase of the live run, `stapp02` responded before `stapp01`, while in the check run `stapp01` responded first. This reflects Ansible's default forked execution strategy and is expected behavior. It has no impact on correctness.

**SSH verification is a required step, not a bonus.** Ansible's `changed` status reflects module-reported state, not independently verified disk state. Post-execution SSH inspection provides ground-truth confirmation that the file, its permissions, and its ownership are exactly as specified, independent of Ansible's internal state tracking.

---

## Project Structure

```
~/playbook/
├── ansible.cfg         # Project-scoped Ansible configuration
├── inventory           # Static inventory with host credentials and become parameters
└── playbook.yml        # Single-play playbook for file creation across app_servers
```




<img width="502" height="332" alt="image" src="https://github.com/user-attachments/assets/50468f12-e82e-4541-9c78-7017f243c6e9" />
<img width="506" height="322" alt="image" src="https://github.com/user-attachments/assets/a5c8d5a2-a873-4f32-99c7-4571bbaa9157" />
<img width="509" height="306" alt="image" src="https://github.com/user-attachments/assets/304991d6-2ac3-43ec-8e6c-5145f56f3b8f" />
<img width="508" height="334" alt="image" src="https://github.com/user-attachments/assets/d1cbed11-56a3-4fff-8aa8-213bb9e4fed3" />
<img width="522" height="370" alt="image" src="https://github.com/user-attachments/assets/c2726bea-e048-471c-ac03-f57374d3bf5b" />
<img width="518" height="389" alt="image" src="https://github.com/user-attachments/assets/6d533bb2-9f16-4e95-b681-1d4bb02ddbfb" />
<img width="518" height="419" alt="image" src="https://github.com/user-attachments/assets/2b7bb2c3-8f15-4625-84b7-ae6cb40531dd" />
<img width="516" height="355" alt="image" src="https://github.com/user-attachments/assets/64e64697-da43-4915-a7bc-e25f45dedc9c" />
<img width="517" height="419" alt="image" src="https://github.com/user-attachments/assets/b715b291-6d7c-4e32-a382-9fe20e8b9d8d" />
<img width="519" height="373" alt="image" src="https://github.com/user-attachments/assets/325328ee-931b-487c-8861-a78176d1841c" />
<img width="516" height="419" alt="image" src="https://github.com/user-attachments/assets/b2b64fdc-9ca0-422d-abc1-75dcd5fc35ac" />
<img width="518" height="397" alt="image" src="https://github.com/user-attachments/assets/a5d05af4-f4ff-44c6-8790-1510f4a03c7d" />
<img width="518" height="337" alt="image" src="https://github.com/user-attachments/assets/a04cc27b-05e1-411a-a37d-95f5a98c1e40" />
<img width="521" height="373" alt="image" src="https://github.com/user-attachments/assets/248652ab-4b46-4711-ba26-93f436ffed69" />
<img width="517" height="410" alt="image" src="https://github.com/user-attachments/assets/ab582ccd-b407-4cbd-a068-b84a19f43265" />
