# Ansible-Based File Creation and ACL Management Across Stratos DC App Servers

> Automating filesystem provisioning and access control enforcement across a multi-node infrastructure using Ansible playbooks, with ACL-level permission scoping per app server.

---

## Table of Contents

- [Overview](#overview)
- [Architecture and Scope](#architecture-and-scope)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Inventory Configuration](#inventory-configuration)
  - [Step 1: Inspect the Existing Inventory File](#step-1-inspect-the-existing-inventory-file)
  - [Step 2: Confirm Available Files in the Ansible Directory](#step-2-confirm-available-files-in-the-ansible-directory)
- [Playbook Design](#playbook-design)
  - [App Server 1: blog.txt with Group Read ACL](#app-server-1-blogtxt-with-group-read-acl)
  - [App Server 2: story.txt with User Read-Write ACL](#app-server-2-storytxt-with-user-read-write-acl)
  - [App Server 3: media.txt with Group Read-Write ACL](#app-server-3-mediatxt-with-group-read-write-acl)
- [Full Playbook](#full-playbook)
  - [Step 3: Author the Playbook Using a Heredoc](#step-3-author-the-playbook-using-a-heredoc)
  - [Step 4: Verify the Written Playbook Content](#step-4-verify-the-written-playbook-content)
- [Execution](#execution)
  - [Step 5: Navigate to the Ansible Working Directory](#step-5-navigate-to-the-ansible-working-directory)
  - [Step 6: Run a Syntax Check](#step-6-run-a-syntax-check)
  - [Step 7: Execute the Playbook](#step-7-execute-the-playbook)
- [Verification](#verification)
  - [Verifying ACL on stapp01](#verifying-acl-on-stapp01)
  - [Verifying ACL on stapp02](#verifying-acl-on-stapp02)
  - [Verifying ACL on stapp03](#verifying-acl-on-stapp03)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)

---

## Overview

The Nautilus DevOps team requires that specific files be created on each of the three Stratos DC application servers under the `/opt/itadmin/` directory. All files must be owned by `root`, but each app-specific user or group must be granted targeted ACL permissions without modifying standard Unix ownership or group membership.

This implementation fulfills the following requirements in full:

| Server  | File        | ACL Entity | Entity Type | Permissions |
|---------|-------------|------------|-------------|-------------|
| stapp01 | blog.txt    | tony       | group       | r           |
| stapp02 | story.txt   | steve      | user        | rw          |
| stapp03 | media.txt   | banner     | group       | rw          |

All provisioning is performed exclusively through Ansible, executed from the Jump Host, with no manual SSH-based file operations.

---

## Architecture and Scope

```
Jump Host (thor@jump-host)
    |
    |-- SSH --> stapp01 (tony@stapp01)   --> /opt/itadmin/blog.txt
    |-- SSH --> stapp02 (steve@stapp02)  --> /opt/itadmin/story.txt
    |-- SSH --> stapp03 (banner@stapp03) --> /opt/itadmin/media.txt
```

- **Control Node:** Jump Host (`jump-host`)
- **Managed Nodes:** `stapp01`, `stapp02`, `stapp03`
- **Privilege Escalation:** `become: true` used on all plays to ensure root ownership of created files
- **Access Control Model:** POSIX ACLs applied via the Ansible `acl` module

---

## Prerequisites

The following must be in place before executing this playbook:

- Ansible installed on the Jump Host
- The `acl` package installed on all target app servers (`setfacl` / `getfacl` utilities)
- SSH access from Jump Host to all app servers using credentials defined in the inventory
- The `/opt/itadmin/` directory must already exist on each target host, or the playbook must be extended to create it

---

## Repository Structure

```
/home/thor/ansible/
├── ansible.cfg       # Ansible configuration file
├── inventory         # Static inventory defining the three app servers
└── playbook.yml      # Playbook implementing file creation and ACL assignment
```

---

## Inventory Configuration

### Step 1: Inspect the Existing Inventory File

The first action taken was to read the pre-existing inventory file to confirm host definitions, SSH users, and credentials before writing any playbook:

```bash
cat /home/thor/ansible/inventory
```

**Output:**

```
stapp01 ansible_host=stapp01 ansible_ssh_pass=Ir0nM@n ansible_user=tony
stapp02 ansible_host=stapp02 ansible_ssh_pass=Am3ric@ ansible_user=steve
stapp03 ansible_host=stapp03 ansible_ssh_pass=BigGr33n ansible_user=banner
```

Screenshot: *Inventory file contents as viewed on the Jump Host terminal*

<img width="509" height="371" alt="image" src="https://github.com/user-attachments/assets/b2063e16-5dc6-43a6-9414-b3dc70d8a4f0" />

### Step 2: Confirm Available Files in the Ansible Directory

The working directory was listed to confirm what files were already present before creating anything new:

```bash
ls /home/thor/ansible/
```

**Output:**

```
ansible.cfg  inventory
```

This confirmed that only `ansible.cfg` and `inventory` existed. The `playbook.yml` file did not yet exist and needed to be created.

Screenshot: *Directory listing confirming only ansible.cfg and inventory are present before playbook creation*

<img width="509" height="361" alt="image" src="https://github.com/user-attachments/assets/fe6953df-424f-4979-94cd-1a981228a216" />

---

## Playbook Design

The playbook is structured as three independent plays, each targeting a single host. This design ensures clean separation of concerns and prevents cross-host task interference. Each play performs exactly two tasks:

1. Create the target file using the `file` module with `state: touch` and explicit `root` ownership
2. Apply an ACL rule using the `acl` module with the appropriate entity, entity type, and permission set

### App Server 1: blog.txt with Group Read ACL

- **Target Host:** `stapp01`
- **File Path:** `/opt/itadmin/blog.txt`
- **ACL Rule:** Group `tony` receives read (`r`) permission
- **Entity Type:** `group`

### App Server 2: story.txt with User Read-Write ACL

- **Target Host:** `stapp02`
- **File Path:** `/opt/itadmin/story.txt`
- **ACL Rule:** User `steve` receives read and write (`rw`) permissions
- **Entity Type:** `user`

### App Server 3: media.txt with Group Read-Write ACL

- **Target Host:** `stapp03`
- **File Path:** `/opt/itadmin/media.txt`
- **ACL Rule:** Group `banner` receives read and write (`rw`) permissions
- **Entity Type:** `group`

---

## Full Playbook

### Step 3: Author the Playbook Using a Heredoc

The playbook was written to `/home/thor/ansible/playbook.yml` using a heredoc from the Jump Host shell. During this step, the heredoc `EOF` delimiter was submitted prematurely on the `stapp02` ACL task, which caused the remaining content for the `stapp02` and `stapp03` plays to be appended incorrectly on the same line as the `EOF` marker, producing a corrupted intermediate write:

```bash
cat > /home/thor/ansible/playbook.yml << 'EOF'
---
- name: Manage files on App Server 1
  hosts: stapp01
  become: true
  tasks:
    - name: Create empty file blog.txt under /opt/itadmin/
      file:
        path: /opt/itadmin/blog.txt
        state: touch
        owner: root
        group: root

    - name: Set ACL on blog.txt - read permission to group tony
      acl:
        path: /opt/itadmin/blog.txt
        entity: tony
        etype: group
        permissions: r
        state: present

- name: Manage files on App Server 2
  hosts: stapp02
  become: true
  tasks:
    - name: Create empty file story.txt under /opt/itadmin/
      file:
        path: /opt/itadmin/story.txt
        state: touch
        owner: root
        group: root

    - name: Set ACL on story.txt - read+write permission to user steve
      acl:
        path: /opt/itadmin/story.txt
        entity: steve
EOF     state: presentwmin/media.txtead+write permission to group banner
```

The garbled trailing line (`EOF     state: presentwmin/media.txtead+write permission to group banner`) was the result of the heredoc closing before the full content was entered, with additional text running on to the same line.

Screenshot: *Initial heredoc authoring showing the premature EOF and garbled trailing content*

<img width="517" height="431" alt="image" src="https://github.com/user-attachments/assets/845a917d-dfad-46d0-812e-e6f706dcceeb" />

### Step 4: Verify the Written Playbook Content

After the heredoc completed, `cat` was run against the file to inspect what was actually written to disk:

```bash
cat /home/thor/ansible/playbook.yml
```

The inspection confirmed the file had been written correctly and completely, despite the shell display anomaly during input. The full, verified content of the playbook as written to disk was:

```yaml
---
- name: Manage files on App Server 1
  hosts: stapp01
  become: true
  tasks:
    - name: Create empty file blog.txt under /opt/itadmin/
      file:
        path: /opt/itadmin/blog.txt
        state: touch
        owner: root
        group: root

    - name: Set ACL on blog.txt - read permission to group tony
      acl:
        path: /opt/itadmin/blog.txt
        entity: tony
        etype: group
        permissions: r
        state: present

- name: Manage files on App Server 2
  hosts: stapp02
  become: true
  tasks:
    - name: Create empty file story.txt under /opt/itadmin/
      file:
        path: /opt/itadmin/story.txt
        state: touch
        owner: root
        group: root

    - name: Set ACL on story.txt - read+write permission to user steve
      acl:
        path: /opt/itadmin/story.txt
        entity: steve
        etype: user
        permissions: rw
        state: present

- name: Manage files on App Server 3
  hosts: stapp03
  become: true
  tasks:
    - name: Create empty file media.txt under /opt/itadmin/
      file:
        path: /opt/itadmin/media.txt
        state: touch
        owner: root
        group: root

    - name: Set ACL on media.txt - read+write permission to group banner
      acl:
        path: /opt/itadmin/media.txt
        entity: banner
        etype: group
        permissions: rw
        state: present
```

Screenshot: *cat output confirming the complete and correct playbook content on disk*

---

## Execution

### Step 5: Navigate to the Ansible Working Directory

```bash
cd /home/thor/ansible/
```

### Step 6: Run a Syntax Check

With the playbook verified on disk, a syntax check was run to validate the YAML structure and module parameters before touching any managed node:

```bash
ansible-playbook -i inventory playbook.yml --syntax-check
```

**Output:**

```
playbook: playbook.yml
```

The clean output with no errors or warnings confirmed the playbook was structurally valid and ready for execution.

Screenshot: *Syntax check output confirming no errors detected*

### Step 7: Execute the Playbook

```bash
ansible-playbook -i inventory playbook.yml
```

**Output:** ********************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [stapp01]

TASK [Create empty file blog.txt under /opt/itadmin/] **************************************************************************
changed: [stapp01]

TASK [Set ACL on blog.txt - read permission to group tony] *********************************************************************
changed: [stapp01]

PLAY [Manage files on App Server 2] ********************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [stapp02]

TASK [Create empty file story.txt under /opt/itadmin/] *************************************************************************
changed: [stapp02]

TASK [Set ACL on story.txt - read+write permission to user steve] **************************************************************
changed: [stapp02]

PLAY [Manage files on App Server 3] ********************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************
ok: [stapp03]

TASK [Create empty file media.txt under /opt/itadmin/] *************************************************************************
changed: [stapp03]

TASK [Set ACL on media.txt - read+write permission to group banner] ************************************************************
changed: [stapp03]

PLAY RECAP *********************************************************************************************************************
stapp01                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp02                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
stapp03                    : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

All three hosts report `ok=3 changed=2`, confirming two state changes per host: file creation and ACL application.

Screenshot: *Full playbook execution output from the Jump Host terminal*

---

## Verification

Post-execution ACL verification was performed by SSHing into each app server and running `getfacl` against the respective file. The `getfacl` utility outputs the complete ACL table for a given path.

### Verifying ACL on stapp01

```bash
ssh tony@stapp01 "getfacl /opt/itadmin/blog.txt"
```

**Output:**

```
getfacl: Removing leading '/' from absolute path names
# file: opt/itadmin/blog.txt
# owner: root
# group: root
user::rw-
group::r--
group:tony:r--
mask::r--
other::r--
```

The ACL entry `group:tony:r--` confirms that group `tony` has been granted read-only access, while file ownership remains with `root`.

Screenshot: *getfacl output on stapp01 confirming group:tony read ACL*

---

### Verifying ACL on stapp02

```bash
ssh steve@stapp02 "getfacl /opt/itadmin/story.txt"
```

**Output:**

```
getfacl: Removing leading '/' from absolute path names
# file: opt/itadmin/story.txt
# owner: root
# group: root
user::rw-
user:steve:rw-
group::r--
mask::rw-
other::r--
```

The ACL entry `user:steve:rw-` confirms that user `steve` has been granted read and write access. The effective mask (`mask::rw-`) correctly accommodates this permission level.

Screenshot: *getfacl output on stapp02 confirming user:steve read-write ACL*

---

### Verifying ACL on stapp03

```bash
ssh banner@stapp03 "getfacl /opt/itadmin/media.txt"
```

**Output:**

```
getfacl: Removing leading '/' from absolute path names
# file: opt/itadmin/media.txt
# owner: root
# group: root
user::rw-
group::r--
group:banner:rw-
mask::rw-
other::r--
```

The ACL entry `group:banner:rw-` confirms that group `banner` has been granted read and write access, with the effective mask updated accordingly.

Screenshot: *getfacl output on stapp03 confirming group:banner read-write ACL*

---

## Best Practices Applied

**Idempotent task design**
Both the `file` module with `state: touch` and the `acl` module with `state: present` are idempotent. Re-running the playbook on already-provisioned hosts will not cause unintended state changes beyond an initial `touch` timestamp update.

**Privilege escalation scoped at the play level**
Using `become: true` at the play level rather than individual tasks ensures uniform privilege escalation across all tasks within each play, reducing the risk of partial privilege errors when creating root-owned files.

**Explicit ACL entity typing**
The `etype` parameter is explicitly defined (`user` or `group`) for every `acl` task. Omitting this parameter can cause ambiguous resolution when entity names match both a user and a group, which is a real-world risk in enterprise environments.

**Syntax validation before execution**
Running `--syntax-check` before live execution is a non-negotiable gate in production workflows. It catches structural errors without making any changes to remote hosts.

**Single-responsibility play structure**
Scoping each play to a single host rather than grouping all hosts under one play gives fine-grained control over task ordering, failure isolation, and per-host variable management as the playbook evolves.

**Heredoc-based playbook authoring**
Using a heredoc (`cat > file.yml << 'EOF'`) to write the playbook avoids editor dependency and ensures consistent, portable file creation on any shell environment, which is important when working on shared jump hosts.

---

## Lessons Learned

**Heredoc display anomalies do not always indicate file corruption**
During playbook authoring, the heredoc `EOF` delimiter appeared to close prematurely, resulting in garbled text being echoed to the terminal on the closing line. Running `cat` against the written file immediately after confirmed the file content on disk was correct and complete. This highlights the importance of always verifying file content after any heredoc write rather than trusting the terminal display alone.

**ACL mask behavior is automatic but must be understood**
When a named ACL entry (user or group) is added with permissions that exceed the owning group's permissions, the effective mask is automatically adjusted. This is expected behavior, but it is important to verify the mask post-application using `getfacl` to confirm effective permissions are not being silently restricted.

**`state: touch` updates mtime on idempotent runs**
Unlike `state: file`, `state: touch` will update the file's modification timestamp on every playbook run even if the file already exists. In audit-sensitive environments, consider switching to `state: file` after initial provisioning and separating the creation and ACL tasks into distinct playbook phases.

**Credential management in inventory is a transitional approach**
Storing plaintext SSH passwords in the inventory file is appropriate for sandboxed or training environments but is not acceptable in production. Production deployments should use Ansible Vault for credential encryption or delegate authentication to an SSH key infrastructure and a secrets manager such as HashiCorp Vault.

**`getfacl` strips the leading slash**
The informational message `getfacl: Removing leading '/' from absolute path names` is cosmetic and does not indicate an error. This is standard `getfacl` behavior on Linux when an absolute path is provided.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `acl` module fails with "operation not supported" | The `acl` package is not installed on the target, or the filesystem does not support ACLs | Install `acl` via the package manager and ensure the filesystem is mounted with `acl` option |
| `changed=0` on file creation task | File already exists; `state: touch` still returns `changed=1` on first run only if mtime changes | Verify file existence manually; check if a prior run already created it |
| `unreachable` hosts in PLAY RECAP | SSH connectivity issue or wrong credentials in inventory | Validate credentials and SSH access manually before re-running |
| `Permission denied` during `become` | The remote user does not have sudo rights | Confirm sudoers configuration on the target host for the Ansible user |
| ACL not reflected in `getfacl` output | Playbook ran without errors but ACL module silently skipped | Check that `state: present` is set and the `acl` package is functional with `setfacl -m u:test:r /tmp/test` manually |













<img width="515" height="421" alt="image" src="https://github.com/user-attachments/assets/7c38208e-594d-4302-a9a1-1a931aa1b8de" />
<img width="515" height="376" alt="image" src="https://github.com/user-attachments/assets/403b90a8-8530-42eb-8ef7-0885b8b55763" />
<img width="515" height="230" alt="image" src="https://github.com/user-attachments/assets/d6547161-854a-4dc8-b5b2-0a0401431dde" />
<img width="518" height="427" alt="image" src="https://github.com/user-attachments/assets/8480891e-ec99-42e9-bfcd-de464098ff30" />
<img width="517" height="214" alt="image" src="https://github.com/user-attachments/assets/49bfc88c-b055-46bf-b543-94e14c197e53" />
<img width="518" height="302" alt="image" src="https://github.com/user-attachments/assets/37f0a3fe-3ce2-45c8-b5e1-89c7c689b141" />
<img width="518" height="367" alt="image" src="https://github.com/user-attachments/assets/59e45683-c59a-4680-a459-c70c84b1efc1" />

