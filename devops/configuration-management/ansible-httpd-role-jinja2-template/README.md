# Ansible Role-Based HTTPD Deployment with Jinja2 Templating

## Table of Contents

* [Overview](#overview)
* [Architecture and Design Intent](#architecture-and-design-intent)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Inspect the Home Directory on the Jump Host](#step-1-inspect-the-home-directory-on-the-jump-host)
  * [Step 2: Review the Ansible Inventory](#step-2-review-the-ansible-inventory)
  * [Step 3: Inspect the Existing Playbook](#step-3-inspect-the-existing-playbook)
  * [Step 4: Audit the Full Ansible Directory Tree](#step-4-audit-the-full-ansible-directory-tree)
  * [Step 5: Review the Existing Role Task File](#step-5-review-the-existing-role-task-file)
  * [Step 6: Overwrite the Playbook to Target App Server 3](#step-6-overwrite-the-playbook-to-target-app-server-3)
  * [Step 7: Verify the Updated Playbook](#step-7-verify-the-updated-playbook)
  * [Step 8: Create the Templates Directory](#step-8-create-the-templates-directory)
  * [Step 9: Create the Jinja2 Template File](#step-9-create-the-jinja2-template-file)
  * [Step 10: Verify the Template File Content](#step-10-verify-the-template-file-content)
  * [Step 11: Append the Template Deployment Task to the Role](#step-11-append-the-template-deployment-task-to-the-role)
  * [Step 12: Verify the Final Role Task File](#step-12-verify-the-final-role-task-file)
  * [Step 13: Run Playbook Syntax Check](#step-13-run-playbook-syntax-check)
  * [Step 14: Execute the Playbook](#step-14-execute-the-playbook)
  * [Step 15: SSH into App Server 3 and Verify the Deployed File](#step-15-ssh-into-app-server-3-and-verify-the-deployed-file)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Technologies Used](#technologies-used)

---

## Overview

This project extends an existing Ansible role (`httpd`) to include dynamic web content deployment using Jinja2 templating. The objective is to ensure that Apache HTTP Server (`httpd`) is installed and running on **App Server 3** (`stapp03`), and that a dynamically generated `index.html` file is deployed under the web root with correct ownership and permissions.

The implementation is executed from a dedicated `jump-host`, targeting the application server via a pre-configured inventory. The Jinja2 template leverages Ansible's built-in `inventory_hostname` variable to render the correct server name at execution time, eliminating hard-coded values and making the role reusable across the fleet.

---

## Architecture and Design Intent

```
jump-host (thor)
    |
    |-- ansible/
    |       |-- ansible.cfg
    |       |-- inventory               (stapp01, stapp02, stapp03)
    |       |-- playbook.yml            (targets stapp03, applies role/httpd)
    |       |-- role/
    |               |-- httpd/
    |                       |-- tasks/main.yml      (install + start + deploy template)
    |                       |-- templates/
    |                               |-- index.html.j2
    |
    v
stapp03 (banner@stapp03)
    |
    |-- /var/www/html/index.html
            owner: banner
            group: banner
            mode:  0744
```

**Design rationale:**

* The role structure enforces separation of concerns: task logic, template content, and defaults each live in their own subdirectory.
* The Jinja2 template variable `{{ inventory_hostname }}` resolves dynamically at playbook runtime, making the role portable without modification to any other host in the inventory.
* The `become: yes` / `become_user: root` directive in the playbook ensures privileged operations (package install, service management, file placement) execute correctly without requiring direct root SSH access.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Control Node | jump-host running as user `thor` |
| Target Host | stapp03 (App Server 3) |
| Target OS | RHEL/CentOS (yum-based package manager) |
| Ansible | Installed and configured on jump-host |
| SSH Access | Pre-shared credentials defined in inventory via `ansible_ssh_pass` |
| Privilege Escalation | `become: yes` configured at play level in playbook |

---

## Repository Structure

```
ansible/
|-- ansible.cfg
|-- inventory
|-- playbook.yml
|-- role/
    |-- httpd/
        |-- README.md
        |-- defaults/
        |   |-- main.yml
        |-- handlers/
        |   |-- main.yml
        |-- meta/
        |   |-- main.yml
        |-- tasks/
        |   |-- main.yml
        |-- templates/
        |   |-- index.html.j2
        |-- tests/
        |   |-- inventory
        |   |-- test.yml
        |-- vars/
            |-- main.yml
```

---

## Implementation Guide

### Step 1: Inspect the Home Directory on the Jump Host

Log in to the jump-host as `thor` and list the home directory to confirm the working environment:

```bash
thor@jump-host ~$ ls -la
```

**Output:**

```
total 40
drwx------ 1 thor thor 4096 May 19 02:49 .
drwxr-xr-x 1 root root 4096 Mar  5 10:51 ..
-rw-r--r-- 1 thor thor   18 Feb 15  2024 .bash_logout
-rw-r--r-- 1 thor thor  141 Feb 15  2024 .bash_profile
-rw-r--r-- 1 thor thor  581 Mar  5 10:52 .bashrc
drwxr-xr-x 1 thor thor 4096 Mar  5 10:52 .config
drwxr-xr-x 3 thor thor 4096 May 19 02:49 ansible
```

The `ansible/` directory is confirmed present under the `thor` home directory.

> Screenshot: Jump-host home directory listing confirming the ansible directory is present

<img width="498" height="311" alt="image" src="https://github.com/user-attachments/assets/e411b8f6-ea1d-426f-ae7e-fb15a10b2fa9" />

---

### Step 2: Review the Ansible Inventory

```bash
thor@jump-host ~$ cat ~/ansible/inventory
```

**Output:**

```
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner
```

Three application servers are defined. `stapp03` is managed via the `banner` user. This is the target host for the deployment, and `banner` will be set as the owner of the rendered file.

> Screenshot: Inventory file showing all three application servers with their respective users and credentials

<img width="517" height="347" alt="image" src="https://github.com/user-attachments/assets/b37c25d5-1903-41ce-9354-60cb2c13b488" />

---

### Step 3: Inspect the Existing Playbook

```bash
thor@jump-host ~$ cat ~/ansible/playbook.yml
```

**Output:**

```yaml
---
- hosts: 
  become: yes
  become_user: root
  roles:
    - role/httpd
```

The `hosts` key is empty. The playbook has no execution target as written and must be corrected before it can run against any host.

> Screenshot: Original playbook.yml showing the empty hosts field

<img width="514" height="380" alt="image" src="https://github.com/user-attachments/assets/ec1a9c7e-c4b9-490f-80e6-98c8944630aa" />

---

### Step 4: Audit the Full Ansible Directory Tree

```bash
thor@jump-host ~$ find ~/ansible -type f | sort
```

**Output:**

```
/home/thor/ansible/ansible.cfg
/home/thor/ansible/inventory
/home/thor/ansible/playbook.yml
/home/thor/ansible/role/httpd/README.md
/home/thor/ansible/role/httpd/defaults/main.yml
/home/thor/ansible/role/httpd/handlers/main.yml
/home/thor/ansible/role/httpd/meta/main.yml
/home/thor/ansible/role/httpd/tasks/main.yml
/home/thor/ansible/role/httpd/tests/inventory
/home/thor/ansible/role/httpd/tests/test.yml
/home/thor/ansible/role/httpd/vars/main.yml
```

The standard Ansible role skeleton is in place. Notably, there is no `templates/` subdirectory present yet. It must be created before the `template` module task can be added.

> Screenshot: find command output showing the full file tree with no templates directory present

<img width="514" height="338" alt="image" src="https://github.com/user-attachments/assets/e9732a5a-0877-48d1-9e4b-7db851419ef6" />

---

### Step 5: Review the Existing Role Task File

```bash
thor@jump-host ~$ cat /home/thor/ansible/role/httpd/tasks/main.yml
```

**Output:**

```yaml
---
# tasks file for role/test

- name: install the latest version of HTTPD
  yum:
    name: httpd
    state: latest

- name: Start service httpd
  service:
    name: httpd
    state: started
```

Two tasks are currently defined: package installation via `yum` and service activation via `service`. A third task for template-based file deployment will be appended after the template infrastructure is created.

> Screenshot: Existing tasks/main.yml showing the two initial tasks before any modification

<img width="514" height="349" alt="image" src="https://github.com/user-attachments/assets/b18426ff-2a04-4c37-b080-ad6e756dfe6c" />

---

### Step 6: Overwrite the Playbook to Target App Server 3

Use a heredoc to overwrite `playbook.yml` with `stapp03` set as the target host:

```bash
thor@jump-host ~$ cat > ~/ansible/playbook.yml << 'EOF'
---
- hosts: stapp03
  become: yes
  become_user: root
  roles:
    - role/httpd
EOF
```

The single-quoted `'EOF'` delimiter prevents the shell from expanding any variables inside the heredoc block, ensuring the content is written exactly as specified.

> Screenshot: heredoc command overwriting playbook.yml with stapp03 as the target host

<img width="514" height="299" alt="image" src="https://github.com/user-attachments/assets/c47d3e0d-14b6-4136-8fe6-54717c84ec39" />

---

### Step 7: Verify the Updated Playbook

```bash
thor@jump-host ~$ cat ~/ansible/playbook.yml
```

**Output:**

```yaml
---
- hosts: stapp03
  become: yes
  become_user: root
  roles:
    - role/httpd
```

The `hosts` field now correctly points to `stapp03`. Privilege escalation is retained and the `role/httpd` role reference is intact.

> Screenshot: Updated playbook.yml confirming stapp03 is set as the target host

<img width="515" height="386" alt="image" src="https://github.com/user-attachments/assets/313d9125-e646-4726-9e06-631c926ca3a7" />

---

### Step 8: Create the Templates Directory

```bash
thor@jump-host ~$ mkdir -p /home/thor/ansible/role/httpd/templates/
```

The `-p` flag ensures the command succeeds without error even if any parent directories in the path do not yet exist. This creates the `templates/` subdirectory that Ansible's `template` module requires for resolving the `src` parameter at role execution time.

> Screenshot: mkdir command creating the templates directory inside the httpd role

<img width="519" height="233" alt="image" src="https://github.com/user-attachments/assets/119e6545-7603-4c4f-8d7c-640c38460cba" />

---

### Step 9: Create the Jinja2 Template File

```bash
thor@jump-host ~$ cat > /home/thor/ansible/role/httpd/templates/index.html.j2 << 'EOF'
This file was created using Ansible on {{ inventory_hostname }}
EOF
```

The template contains a single line with the `{{ inventory_hostname }}` Jinja2 variable. Ansible resolves this variable at runtime to the name of the host being targeted, which in this case will render as `stapp03`. The quoted heredoc delimiter `'EOF'` ensures the Jinja2 syntax is written to disk literally and is not interpreted by the shell.

> Screenshot: heredoc command writing the Jinja2 template file with the inventory_hostname variable

<img width="515" height="269" alt="image" src="https://github.com/user-attachments/assets/78075f9e-721b-43a7-a5d1-01886063bb6f" />

---

### Step 10: Verify the Template File Content

```bash
thor@jump-host ~$ cat /home/thor/ansible/role/httpd/templates/index.html.j2
```

**Output:**

```
This file was created using Ansible on {{ inventory_hostname }}
```

The Jinja2 syntax is preserved exactly as written. The `{{ inventory_hostname }}` placeholder is confirmed present and uninterpreted.

> Screenshot: Template file content confirming the Jinja2 variable is intact on disk

<img width="515" height="296" alt="image" src="https://github.com/user-attachments/assets/4c917aa4-95d1-4d11-b170-fcac1a5b3c88" />

---

### Step 11: Append the Template Deployment Task to the Role

```bash
thor@jump-host ~$ cat >> /home/thor/ansible/role/httpd/tasks/main.yml << 'EOF'

- name: Copy index.html template to App Server 3
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: banner
    group: banner
    mode: "0744"
EOF
```

The `>>` operator appends to the existing file, preserving the two original tasks. The new task uses the `template` module to render `index.html.j2` and place the output at `/var/www/html/index.html` on the target server.

**Parameter breakdown:**

| Parameter | Value | Reason |
|---|---|---|
| `src` | `index.html.j2` | Template filename resolved relative to the role's `templates/` directory |
| `dest` | `/var/www/html/index.html` | Standard Apache HTTP server web root path |
| `owner` | `banner` | Sudo user for stapp03 as defined in the inventory |
| `group` | `banner` | Matching group ownership to align with the file owner |
| `mode` | `"0744"` | Owner read/write/execute; group and others read-only |

> Screenshot: Appending the template deployment task to tasks/main.yml using the append heredoc operator

<img width="515" height="319" alt="image" src="https://github.com/user-attachments/assets/a1522844-d31f-421f-94b2-bdb66ec144ec" />

---

### Step 12: Verify the Final Role Task File

```bash
thor@jump-host ~$ cat /home/thor/ansible/role/httpd/tasks/main.yml
```

**Output:**

```yaml
---
# tasks file for role/test

- name: install the latest version of HTTPD
  yum:
    name: httpd
    state: latest

- name: Start service httpd
  service:
    name: httpd
    state: started
- name: Copy index.html template to App Server 3
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: banner
    group: banner
    mode: "0744"
```

All three tasks are present in the correct order: package installation, service activation, and template deployment. The original two tasks remain unmodified.

> Screenshot: Final tasks/main.yml showing all three tasks in the correct sequence

<img width="514" height="416" alt="image" src="https://github.com/user-attachments/assets/72aeef5b-d171-482a-9168-93b0ef1f1a72" />

---

### Step 13: Run Playbook Syntax Check

Navigate to the ansible working directory and run a syntax check before executing:

```bash
thor@jump-host ~$ cd ~/ansible && ansible-playbook -i inventory playbook.yml --syntax-check
```

**Output:**

```
playbook: playbook.yml
```

The clean output with no errors confirms the playbook structure, role reference, and task YAML are all syntactically valid. The directory was changed to `~/ansible` before running so that relative paths in `ansible.cfg` and the inventory resolve correctly.

> Screenshot: Syntax check output returning cleanly with no errors

<img width="518" height="325" alt="image" src="https://github.com/user-attachments/assets/3a251137-103f-45b8-a6eb-54534bd2a0a0" />

---

### Step 14: Execute the Playbook

```bash
thor@jump-host ~/ansible$ ansible-playbook -i inventory playbook.yml
```

**Output:**

```
PLAY [stapp03] ****************************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [stapp03]

TASK [role/httpd : install the latest version of HTTPD] ***********************************
changed: [stapp03]

TASK [role/httpd : Start service httpd] ***************************************************
changed: [stapp03]

TASK [role/httpd : Copy index.html template to App Server 3] ******************************
changed: [stapp03]

PLAY RECAP ********************************************************************************
stapp03                    : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

**Play recap analysis:**

| Counter | Value | Interpretation |
|---|---|---|
| `ok` | 4 | All tasks including fact gathering completed without error |
| `changed` | 3 | HTTPD installed, service started, and index.html deployed |
| `unreachable` | 0 | stapp03 was reachable throughout the entire play |
| `failed` | 0 | No task failures |
| `skipped` | 0 | No tasks were conditionally skipped |

> Screenshot: Full ansible-playbook run output showing the play recap with ok=4 and changed=3

<img width="517" height="322" alt="image" src="https://github.com/user-attachments/assets/0ff15429-70db-46e3-ae9b-0a5a227b96ff" />

---

### Step 15: SSH into App Server 3 and Verify the Deployed File

SSH into `stapp03` as the `banner` user:

```bash
thor@jump-host ~/ansible$ ssh banner@stapp03
```

Enter password `BigGr33n` when prompted:

```
banner@stapp03's password:
Last login: Tue May 19 03:04:13 2026 from 10.244.13.50
```

Verify the rendered file content and its filesystem metadata:

```bash
[banner@stapp03 ~]$ cat /var/www/html/index.html
ls -la /var/www/html/index.html
```

**Output:**

```
This file was created using Ansible on stapp03
-rwxr--r-- 1 banner banner 47 May 19 03:04 /var/www/html/index.html
```

**Verification checklist:**

* `{{ inventory_hostname }}` correctly resolved to `stapp03` in the rendered file content
* File permissions are `0744`, confirmed by the `-rwxr--r--` mode string
* Owner is `banner` and group is `banner` as declared in the task
* File size is 47 bytes, consistent with the expected rendered content length

Exit the SSH session:

```bash
[banner@stapp03 ~]$ exit
logout
Connection to stapp03 closed.
```

> Screenshot: SSH session on stapp03 showing cat output and ls -la confirming rendered content, ownership, and permissions

---

## Best Practices Applied

* **Jinja2 variable substitution over hard-coded server names:** Using `{{ inventory_hostname }}` in the template instead of a literal string makes the template reusable across any host in the inventory. Adding another server to the role requires zero template modification.

* **Quoted heredoc delimiter to protect Jinja2 syntax:** All file write operations used `<< 'EOF'` with a single-quoted delimiter. This prevents the shell from evaluating `{{ inventory_hostname }}` as a variable during the write, ensuring the Jinja2 syntax reaches the file on disk exactly as intended.

* **Intermediate verification after each write operation:** Every file creation or modification step was immediately followed by a `cat` command to confirm the written content before proceeding. This catch-early pattern prevents downstream failures caused by silent write errors or unexpected shell behaviour.

* **Append operator for non-destructive task file extension:** The `>>` operator was used to add the third task to `tasks/main.yml` rather than rewriting the entire file. This preserves the original task content and its exact formatting, minimising the risk of introducing indentation errors into existing YAML.

* **Explicit owner, group, and mode in the template task:** Rather than relying on the target server's default umask, `owner`, `group`, and `mode` were declared explicitly. This makes the desired file state self-documenting and reproducible across any environment without dependency on system-level defaults.

* **Syntax check as a mandatory pre-flight step:** Running `--syntax-check` before live execution is a zero-cost safeguard that catches YAML structure errors, undefined role paths, and module parameter issues before they affect the target host.

* **Privilege escalation declared at play level:** `become: yes` and `become_user: root` are set once at the play level rather than repeated per task. This keeps the task file clean and ensures all tasks in the role execute with consistent and predictable privilege levels.

---

## Lessons Learned

* **An empty `hosts` field in a playbook produces no execution and no error.** The original `playbook.yml` had `hosts:` with no value. Ansible treats this as a valid play that matches zero hosts and exits silently without running any tasks. Always confirm the `hosts` field is populated before running.

* **The `templates/` directory must be created manually when it is absent from an existing role.** Ansible's `template` module resolves the `src` parameter relative to the `templates/` subdirectory of the role at execution time. If the directory does not exist, the task fails with a file-not-found error. Auditing the role's directory tree with `find` before adding new module types is the correct pre-check.

* **Mode values must be passed as quoted strings to avoid YAML integer parsing issues.** Writing `mode: 0744` without quotes causes some YAML parsers to interpret the leading zero as an octal literal and convert it, resulting in unexpected numeric values being written to disk. Using `mode: "0744"` as a string is the safe, version-consistent convention.

* **Verifying file content after every write operation catches silent failures early.** Each heredoc write was followed immediately by a `cat` to confirm the result. Without this step, a mistake in the heredoc content would only surface during playbook execution or post-deployment verification, making root cause identification significantly harder.

* **Direct post-execution SSH verification is the definitive confirmation of a successful deployment.** A clean play recap with zero failures confirms the state Ansible reported, but does not replace direct inspection. SSHing to the target server and running `cat` and `ls -la` validates that the Jinja2 variable resolved correctly, the content is accurate, and the file permissions and ownership match exactly what was declared in the task.

---

## Technologies Used

| Technology | Purpose |
|---|---|
| Ansible | Configuration management and orchestration engine |
| Ansible Roles | Modular encapsulation of tasks, templates, handlers, and variables |
| Jinja2 | Dynamic template rendering using Ansible inventory variables |
| Apache HTTPD (`httpd`) | Web server installed and managed on stapp03 |
| `yum` module | Package installation on RHEL/CentOS target host |
| `service` module | Service lifecycle management (start/enable) |
| `template` module | Jinja2 template rendering and remote file deployment |
| SSH | Secure remote access from jump-host to application server |












<img width="513" height="323" alt="image" src="https://github.com/user-attachments/assets/3f2d9f82-08b0-41d2-b0a3-742167a45be7" />
<img width="514" height="378" alt="image" src="https://github.com/user-attachments/assets/f68528b1-22de-499d-85b8-156e1008245c" />
<img width="514" height="417" alt="image" src="https://github.com/user-attachments/assets/2e16452c-cba6-4b60-a864-271282f2baae" />


