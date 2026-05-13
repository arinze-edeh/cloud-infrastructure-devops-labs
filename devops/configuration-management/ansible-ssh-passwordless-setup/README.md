# Ansible Passwordless SSH Authentication Setup | Stratos DC

## Table of Contents

* [Overview](#overview)
* [Architecture and Context](#architecture-and-context)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify Identity and Environment](#step-1-verify-identity-and-environment)
  * [Step 2: Inspect the SSH Directory](#step-2-inspect-the-ssh-directory)
  * [Step 3: Generate RSA Key Pair](#step-3-generate-rsa-key-pair)
  * [Step 4: Verify Key Generation](#step-4-verify-key-generation)
  * [Step 5: Copy Public Key to App Server 2](#step-5-copy-public-key-to-app-server-2)
  * [Step 6: Validate Passwordless SSH Login](#step-6-validate-passwordless-ssh-login)
  * [Step 7: Inspect the Ansible Inventory File](#step-7-inspect-the-ansible-inventory-file)
  * [Step 8: Test Ansible Ping with Default Inventory](#step-8-test-ansible-ping-with-default-inventory)
  * [Step 9: Run Ansible Ping with Explicit Key Override](#step-9-run-ansible-ping-with-explicit-key-override)
  * [Step 10: Update the Inventory File for Key-Based Auth](#step-10-update-the-inventory-file-for-key-based-auth)
  * [Step 11: Validate Final Ansible Ping](#step-11-validate-final-ansible-ping)
* [Errors and Resolutions](#errors-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Repository Structure](#repository-structure)

---

## Overview

This implementation documents the end-to-end setup of passwordless SSH authentication between an **Ansible controller node** (`jump host`) and a **managed node** (`stapp02` / App Server 2) within the **Stratos DC** environment. The objective is to satisfy a pre-requisite required before Ansible playbook execution across the application server fleet.

The task was executed as the `thor` user on `jump host`, leveraging RSA key-based authentication and a structured Ansible inventory update to replace credential-based access with key-pair-based access for `stapp02`.

| Attribute | Value |
|---|---|
| **Ansible Controller** | `jump host` |
| **Managed Node** | `stapp02` (App Server 2) |
| **Remote User on stapp02** | `steve` |
| **Execution User on jump host** | `thor` |
| **Authentication Method** | RSA 4096-bit Key Pair |
| **Inventory Path** | `/home/thor/ansible/inventory` |

---

## Architecture and Context

The Nautilus DevOps team operates a multi-server application environment within Stratos DC. Ansible playbooks are scheduled for execution across multiple app servers (`stapp01`, `stapp02`, `stapp03`). As a prerequisite, all managed nodes must be reachable via passwordless SSH from the Ansible controller to support automated, non-interactive playbook runs.

This implementation focuses specifically on establishing that trust between `jump host` and `stapp02`.

```
+------------------+          SSH (Key-Based)          +------------------+
|   jump host      |  --------------------------------> |   stapp02        |
|   user: thor     |                                    |   user: steve    |
|   Ansible Ctrl   |    RSA 4096 Public Key Auth        |   Managed Node   |
+------------------+                                    +------------------+
         |
         | Inventory: /home/thor/ansible/inventory
         | ansible_user=steve
         | ansible_ssh_private_key_file=/home/thor/.ssh/id_rsa
```

---

## Prerequisites

* SSH client installed on `jump host`
* `ssh-keygen` and `ssh-copy-id` available in PATH
* Network connectivity between `jump host` and `stapp02` (verified via `ssh-copy-id` handshake)
* Known password for `steve@stapp02` required exactly once during key installation
* Ansible installed on `jump host` with access to the inventory file at `/home/thor/ansible/inventory`

---

## Implementation Guide

### Step 1: Verify Identity and Environment

Before any configuration work, confirm the active user identity and hostname to ensure operations are performed on the correct node.

```bash
thor@jump-host ~$ whoami
thor

thor@jump-host ~$ hostname
jump-host
```

> This confirms execution context as user `thor` on `jump host`, the designated Ansible controller.

*Screenshot: Terminal output confirming `whoami` returns `thor` and `hostname` returns `jump-host`*

<img width="509" height="284" alt="image" src="https://github.com/user-attachments/assets/a11a4f7f-35dc-40b1-b830-3350849e575d" />

---

### Step 2: Inspect the SSH Directory

Check the current state of the `.ssh` directory to confirm no existing keys are present before generating new ones.

```bash
thor@jump-host ~$ ls -la /home/thor/.ssh/
total 12
drwx------ 2 thor thor 4096 May 13 05:11 .
drwx------ 1 thor thor 4096 May 13 05:11 ..
```

> The `.ssh` directory exists with correct `700` permissions but contains no key files, confirming a clean starting state.

*Screenshot: `ls -la /home/thor/.ssh/` showing an empty directory with no key files*

<img width="508" height="283" alt="image" src="https://github.com/user-attachments/assets/d6f2231c-b0be-482d-9e97-eff9736103fe" />

---

### Step 3: Generate RSA Key Pair

Generate a 4096-bit RSA key pair non-interactively. The `-N ""` flag sets an empty passphrase to enable fully automated, passwordless Ansible operations.

```bash
thor@jump-host ~$ ssh-keygen -t rsa -b 4096 -C "thor@jump-host" -f /home/thor/.ssh/id_rsa -N ""
Generating public/private rsa key pair.
Your identification has been saved in /home/thor/.ssh/id_rsa
Your public key has been saved in /home/thor/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:R+qdlEWKTxHlS46jAqycN5Dd5uFXSEYgq1A7FoV6Bbk thor@jump-host
```

**Key generation parameters:**

| Flag | Value | Purpose |
|---|---|---|
| `-t rsa` | RSA algorithm | Industry-standard asymmetric key algorithm |
| `-b 4096` | 4096 bits | High-entropy key for production-grade security |
| `-C "thor@jump-host"` | Comment | Human-readable identifier for key tracking |
| `-f /home/thor/.ssh/id_rsa` | Output path | Explicit key location |
| `-N ""` | Empty passphrase | Enables non-interactive Ansible SSH operations |

*Screenshot: `ssh-keygen` output showing key fingerprint and randomart image confirming successful key generation*

<img width="507" height="380" alt="image" src="https://github.com/user-attachments/assets/d8dd7cb6-c76f-4788-a658-1c821f83d01d" />

---

### Step 4: Verify Key Generation

Confirm both the private key (`id_rsa`) and public key (`id_rsa.pub`) were created with correct file permissions.

```bash
thor@jump-host ~$ ls -la /home/thor/.ssh/
total 20
drwx------ 2 thor thor 4096 May 13 05:21 .
drwx------ 1 thor thor 4096 May 13 05:11 ..
-rw------- 1 thor thor 3381 May 13 05:21 id_rsa
-rw-r--r-- 1 thor thor  740 May 13 05:21 id_rsa.pub
```

> The private key (`id_rsa`) carries `600` permissions (owner read/write only). The public key (`id_rsa.pub`) carries `644` permissions (world-readable, as required). Both are correct and expected.

*Screenshot: `ls -la /home/thor/.ssh/` showing `id_rsa` at `600` and `id_rsa.pub` at `644`*

---

### Step 5: Copy Public Key to App Server 2

Use `ssh-copy-id` to install the public key into the `authorized_keys` file of `steve@stapp02`. This step requires the password for `steve` exactly once.

```bash
thor@jump-host ~$ ssh-copy-id steve@stapp02
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/thor/.ssh/id_rsa.pub"
The authenticity of host 'stapp02 (10.244.195.10)' can't be established.
ED25519 key fingerprint is SHA256:DFHMIMiGuNK+ndUbdCA5HU/27XTBaQMSEA+1EtSz8hU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
steve@stapp02's password:
Number of key(s) added: 1
```

> `ssh-copy-id` appends the public key to `/home/steve/.ssh/authorized_keys` on `stapp02`, and sets correct ownership and permissions on the remote side automatically.

*Screenshot: `ssh-copy-id` output showing `Number of key(s) added: 1` confirming successful key installation on stapp02*

---

### Step 6: Validate Passwordless SSH Login

Confirm that SSH access to `stapp02` as `steve` now works without a password prompt.

```bash
thor@jump-host ~$ ssh steve@stapp02
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
```

> A clean login and logout with no password prompt confirms that key-based authentication is fully operational between `jump host` and `stapp02`.

*Screenshot: `ssh steve@stapp02` landing directly at the `stapp02` shell prompt without a password prompt, followed by `exit`*

---

### Step 7: Inspect the Ansible Inventory File

Review the existing inventory file to understand the current authentication configuration across all three app servers.

```bash
thor@jump-host ~$ cat /home/thor/ansible/inventory
stapp01 ansible_ssh_pass=Ir0nM@n
stapp02 ansible_ssh_pass=Am3ric@
stapp03 ansible_ssh_pass=BigGr33n
```

> All three servers currently use `ansible_ssh_pass` for authentication. The task requires updating `stapp02` to use key-based authentication instead. `stapp01` and `stapp03` are out of scope and remain unchanged.

*Screenshot: `cat /home/thor/ansible/inventory` showing all three app server entries with `ansible_ssh_pass`*

---

### Step 8: Test Ansible Ping with Default Inventory

Attempt an Ansible ping against `stapp02` using the existing inventory configuration to observe the failure caused by the password-based entry conflicting with the now-installed key.

```bash
thor@jump-host ~$ ansible stapp02 -m ping -i /home/thor/ansible/inventory
stapp02 | UNREACHABLE! => {
    "changed": false,
    "msg": "Invalid/incorrect password: Permission denied, please try again.",
    "unreachable": true
}
```

> This failure is expected and instructive. After key installation via `ssh-copy-id`, the remote SSH daemon on `stapp02` defaults to key-based auth first. The inventory's `ansible_ssh_pass` value is now rejected because the server no longer accepts the password through the same auth flow when key auth is configured. The inventory entry must be updated.

*Screenshot: Ansible ping failure output showing `UNREACHABLE` with `Invalid/incorrect password` message*

---

### Step 9: Run Ansible Ping with Explicit Key Override

Before modifying the inventory, validate that Ansible can reach `stapp02` successfully when the correct user and private key are explicitly passed via CLI flags.

```bash
thor@jump-host ~$ ansible stapp02 -m ping -i /home/thor/ansible/inventory -u steve --private-key=/home/thor/.ssh/id_rsa
stapp02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

> This confirms that the SSH key pair is fully functional for Ansible connectivity. The explicit `-u steve` and `--private-key` flags override the inventory-level credentials and produce a successful `pong` response. This also validates that Ansible itself is configured and working correctly, isolating the issue purely to the inventory entry.

*Screenshot: Ansible ping returning `SUCCESS` with `pong` response after passing explicit user and key flags*

---

### Step 10: Update the Inventory File for Key-Based Auth

Edit the inventory file to replace the password-based entry for `stapp02` with the correct key-based authentication parameters. `stapp01` and `stapp03` entries are preserved unchanged.

```bash
thor@jump-host ~$ vi /home/thor/ansible/inventory
```

**Updated inventory content:**

```ini
stapp01 ansible_ssh_pass=Ir0nM@n
stapp02 ansible_user=steve ansible_ssh_private_key_file=/home/thor/.ssh/id_rsa
stapp03 ansible_ssh_pass=BigGr33n
```

**Changes made to `stapp02` entry:**

| Parameter Removed | Parameter Added | Reason |
|---|---|---|
| `ansible_ssh_pass=Am3ric@` | `ansible_user=steve` | Explicit remote user declaration |
| | `ansible_ssh_private_key_file=/home/thor/.ssh/id_rsa` | Points to the private key for authentication |

*Screenshot: `vi` editor showing the updated `stapp02` inventory line with `ansible_user` and `ansible_ssh_private_key_file` parameters*

---

### Step 11: Validate Final Ansible Ping

Run the Ansible ping command using only the updated inventory file, with no CLI overrides, to confirm the inventory-driven key-based authentication works end-to-end.

```bash
thor@jump-host ~$ ansible stapp02 -m ping -i /home/thor/ansible/inventory
stapp02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

> The `SUCCESS` response with `pong` confirms that passwordless SSH authentication between the Ansible controller (`jump host`) and the managed node (`stapp02`) is fully operational. Ansible can now execute playbooks against `stapp02` without any interactive credential input.

*Screenshot: Final Ansible ping returning `SUCCESS` with `pong` using only the inventory file, no CLI key flags*

---

## Errors and Resolutions

### Error 1: Ansible Ping Fails After Key Installation

**Command:**
```bash
ansible stapp02 -m ping -i /home/thor/ansible/inventory
```

**Error Output:**
```
stapp02 | UNREACHABLE! => {
    "changed": false,
    "msg": "Invalid/incorrect password: Permission denied, please try again.",
    "unreachable": true
}
```

**Root Cause:**

The inventory still contained `ansible_ssh_pass=Am3ric@` for `stapp02`. After `ssh-copy-id` installed the public key on the remote host, the SSH daemon on `stapp02` began preferring key-based authentication. Ansible attempted to authenticate using the password from the inventory, which was rejected because the authentication flow had changed on the remote side.

**Resolution:**

* First, confirmed that key-based connectivity was functional by running Ansible with explicit `-u steve --private-key` flags (Step 9), which returned `SUCCESS`.
* Then updated the `stapp02` entry in `/home/thor/ansible/inventory` to use `ansible_user=steve` and `ansible_ssh_private_key_file=/home/thor/.ssh/id_rsa`, removing the `ansible_ssh_pass` directive entirely (Step 10).
* Re-ran the ping using only the inventory, which returned `SUCCESS` (Step 11).

---

## Best Practices Applied

* **4096-bit RSA keys** were generated rather than the default 2048-bit, providing significantly higher cryptographic strength appropriate for production infrastructure.
* **Empty passphrase** was set intentionally on the key pair to support non-interactive Ansible automation. In production environments with stricter controls, an SSH agent (`ssh-agent`) would be used to hold passphrase-protected keys in memory.
* **`ssh-copy-id` used over manual key injection** to ensure correct remote permissions (`~/.ssh` at `700`, `authorized_keys` at `600`) are set automatically on the target host, preventing a common misconfiguration source.
* **Intermediate validation step** (Step 9) was performed before modifying the inventory to isolate whether the issue was key-pair connectivity or inventory configuration, enabling precise and targeted remediation.
* **Scoped inventory changes**: Only the `stapp02` entry was modified. `stapp01` and `stapp03` entries were left untouched, respecting the principle of minimum necessary change.
* **Explicit `ansible_user` declaration** was added to the inventory alongside the key path, making the remote user identity unambiguous and eliminating implicit username resolution that can fail across environments with differing default user configurations.
* **`-C` comment flag** was used during key generation to embed a human-readable identifier (`thor@jump-host`) into the public key, enabling traceability when auditing `authorized_keys` files on managed nodes.

---

## Lessons Learned

* **Inventory and SSH state must stay in sync.** After changing the SSH authentication method on a host (password to key), the corresponding Ansible inventory entry must be updated immediately. Leaving a stale `ansible_ssh_pass` directive after key installation causes confusing `Permission denied` errors that appear unrelated to the real cause.

* **Test connectivity before and after at each layer.** Testing direct SSH (`ssh steve@stapp02`) before running Ansible confirmed the key worked at the transport layer, which narrowed the Ansible failure to the inventory configuration layer rather than the key setup itself.

* **`ssh-copy-id` is the preferred key installation method over manual `authorized_keys` editing.** It handles file creation, permission setting, and duplicate-key prevention atomically, reducing the risk of introducing errors that are difficult to diagnose later.

* **The intermediate CLI-flag validation pattern is highly reusable.** Running Ansible with `--private-key` and `-u` to bypass inventory settings is a fast diagnostic technique for separating authentication problems from inventory configuration problems. This pattern applies broadly to any Ansible connectivity issue.

* **Host key verification (`StrictHostKeyChecking`) matters in automation.** When `ssh-copy-id` prompted for host key acceptance (`Are you sure you want to continue connecting?`), it was answered interactively. In fully automated pipelines, the host key should be pre-populated in `known_hosts` or `StrictHostKeyChecking=accept-new` should be configured in `ansible.cfg` to avoid blocking unattended runs.

---






<img width="506" height="406" alt="image" src="https://github.com/user-attachments/assets/b07e4328-57e7-4600-b2af-419982e1303f" />
<img width="511" height="420" alt="image" src="https://github.com/user-attachments/assets/49df5fea-adef-44e4-80aa-b543088f9001" />
<img width="511" height="428" alt="image" src="https://github.com/user-attachments/assets/a2bdac34-157c-433c-a60d-b22038e41328" />
<img width="512" height="427" alt="image" src="https://github.com/user-attachments/assets/9d3e41fe-4546-4a22-aece-9da272e697d2" />
<img width="509" height="381" alt="image" src="https://github.com/user-attachments/assets/e7047ee7-d763-44ee-a9bc-076ab355132d" />
<img width="512" height="425" alt="image" src="https://github.com/user-attachments/assets/9f8faa0a-2043-4933-9fab-792ddeb73294" />
<img width="511" height="290" alt="image" src="https://github.com/user-attachments/assets/250986d0-a443-4096-be6d-40d2bcd65f2e" />
<img width="511" height="425" alt="image" src="https://github.com/user-attachments/assets/17e86cca-215e-4464-83f0-b3f1faf1a74f" />
<img width="509" height="428" alt="image" src="https://github.com/user-attachments/assets/7ae379df-0b57-42ff-a3ba-2e99b82d323b" />
<img width="512" height="304" alt="image" src="https://github.com/user-attachments/assets/e5d8f2db-be58-494b-98d6-eb789e1d0321" />
<img width="511" height="349" alt="image" src="https://github.com/user-attachments/assets/d19980f2-1d56-4647-a769-3b504529ec40" />
<img width="509" height="387" alt="image" src="https://github.com/user-attachments/assets/8e1b8e76-d4b2-4f49-82a1-495f6e60a6c7" />



