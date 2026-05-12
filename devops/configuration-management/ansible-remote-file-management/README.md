# Ansible File Creation and Permission Management Across Remote Hosts

> Automating file provisioning, permission enforcement, and ownership assignment on distributed application servers using Ansible's `file` module in a multi-host Stratos DC environment.

---

## Table of Contents

- [Overview](#overview)
- [Architecture and Environment](#architecture-and-environment)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Confirm the Control Node Identity](#step-1-confirm-the-control-node-identity)
  - [Step 2: Create and Navigate to the Working Directory](#step-2-create-and-navigate-to-the-working-directory)
  - [Step 3: Create the Ansible Inventory](#step-3-create-the-ansible-inventory)
  - [Step 4: Configure Ansible Defaults](#step-4-configure-ansible-defaults)
  - [Step 5: Validate Connectivity with Ping Module](#step-5-validate-connectivity-with-ping-module)
  - [Step 6: Author the Playbook](#step-6-author-the-playbook)
  - [Step 7: Syntax Check](#step-7-syntax-check)
  - [Step 8: Dry Run with Check Mode](#step-8-dry-run-with-check-mode)
  - [Step 9: Execute the Playbook](#step-9-execute-the-playbook)
  - [Step 10: Verify File Creation and Permissions on Each Host](#step-10-verify-file-creation-and-permissions-on-each-host)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Project Structure](#project-structure)

---

## Overview

This project demonstrates a structured Ansible approach to creating a managed file (`/usr/src/appdata.txt`) across all application servers in the Stratos Datacenter. The task enforces consistent file permissions (`0655`) and assigns the correct user and group ownership per host using dynamic variable resolution at runtime.

The implementation follows a gated execution workflow: control node verification, working directory setup, inventory authoring, connectivity verification, syntax check, dry run, and live execution with post-deployment SSH verification.

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
| Working Directory | `/home/thor/playbook` |
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

---

## Implementation Guide

### Step 1: Confirm the Control Node Identity

Before any environment setup, confirm the hostname of the control node to verify you are operating on the correct machine.

```bash
hostname
```

**Expected output:**

```
jump-host
```

*Screenshot: Terminal output of `hostname` returning `jump-host`, confirming the correct control node is active.*

<img width="502" height="332" alt="image" src="https://github.com/user-attachments/assets/50468f12-e82e-4541-9c78-7017f243c6e9" />

---

### Step 2: Create and Navigate to the Working Directory

Create the project working directory using the `-p` flag to ensure no error is raised if the directory already exists, then navigate into it and confirm the current path.

```bash
mkdir -p ~/playbook
cd ~/playbook
pwd
```

**Expected output:**

```
/home/thor/playbook
```

All subsequent files (`inventory`, `ansible.cfg`, `playbook.yml`) are created inside this directory. Scoping all project files to a single directory keeps configuration isolated and makes the project fully portable.

*Screenshot: Terminal output showing `mkdir -p ~/playbook`, `cd ~/playbook`, and `pwd` returning `/home/thor/playbook`.*

<img width="509" height="306" alt="image" src="https://github.com/user-attachments/assets/304991d6-2ac3-43ec-8e6c-5145f56f3b8f" />

---

### Step 3: Create the Ansible Inventory

Write the inventory file defining all three application servers under the group `[app_servers]`. Each entry specifies the SSH user, SSH password, privilege escalation flag, and the escalation password for `sudo` operations.

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

<img width="522" height="370" alt="image" src="https://github.com/user-attachments/assets/c2726bea-e048-471c-ac03-f57374d3bf5b" />

---

### Step 4: Configure Ansible Defaults

Create an `ansible.cfg` file in the working directory to disable SSH host key checking. This prevents connection failures in environments where remote host fingerprints are not pre-registered in the control node's `~/.ssh/known_hosts`.

```bash
cat > ansible.cfg << 'EOF'
[defaults]
host_key_checking = False
EOF
```

*Screenshot: Terminal output confirming creation of `ansible.cfg` with `host_key_checking = False`.*

<img width="518" height="389" alt="image" src="https://github.com/user-attachments/assets/6d533bb2-9f16-4e95-b681-1d4bb02ddbfb" />

---

### Step 5: Validate Connectivity with Ping Module

Before authoring any tasks, validate that Ansible can reach all managed hosts using the built-in `ping` module. This confirms inventory correctness, credential validity, and network reachability in a single command.

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

*Screenshot: Full terminal output of `ansible all -i inventory -m ping` showing `SUCCESS` for `stapp01`, `stapp02`, and `stapp03`.*

<img width="518" height="419" alt="image" src="https://github.com/user-attachments/assets/2b7bb2c3-8f15-4625-84b7-ae6cb40531dd" />

---

### Step 6: Author the Playbook

Write the playbook that targets the `app_servers` group and creates `/usr/src/appdata.txt` using the `file` module. The `ansible_user` variable is used dynamically for both `owner` and `group`, ensuring each host receives the correct ownership mapping without hardcoding per-host values.

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

**Design decision:** Using `{{ ansible_user }}` as both `owner` and `group` allows a single task definition to correctly resolve to `tony`, `steve`, and `banner` on their respective hosts without conditional branching or separate `host_vars` files. The variable is already defined per host in the inventory, so it resolves at runtime with zero additional configuration.

*Screenshot: Terminal output of `cat playbook.yml` confirming correct indentation, module parameters, and Jinja2 variable references.*

<img width="517" height="419" alt="image" src="https://github.com/user-attachments/assets/b715b291-6d7c-4e32-a382-9fe20e8b9d8d" />

---

### Step 7: Syntax Check

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

<img width="519" height="373" alt="image" src="https://github.com/user-attachments/assets/325328ee-931b-487c-8861-a78176d1841c" />

---

### Step 8: Dry Run with Check Mode

Execute the playbook in check mode to simulate what changes would occur without applying them to the remote hosts. This is the final validation gate before a live run and confirms task logic against the real inventory state.

```bash
ansible-playbook -i inventory playbook.yml --check
```

**Expected output:**

```
PLAY [Create appdata.txt on all app servers] *********************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [stapp01]
ok: [stapp02]
ok: [stapp03]

TASK [Create blank file /usr/src/appdata.txt] ********************************************************
changed: [stapp01]
changed: [stapp02]
changed: [stapp03]

PLAY RECAP *******************************************************************************************
stapp01                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp02                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp03                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

The check mode output confirms `changed=1` on all three hosts, indicating the file does not yet exist and will be created upon live execution.

*Screenshot: Full terminal output of `ansible-playbook --check` showing `changed: [stapp01]`, `changed: [stapp02]`, and `changed: [stapp03]` with a clean PLAY RECAP and zero failures.*

---

### Step 9: Execute the Playbook

With connectivity, syntax, and dry-run validation all passing, execute the playbook against the live inventory.

```bash
ansible-playbook -i inventory playbook.yml
```

**Expected output:**

```
PLAY [Create appdata.txt on all app servers] *********************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [stapp02]
ok: [stapp01]
ok: [stapp03]

TASK [Create blank file /usr/src/appdata.txt] ********************************************************
changed: [stapp02]
changed: [stapp01]
changed: [stapp03]

PLAY RECAP *******************************************************************************************
stapp01                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp02                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp03                    : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

All three hosts reported `changed=1` with zero failures, zero unreachable, and zero skipped tasks. The playbook completed successfully across the entire `app_servers` group.

*Screenshot: Full terminal output of the live `ansible-playbook -i inventory playbook.yml` execution showing `changed` status on all three hosts and a clean PLAY RECAP.*

---

### Step 10: Verify File Creation and Permissions on Each Host

After playbook execution, SSH into each application server individually and inspect the created file using `ls -l` to independently confirm path, permissions, owner, and group on each host.

**Verify on stapp01:**

```bash
ssh tony@stapp01 "ls -l /usr/src/appdata.txt"
```

**Expected output:**

```
-rw-r-xr-x 1 tony tony 0 May 12 05:44 /usr/src/appdata.txt
```

*Screenshot: Terminal output of the SSH command to `stapp01` showing `-rw-r-xr-x 1 tony tony 0` confirming correct permissions and ownership.*

---

**Verify on stapp02:**

```bash
ssh steve@stapp02 "ls -l /usr/src/appdata.txt"
```

**Expected output:**

```
-rw-r-xr-x 1 steve steve 0 May 12 05:44 /usr/src/appdata.txt
```

*Screenshot: Terminal output of the SSH command to `stapp02` showing `-rw-r-xr-x 1 steve steve 0` confirming correct permissions and ownership.*

---

**Verify on stapp03:**

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

* File path: `/usr/src/appdata.txt` present on all hosts
* Permissions: `-rw-r-xr-x` (octal `0655`)
* Owner and group resolved correctly per host: `tony`, `steve`, `banner`
* File size: `0` bytes (blank file, as required)

---

## Best Practices Applied

**Control node verification before any action:** Running `hostname` as the first command confirms execution context before any environment changes are made. In shared jump-host environments where multiple engineers operate simultaneously, this prevents accidental operations on the wrong machine.

**Working directory isolation with `mkdir -p`:** Using `mkdir -p` to create the project directory is non-destructive: it succeeds silently whether the directory already exists or not. This makes the setup step safely repeatable and avoids script failures in automation pipelines where the directory may have been pre-created.

**Gated execution workflow:** The implementation followed a structured pre-execution validation pipeline across four stages: inventory content verification via `cat`, connectivity confirmation via the `ping` module, syntax validation via `--syntax-check`, and change simulation via `--check`. Each gate must pass before advancing to the next. This eliminates ambiguity and prevents unintended remote state changes in shared environments.

**Dynamic variable resolution for per-host ownership:** Rather than defining separate tasks, using `when` conditionals, or maintaining `host_vars` files, `{{ ansible_user }}` was used to resolve the correct `owner` and `group` at runtime. Because `ansible_user` is already defined per host in the inventory, this produces the correct ownership on each target with a single task definition and no additional configuration overhead.

**Project-scoped `ansible.cfg`:** Placing `ansible.cfg` inside `~/playbook` scopes Ansible configuration to the project and prevents global configuration pollution. In environments with multiple projects running under different Ansible profiles, this isolation is critical for reproducibility and prevents one project's settings from silently affecting another.

**Post-execution SSH verification as ground truth:** Ansible's `changed` status reflects the module's internal assessment of state, not independently verified disk reality. Direct SSH-based `ls -l` inspection on each host provides authoritative confirmation that the file exists, carries the correct permissions, and is owned by the expected user and group, independently of Ansible's reporting layer.

**`become: true` applied at play scope:** Privilege escalation is declared at the play level rather than per task, ensuring all tasks in the play run under a consistent elevated context. Applying `become` at the task level creates risk of tasks silently executing without escalation if the declaration is accidentally omitted during future playbook expansion.

---

## Lessons Learned

**Always confirm your execution context first.** In multi-user jump-host environments, it is easy to assume you are on the right machine. Running `hostname` as the first command is a five-second check that eliminates an entire class of environment-related errors before any files are created or commands are run against remote hosts.

**`mkdir -p` is the correct primitive for idempotent directory setup.** Using `mkdir` without `-p` will fail if the directory already exists, making scripts and runbooks non-repeatable. The `-p` flag ensures the command is safe to run at any point in the workflow without side effects.

**`ansible_user` resolves ownership without additional variables.** A common pattern is to define separate `host_vars` entries or group-level variables to manage per-host file ownership. When the SSH user and intended file owner are the same entity, which is the standard case in single-user host configurations, `{{ ansible_user }}` provides the same result with zero additional inventory management overhead.

**Check mode output is a pre-execution contract.** The `--check` run returned `changed=1` on all three hosts before live execution, confirming the file did not yet exist and the task would apply changes. This is not a formality but a verification step that establishes expected behavior before touching remote paths. A mismatch between check mode output and live output indicates an environmental inconsistency that must be investigated before proceeding.

**Octal permissions require string quoting in YAML.** The `mode` parameter must be passed as a quoted string (`"0655"`) rather than a bare integer. YAML parsers may silently drop the leading zero from unquoted octal literals, interpreting `0655` as decimal `429` and applying incorrect permissions. Quoting is mandatory for all octal mode values in Ansible playbooks.

**Parallel execution produces non-deterministic task response ordering.** During the check run, facts were gathered in the order `stapp01`, `stapp02`, `stapp03`. During the live run, `stapp02` responded before `stapp01`. This is the expected behavior of Ansible's default forked execution model and has no impact on correctness or idempotency. Engineers reviewing PLAY output should not interpret ordering differences between runs as an indicator of task failure.

**SSH verification closes the observability gap between Ansible and disk state.** Ansible reports task outcomes based on module logic, not on independent file system inspection. Post-execution SSH-based verification confirms ground truth: the file was physically written to the correct path, with the correct permissions and ownership, on each target host. This step is a required part of a complete deployment verification workflow, not an optional sanity check.

---

## Project Structure

```
~/playbook/
├── ansible.cfg         # Project-scoped Ansible configuration (host_key_checking disabled)
├── inventory           # Static inventory with per-host credentials and become parameters
└── playbook.yml        # Single-play playbook targeting app_servers for file provisioning
```





<img width="516" height="355" alt="image" src="https://github.com/user-attachments/assets/64e64697-da43-4915-a7bc-e25f45dedc9c" />

<img width="516" height="419" alt="image" src="https://github.com/user-attachments/assets/b2b64fdc-9ca0-422d-abc1-75dcd5fc35ac" />
<img width="518" height="397" alt="image" src="https://github.com/user-attachments/assets/a5d05af4-f4ff-44c6-8790-1510f4a03c7d" />
<img width="518" height="337" alt="image" src="https://github.com/user-attachments/assets/a04cc27b-05e1-411a-a37d-95f5a98c1e40" />
<img width="521" height="373" alt="image" src="https://github.com/user-attachments/assets/248652ab-4b46-4711-ba26-93f436ffed69" />
<img width="517" height="410" alt="image" src="https://github.com/user-attachments/assets/ab582ccd-b407-4cbd-a068-b84a19f43265" />
