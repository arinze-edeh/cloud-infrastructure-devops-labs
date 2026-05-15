# Ansible: Automated Apache HTTP Server Deployment Across Multi-Node Infrastructure

---

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Phase 1: Inspect Existing Directory, Configuration, and Inventory](#phase-1-inspect-existing-directory-configuration-and-inventory)
  * [Phase 2: Author the Ansible Playbook](#phase-2-author-the-ansible-playbook)
  * [Phase 3: Update Inventory with Privilege Escalation Credentials](#phase-3-update-inventory-with-privilege-escalation-credentials)
  * [Phase 4: Verify Playbook and Inventory Contents](#phase-4-verify-playbook-and-inventory-contents)
  * [Phase 5: Syntax Validation](#phase-5-syntax-validation)
  * [Phase 6: Playbook Execution](#phase-6-playbook-execution)
  * [Phase 7: Post-Deployment Verification](#phase-7-post-deployment-verification)
* [Playbook Task Breakdown](#playbook-task-breakdown)
* [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)

---

## Overview

This project automates the provisioning and configuration of an Apache HTTP Server (`httpd`) across three application servers in the Stratos DC environment using Ansible. The automation is executed from a centralized jump host and targets all app nodes simultaneously via a static inventory.

The playbook handles the complete server setup lifecycle: package installation, service management, file creation, content injection via the `blockinfile` module, and enforcement of correct file ownership and permissions.

**Tooling:** Ansible, YAML, SSH, `blockinfile`, `yum`, `service`, `file` modules

**Target Nodes:** `stapp01`, `stapp02`, `stapp03`

**Execution Host:** `jump-host`

---

## Architecture

```
jump-host (Ansible Control Node)
    |
    |-- SSH --> stapp01 (tony@stapp01)   | httpd installed, content deployed
    |-- SSH --> stapp02 (steve@stapp02)  | httpd installed, content deployed
    |-- SSH --> stapp03 (banner@stapp03) | httpd installed, content deployed
```

All three app servers are targeted simultaneously using `hosts: all`. Privilege escalation via `sudo` is used on each node to perform privileged operations including package installation and service management.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Ansible installed on jump-host | Any version supporting `yum`, `service`, `file`, `blockinfile` modules |
| SSH connectivity to all app servers | Verified via inventory credentials |
| `sudo` access for remote users | Required for package and service operations |
| Inventory file pre-existing | Located at `/home/thor/ansible/inventory` |
| `ansible.cfg` pre-existing | Located at `/home/thor/ansible/ansible.cfg` |

---

## Repository Structure

```
/home/thor/ansible/
├── ansible.cfg        # Ansible control configuration (host_key_checking disabled)
├── inventory          # Static inventory with SSH and sudo credentials
└── playbook.yml       # Main automation playbook
```

---

## Implementation Guide

### Phase 1: Inspect Existing Directory, Configuration, and Inventory

Before authoring any files, the existing environment artifacts were inspected to understand the working directory structure, Ansible configuration, and inventory in place.

**Inspect the working directory:**

```bash
ls -la /home/thor/ansible/
```

**Output:**

```
total 20
drwxr-xr-x 2 thor thor 4096 May 15 04:59 .
drwx------ 1 thor thor 4096 May 15 04:59 ..
-rw-r--r-- 1 thor thor   36 May 15 04:59 ansible.cfg
-rw-r--r-- 1 thor thor  219 May 15 04:59 inventory
```

**Inspect `ansible.cfg`:**

```bash
cat /home/thor/ansible/ansible.cfg
```

**Output:**

```ini
[defaults]
host_key_checking = False
```

`host_key_checking = False` disables SSH host key verification. This is appropriate in ephemeral environments where host keys are not pre-registered in `known_hosts`. It should not be used in production without compensating controls.

**Inspect the existing inventory:**

```bash
cat /home/thor/ansible/inventory
```

**Output:**

```ini
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner
```

The inventory was functional for SSH connectivity but lacked `ansible_become_pass` entries required for `sudo` privilege escalation during playbook execution. This was noted for correction after the playbook was authored.

*Screenshot: Terminal output showing `ls -la`, `ansible.cfg` content, and initial inventory content*

<img width="509" height="308" alt="image" src="https://github.com/user-attachments/assets/08066865-2b6c-4884-9370-500f092aa049" />

---

### Phase 2: Author the Ansible Playbook

With the environment understood, the playbook was authored at `/home/thor/ansible/playbook.yml`. It targets all hosts defined in the inventory, escalates privileges using `sudo`, and executes five ordered tasks.

**Note:** The `blockinfile` module was used without a custom `marker` parameter, relying on the default Ansible-managed block markers (`# BEGIN ANSIBLE MANAGED BLOCK` / `# END ANSIBLE MANAGED BLOCK`), as required by the task specification.

```bash
cat > /home/thor/ansible/playbook.yml << 'EOF'
---
- name: Install and configure httpd on all app servers
  hosts: all
  become: yes
  become_method: sudo

  tasks:

    - name: Install httpd package
      yum:
        name: httpd
        state: present

    - name: Ensure httpd service is started and enabled
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Ensure /var/www/html/index.html exists
      file:
        path: /var/www/html/index.html
        state: touch
        owner: apache
        group: apache
        mode: '0655'

    - name: Add content to /var/www/html/index.html using blockinfile
      blockinfile:
        path: /var/www/html/index.html
        block: |
          Welcome to XfusionCorp!
          This is Nautilus sample file, created using Ansible!
          Please do not modify this file manually!

    - name: Set owner, group and permissions on index.html
      file:
        path: /var/www/html/index.html
        owner: apache
        group: apache
        mode: '0655'
EOF
```

*Screenshot: Terminal after heredoc write confirming playbook.yml created with no errors*

<img width="517" height="435" alt="image" src="https://github.com/user-attachments/assets/2a61a559-ab68-4f13-9db3-1fe55ceffb62" />

---

### Phase 3: Update Inventory with Privilege Escalation Credentials

With the playbook authored and `become: yes` declared, the inventory was updated to include `ansible_become_pass` for each host. Without this, Ansible cannot authenticate the `sudo` escalation on nodes that require a password for privilege elevation, which would cause failures on every privileged task.

The inventory was rewritten in full using a heredoc to ensure clean output with no residual lines from the original file:

```bash
cat > /home/thor/ansible/inventory << 'EOF'
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony ansible_become_pass=Ir0nM@n
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve ansible_become_pass=Am3ric@
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner ansible_become_pass=BigGr33n
EOF
```

*Screenshot: Terminal confirming inventory heredoc write completed successfully*

<img width="518" height="313" alt="image" src="https://github.com/user-attachments/assets/baa0db9d-8ab9-4ea6-ae98-64e31d387fdd" />

---

### Phase 4: Verify Playbook and Inventory Contents

Both files were read back in full to confirm their contents matched the intended configuration before proceeding to validation and execution.

**Verify the playbook:**

```bash
cat /home/thor/ansible/playbook.yml
```

**Output:**

```yaml
---
- name: Install and configure httpd on all app servers
  hosts: all
  become: yes
  become_method: sudo

  tasks:

    - name: Install httpd package
      yum:
        name: httpd
        state: present

    - name: Ensure httpd service is started and enabled
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Ensure /var/www/html/index.html exists
      file:
        path: /var/www/html/index.html
        state: touch
        owner: apache
        group: apache
        mode: '0655'

    - name: Add content to /var/www/html/index.html using blockinfile
      blockinfile:
        path: /var/www/html/index.html
        block: |
          Welcome to XfusionCorp!
          This is Nautilus sample file, created using Ansible!
          Please do not modify this file manually!

    - name: Set owner, group and permissions on index.html
      file:
        path: /var/www/html/index.html
        owner: apache
        group: apache
        mode: '0655'
```

**Verify the updated inventory:**

```bash
cat /home/thor/ansible/inventory
```

**Output:**

```ini
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony ansible_become_pass=Ir0nM@n
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve ansible_become_pass=Am3ric@
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner ansible_become_pass=BigGr33n
```

*Screenshots: Both `cat` outputs confirming playbook structure and updated inventory with `ansible_become_pass` on all three hosts*

<img width="530" height="437" alt="image" src="https://github.com/user-attachments/assets/4ab76405-56b1-4a10-962e-d2b81baa6a72" />
<img width="517" height="273" alt="image" src="https://github.com/user-attachments/assets/a8fe8210-69fa-4312-b304-667deba285ac" />

---

### Phase 5: Syntax Validation

Before executing against live targets, the playbook was validated using Ansible's built-in syntax checker to catch YAML and structural errors without making any changes to the target nodes.

```bash
cd /home/thor/ansible && ansible-playbook -i inventory playbook.yml --syntax-check
```

**Output:**

```
playbook: playbook.yml
```

A clean return of only the playbook path with no errors confirms the playbook is syntactically valid and ready for execution.

*Screenshot: Syntax check output showing clean validation with no errors*

<img width="515" height="306" alt="image" src="https://github.com/user-attachments/assets/a7468308-baf5-42ab-b0f3-241ba30b80cd" />

---

### Phase 6: Playbook Execution

The playbook was executed from the `/home/thor/ansible` directory using the standard invocation format required by the task specification.

```bash
cd /home/thor/ansible && ansible-playbook -i inventory playbook.yml
```

**Output:**

```
PLAY [Install and configure httpd on all app servers] **************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [stapp03]
ok: [stapp01]
ok: [stapp02]

TASK [Install httpd package] ***************************************************************************************************
changed: [stapp02]
changed: [stapp03]
changed: [stapp01]

TASK [Ensure httpd service is started and enabled] *****************************************************************************
changed: [stapp02]
changed: [stapp03]
changed: [stapp01]

TASK [Ensure /var/www/html/index.html exists] **********************************************************************************
changed: [stapp03]
changed: [stapp02]
changed: [stapp01]

TASK [Add content to /var/www/html/index.html using blockinfile] ***************************************************************
changed: [stapp03]
changed: [stapp02]
changed: [stapp01]

TASK [Set owner, group and permissions on index.html] **************************************************************************
ok: [stapp03]
ok: [stapp02]
ok: [stapp01]

PLAY RECAP *********************************************************************************************************************
stapp01                    : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp02                    : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
stapp03                    : ok=6    changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

All three nodes completed with `ok=6`, `changed=4`, `unreachable=0`, and `failed=0`. The `Set owner, group and permissions on index.html` task returned `ok` rather than `changed` on all nodes, confirming the ownership and permissions applied during the `file: state: touch` task were already correct and no further modification was needed.

*Screenshot: Full play recap showing all three app servers with zero failures and zero unreachable*

<img width="519" height="434" alt="image" src="https://github.com/user-attachments/assets/515c73e4-e81e-4742-bc1f-159ff2b38432" />

---

### Phase 7: Post-Deployment Verification

Following execution, each app server was verified individually using `sshpass` to authenticate and remotely inspect file content, file metadata, and service state.

**Verify stapp01:**

```bash
sshpass -p 'Ir0nM@n' ssh tony@stapp01 "cat /var/www/html/index.html"
```

**Output:**

```
# BEGIN ANSIBLE MANAGED BLOCK
Welcome to XfusionCorp!
This is Nautilus sample file, created using Ansible!
Please do not modify this file manually!
# END ANSIBLE MANAGED BLOCK
```

```bash
sshpass -p 'Ir0nM@n' ssh tony@stapp01 "ls -la /var/www/html/index.html"
```

**Output:**

```
-rw-r-xr-x 1 apache apache 176 May 15 05:13 /var/www/html/index.html
```

```bash
sshpass -p 'Ir0nM@n' ssh tony@stapp01 "sudo systemctl is-active httpd && sudo systemctl is-enabled httpd"
```

**Output:**

```
active
enabled
```

*Screenshot: stapp01 verification showing correct file content, `-rw-r-xr-x` permissions, apache ownership, and httpd active/enabled*

---

**Verify stapp02:**

```bash
sshpass -p 'Am3ric@' ssh steve@stapp02 "cat /var/www/html/index.html"
```

**Output:**

```
# BEGIN ANSIBLE MANAGED BLOCK
Welcome to XfusionCorp!
This is Nautilus sample file, created using Ansible!
Please do not modify this file manually!
# END ANSIBLE MANAGED BLOCK
```

```bash
sshpass -p 'Am3ric@' ssh steve@stapp02 "ls -la /var/www/html/index.html"
```

**Output:**

```
-rw-r-xr-x 1 apache apache 176 May 15 05:13 /var/www/html/index.html
```

```bash
sshpass -p 'Am3ric@' ssh steve@stapp02 "sudo systemctl is-active httpd && sudo systemctl is-enabled httpd"
```

**Output:**

```
active
enabled
```

*Screenshot: stapp02 verification showing correct file content, permissions, apache ownership, and httpd active/enabled*

---

**Verify stapp03:**

```bash
sshpass -p 'BigGr33n' ssh banner@stapp03 "cat /var/www/html/index.html"
sshpass -p 'BigGr33n' ssh banner@stapp03 "ls -la /var/www/html/index.html"
sshpass -p 'BigGr33n' ssh banner@stapp03 "sudo systemctl is-active httpd && sudo systemctl is-enabled httpd"
```

**Output:**

```
# BEGIN ANSIBLE MANAGED BLOCK
Welcome to XfusionCorp!
This is Nautilus sample file, created using Ansible!
Please do not modify this file manually!
# END ANSIBLE MANAGED BLOCK
-rw-r-xr-x 1 apache apache 176 May 15 05:13 /var/www/html/index.html
active
enabled
```

*Screenshot: stapp03 verification showing correct file content, permissions, apache ownership, and httpd active/enabled*

---

## Playbook Task Breakdown

| Task | Module | Purpose | Result |
|---|---|---|---|
| Install httpd package | `yum` | Installs the Apache HTTP Server package | `changed` on all nodes |
| Ensure httpd is started and enabled | `service` | Starts the httpd service and enables it at boot | `changed` on all nodes |
| Ensure index.html exists | `file` (state: touch) | Creates the file if absent; sets owner, group, and mode | `changed` on all nodes |
| Add content via blockinfile | `blockinfile` | Injects the three-line block using default managed block markers | `changed` on all nodes |
| Set owner, group and permissions | `file` | Enforces final ownership (`apache:apache`) and mode (`0655`) | `ok` on all nodes (idempotent) |

**Permission notation `0655` explained:**

| Segment | Octal | Symbolic | Applies To |
|---|---|---|---|
| Owner | 6 | `rw-` | `apache` user |
| Group | 5 | `r-x` | `apache` group |
| Other | 5 | `r-x` | World |

The rendered permission string on disk is `-rw-r-xr-x`, consistent with what was confirmed during post-deployment verification across all three nodes.

---

## Errors Encountered and Resolutions

### Error 1: Missing `ansible_become_pass` in Original Inventory

**Context:** The original inventory did not include `ansible_become_pass` entries. Without this, Ansible cannot authenticate the `sudo` escalation on nodes that require a password for privilege elevation. This was identified during playbook authoring when `become: yes` and `become_method: sudo` were declared at the play level, making it clear that escalation credentials would be required at runtime.

**Resolution:** After the playbook was written, the inventory was rewritten in full to include `ansible_become_pass` for each host, matching the respective SSH password:

```ini
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony ansible_become_pass=Ir0nM@n
```

This ensured all privileged tasks executed without escalation failures across all three nodes.

---

### Error 2: SSH Authentication Failure During Manual Post-Deployment Verification

**Context:** After playbook execution, a manual SSH verification attempt was made using plain `ssh` without supplying credentials, which caused repeated authentication failures on all three nodes.

**Commands attempted:**

```bash
ssh tony@stapp01 "cat /var/www/html/index.html"
ssh steve@stapp02 "cat /var/www/html/index.html"
ssh banner@stapp03 "cat /var/www/html/index.html"
```

**Observed behavior:**

```
tony@stapp01's password:
Permission denied, please try again.
tony@stapp01's password:
Permission denied, please try again.
tony@stapp01's password:
Connection closed by 10.244.189.219 port 22
```

Similar failures were observed for `stapp02` and `stapp03`.

**Root cause:** The jump host does not have pre-configured SSH key-based authentication for the app server users. The `ssh` command prompts for a password interactively, and the environment does not support that input method in this context.

**Resolution:** Replaced bare `ssh` calls with `sshpass`, which injects the password non-interactively at the command line:

```bash
sshpass -p 'Ir0nM@n' ssh tony@stapp01 "cat /var/www/html/index.html"
```

This resolved the authentication failures and allowed all three nodes to be verified successfully.

---

## Best Practices Applied

* **Idempotent task design:** Each task was written to be safely re-executable. The `blockinfile` module manages its own content block markers to avoid duplicate content on repeated runs. The final `file` task re-enforces ownership and permissions idempotently, confirmed by `ok` status on first run.

* **Privilege escalation scoped to play level:** `become: yes` and `become_method: sudo` were declared at the play level rather than per-task, ensuring all tasks run with the required privileges without redundant per-task declarations.

* **`ansible_become_pass` supplied via inventory:** Privilege escalation credentials were supplied through inventory variables rather than command-line flags, ensuring the playbook can be invoked with the standard `ansible-playbook -i inventory playbook.yml` format without any additional arguments.

* **Playbook authored before inventory finalized:** Writing the playbook first surfaced exactly which inventory variables were required by the automation. Declaring `become: yes` made it immediately clear that `ansible_become_pass` was missing from the existing inventory, allowing the correction to be made as a deliberate step before any execution attempt.

* **Pre-execution content verification:** Both the playbook and the updated inventory were read back in full before syntax checking and execution, confirming the written content matched the intended configuration and eliminating the risk of executing against a malformed file.

* **Syntax validation before execution:** `--syntax-check` was run before live execution to catch YAML or structural issues without making any changes to target nodes.

* **Separation of file creation and content injection:** The `file: state: touch` task and the `blockinfile` task are kept as distinct steps, maintaining clear separation of concerns: existence and ownership are managed by `file`, while content is managed by `blockinfile`.

* **Default `blockinfile` markers retained:** No custom `marker` was defined in the `blockinfile` task, intentionally preserving the default `# BEGIN ANSIBLE MANAGED BLOCK` / `# END ANSIBLE MANAGED BLOCK` markers as specified by the task requirements.

* **Post-deployment verification per node:** Each node was individually verified for file content, file metadata, and service state using `sshpass` to ensure no silent failures were masked by the play recap summary.

---

## Lessons Learned

* **Always include `ansible_become_pass` when remote users require sudo authentication.** The absence of this variable is a silent failure risk. Ansible will hang or fail at the first privileged task without it, producing non-obvious error output depending on the target node's sudo configuration.

* **Author the playbook before finalizing the inventory.** Writing the playbook first surfaces which inventory variables are actually required. In this case, declaring `become: yes` immediately revealed that `ansible_become_pass` was missing from the original inventory, allowing the gap to be closed before execution rather than discovered through a failed run.

* **`blockinfile` is idempotent by design.** It tracks its managed block using the default `# BEGIN ANSIBLE MANAGED BLOCK` and `# END ANSIBLE MANAGED BLOCK` markers. Re-running the playbook will not duplicate the content block. Custom markers should only be introduced when managing multiple distinct blocks within the same file.

* **`file: state: touch` sets timestamps as well as creates files.** In production workflows, prefer `state: touch` only when you explicitly need creation-without-content behavior. If the file needs content from the start, use `copy` or `template` directly to avoid an extra task.

* **The `ok` status on the final permissions task indicates idempotency, not failure.** When `file` is used to enforce ownership and mode after an earlier task has already applied those attributes, Ansible correctly reports `ok` rather than `changed`. This is expected and healthy behavior confirming the desired state was already reached.

* **`sshpass` is essential for non-interactive password injection in scripted verification contexts.** In environments without SSH key-based authentication, `sshpass` is the correct tool for scripted verification against password-authenticated hosts. It should not be used as a long-term substitute for key-based authentication in production.

* **Ansible handles parallel execution across inventory hosts by default.** All three nodes received each task simultaneously based on the default forks configuration, which significantly reduces total deployment time compared to sequential node-by-node execution.



















<img width="517" height="137" alt="image" src="https://github.com/user-attachments/assets/e5a384ed-c7e1-4dc2-9c02-973635c7c03a" />
<img width="545" height="228" alt="image" src="https://github.com/user-attachments/assets/cc3f1250-279a-4e14-baa3-e72f5491a7f2" />
<img width="547" height="331" alt="image" src="https://github.com/user-attachments/assets/de05e44a-7d13-4058-bbb8-cebfe3f74da3" />
