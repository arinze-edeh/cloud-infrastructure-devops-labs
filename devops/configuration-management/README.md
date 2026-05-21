# Configuration Management

Ansible-based automation covering infrastructure provisioning, package management, access control, service deployment, and SSH authentication across the Stratos DC multi-node environment. All tasks execute from a centralized jump host against a fleet of RHEL-based application servers (`stapp01`, `stapp02`, `stapp03`) with zero manual intervention on target nodes.

---

## Directory Structure

```
configuration-management/
├── ansible-acl-file-provisioning/
├── ansible-app-server-file-distribution/
├── ansible-controller-setup/
├── ansible-httpd-deployment/
├── ansible-httpd-role-jinja2-template/
├── ansible-httpd-service-provisioning/
├── ansible-httpd-web-server-provisioning/
├── ansible-playbook-file-provisioning/
├── ansible-remote-file-management/
├── ansible-samba-package-install/
├── ansible-ssh-passwordless-setup/
├── ansible-static-inventory-setup/
└── README.md
```

---

## Project Summaries

### [ansible-acl-file-provisioning](./ansible-acl-file-provisioning)

**Quick Summary:** Creates files on three app servers and enforces POSIX ACL permissions scoped per host using the Ansible `acl` module, with no changes to standard Unix ownership.

| | |
|---|---|
| **Purpose** | Automate filesystem provisioning and per-host access control enforcement across `stapp01`, `stapp02`, and `stapp03`. |
| **Approach** | Three independent plays target one host each. The `file` module creates root-owned files; the `acl` module applies named user or group ACL entries (`r` or `rw`) using explicit `etype` declarations to avoid ambiguous entity resolution. |
| **Outcome** | All files confirmed via `getfacl` with correct ACL masks and ownership. Fully idempotent across re-runs. |

---

### [ansible-app-server-file-distribution](./ansible-app-server-file-distribution)

**Quick Summary:** Distributes a static HTML file from the jump host to all three app servers using the `copy` module, with post-run verification via ad-hoc shell commands.

| | |
|---|---|
| **Purpose** | Replicate `/usr/src/itadmin/index.html` to `/opt/itadmin/index.html` across the fleet in a single playbook run. |
| **Approach** | A pre-flight `ping` validates inventory connectivity and Python discovery before execution. The `file` module ensures the destination directory exists; `copy` handles file transfer. `become: true` enforces root ownership on all delivered files. |
| **Outcome** | File confirmed on all three nodes with consistent 35-byte size, `root:root` ownership, and identical delivery timestamps. |

---

### [ansible-controller-setup](./ansible-controller-setup)

**Quick Summary:** Provisions the jump host as an Ansible controller by installing Ansible 4.7.0 via `pip3`, resolving a `sudo` PATH conflict introduced by the pip upgrade process.

| | |
|---|---|
| **Purpose** | Establish a version-pinned, globally accessible Ansible installation on the Stratos DC jump host. |
| **Approach** | After upgrading pip to 26.0.1, the new binary landed in `/usr/local/bin`, which is excluded from `sudo`'s `secure_path` on RHEL. Resolved by invoking pip via its absolute path. Binary permissions and `ansible --version` output were verified post-install. |
| **Outcome** | Ansible 4.7.0 with `ansible-core 2.11.12` installed system-wide, executable by all users, with `libyaml = True` confirmed. |

---

### [ansible-httpd-deployment](./ansible-httpd-deployment)

**Quick Summary:** Installs and configures Apache HTTPD across all app servers, deploys a custom `index.html` via the `blockinfile` module, and enforces `apache:apache` ownership at mode `0655`.

| | |
|---|---|
| **Purpose** | Full service lifecycle provisioning: package install, service management, file creation, and content injection. |
| **Approach** | Five ordered tasks separate concerns cleanly. `file: state: touch` creates the file with ownership set. `blockinfile` injects a managed content block using default markers, preserving idempotency on re-runs. A final `file` task re-enforces permissions. `ansible_become_pass` was added to the inventory after identifying it was absent from the initial handoff. |
| **Outcome** | `ok=6 changed=4 failed=0` across all three nodes. Service confirmed `active` and `enabled`. File content, ownership, and permissions verified per node via `sshpass`. |

---

### [ansible-httpd-role-jinja2-template](./ansible-httpd-role-jinja2-template)

**Quick Summary:** Extends an existing Ansible role to deploy a dynamically rendered `index.html` to `stapp03` using Jinja2 templating with `{{ inventory_hostname }}` variable substitution.

| | |
|---|---|
| **Purpose** | Demonstrate role-based automation with dynamic content generation, targeting a single host from a multi-host inventory. |
| **Approach** | The `templates/` directory was absent from the existing role skeleton and created manually before adding the `template` module task. The playbook's empty `hosts:` field was corrected to `stapp03`. A single-quoted heredoc delimiter protected Jinja2 syntax during file writes. The task was appended to `tasks/main.yml` using `>>` to preserve existing tasks intact. |
| **Outcome** | `{{ inventory_hostname }}` resolved correctly to `stapp03` in the rendered file. Ownership (`banner:banner`) and mode (`0744`) confirmed via direct SSH inspection. |

---

### [ansible-httpd-service-provisioning](./ansible-httpd-service-provisioning)

**Quick Summary:** Installs `httpd` and enforces service state across all app servers using the distribution-agnostic `package` module, with combined start and boot-persistence management in a single task.

| | |
|---|---|
| **Purpose** | Idempotent HTTPD provisioning executable with no arguments beyond `ansible-playbook -i inventory playbook.yml`. |
| **Approach** | `package` replaces `yum` for OS portability. `service` sets both `state: started` and `enabled: yes` atomically. `ansible_become_pass` is scoped per host in the inventory to enable fully unattended execution. |
| **Outcome** | `ok=3 changed=2 failed=0` confirmed across all nodes. Re-run produces `changed=0`, validating idempotency. |

---

### [ansible-httpd-web-server-provisioning](./ansible-httpd-web-server-provisioning)

**Quick Summary:** Provisions HTTPD and deploys a two-line `index.html` using `copy` for initial content and `lineinfile` with `insertbefore: BOF` to prepend a welcome message at the top of the file.

| | |
|---|---|
| **Purpose** | Demonstrate ordered, multi-module file construction with precise content placement and enforced file metadata. |
| **Approach** | Task ordering is critical: `copy` establishes the file before `lineinfile` operates on it. `insertbefore: BOF` is the explicit, position-safe way to prepend content without relying on line numbers. Mode `"0655"` is quoted as a string to prevent YAML octal misinterpretation. |
| **Outcome** | Two-line file confirmed in correct order on `stapp01` with `apache:apache` ownership, `active` service state, and `-rw-r-xr-x` permissions. |

---

### [ansible-playbook-file-provisioning](./ansible-playbook-file-provisioning)

**Quick Summary:** Creates `/usr/src/appdata.txt` across all app servers with mode `0655` and per-host ownership resolved dynamically using `{{ ansible_user }}`.

| | |
|---|---|
| **Purpose** | Provision a managed file with correct per-host ownership without hard-coding values or using conditional branching. |
| **Approach** | `{{ ansible_user }}` is already defined per host in the inventory, so it resolves at runtime to the correct owner and group on each node. The gated workflow runs connectivity validation, syntax check, and `--check` dry-run before live execution. |
| **Outcome** | `tony`, `steve`, and `banner` ownership confirmed via direct SSH `ls -l` on each node. Permissions `-rw-r-xr-x` match the `0655` specification. |

---

### [ansible-remote-file-management](./ansible-remote-file-management)

**Quick Summary:** Configures a static inventory and authors a targeted playbook to create `/tmp/file.txt` on `stapp03`, resolving a team handoff with a placeholder password and missing group definition.

| | |
|---|---|
| **Purpose** | Correct an inherited incomplete inventory and establish group-based host targeting for reliable playbook execution. |
| **Approach** | The original inventory contained a literal `$pwd` placeholder and no group header. Both were corrected: the `[stapp03]` group was added and the password replaced. The playbook targets the group by name. A recurring `Found both group and host with same name` warning is noted as non-fatal and cosmetic. |
| **Outcome** | `changed=1 failed=0` on first run. File presence on `stapp03` confirmed via play recap. |

---

### [ansible-samba-package-install](./ansible-samba-package-install)

**Quick Summary:** Installs `logrotate` across all app servers using a structured execution workflow: manual SSH validation, ad-hoc ping, syntax check, and playbook execution.

| | |
|---|---|
| **Purpose** | Automate package delivery fleet-wide with a repeatable pre-execution validation pipeline. |
| **Approach** | Manual SSH connectivity is confirmed before Ansible is involved, isolating potential credential issues from automation errors. `state: present` over `state: latest` prevents unintended upgrades on re-runs. |
| **Outcome** | `changed=1 failed=0` on first run. `changed=0` on re-run confirms idempotency. |

---

### [ansible-ssh-passwordless-setup](./ansible-ssh-passwordless-setup)

**Quick Summary:** Establishes RSA key-based passwordless SSH between the jump host and `stapp02`, updates the Ansible inventory to use key authentication, and validates end-to-end Ansible connectivity without CLI overrides.

| | |
|---|---|
| **Purpose** | Replace password-based authentication with key-based auth for `stapp02` as a prerequisite for non-interactive playbook execution. |
| **Approach** | A 4096-bit RSA key is generated with an empty passphrase for automation compatibility. After `ssh-copy-id`, the inventory entry for `stapp02` is updated to remove `ansible_ssh_pass` and add `ansible_ssh_private_key_file`. An intermediate CLI-flag validation step (`--private-key`) isolates key functionality from inventory configuration before the final inventory-driven ping. |
| **Outcome** | Passwordless SSH confirmed. Final `ansible stapp02 -m ping` returns `SUCCESS` using inventory only, with no CLI overrides. `stapp01` and `stapp03` inventory entries unchanged. |

---

### [ansible-static-inventory-setup](./ansible-static-inventory-setup)

**Quick Summary:** Creates a compliant INI inventory file for `stapp02` from scratch, enabling an existing HTTPD playbook to execute successfully against App Server 2 without playbook modification.

| | |
|---|---|
| **Purpose** | Bridge a pre-existing playbook to a target host by constructing the missing inventory layer with all required connection and escalation variables. |
| **Approach** | The playbook's `hosts: all` and `become: yes` directives dictated the required inventory variables. `ansible_become` and `ansible_become_pass` were included alongside SSH credentials. The heredoc delimiter was single-quoted to prevent shell expansion of special characters in the password. |
| **Outcome** | `ok=3 changed=2 failed=0` on first run. `changed=0` on re-run confirms idempotency. No playbook changes were required. |

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Configuration Management | Ansible Core 2.11.x, 2.14.x |
| Templating | Jinja2 |
| Target OS | RHEL / CentOS (yum-based) |
| Modules Used | `file`, `copy`, `template`, `acl`, `yum`, `package`, `service`, `blockinfile`, `lineinfile`, `ping`, `shell` |
| Access Control | POSIX ACLs (`setfacl` / `getfacl`), SSH key-based authentication |
| Scripting | Bash heredocs, `ssh-keygen`, `ssh-copy-id`, `sshpass` |
| Package Management | pip3, yum |
| Infrastructure | Jump host controller model, multi-node SSH-based orchestration |

---

## Key Skills Demonstrated

- Ansible inventory design: INI format, per-host variables, group targeting, key-based vs. password-based auth
- Idempotent playbook construction with verifiable state outcomes
- POSIX ACL management via the `acl` module with explicit entity typing
- Role-based automation with Jinja2 dynamic templating
- Multi-module file construction with enforced ordering (`copy` before `lineinfile`)
- Privilege escalation scoped at play level with `ansible_become_pass` in inventory
- Pre-execution validation pipelines: ping, syntax check, check mode, then live run
- Diagnosing and resolving real execution failures: PATH conflicts, missing `become` credentials, stale inventory entries, empty `hosts` fields, placeholder passwords
- Post-execution verification via `getfacl`, `ls -l`, `systemctl`, and direct SSH inspection

---

## How to Navigate

Each subdirectory is a self-contained project with its own `README.md` covering:

- **Purpose and context** for the automation task
- **Inventory and playbook files** with annotated design decisions
- **Execution output** with play recap interpretation
- **Errors encountered and resolutions** where applicable
- **Best practices and lessons learned**

All tasks were executed on a RHEL-based jump host with Ansible Core 2.14.x and Python 3.9.

---

*Arinze Edeh | Cloud and DevOps Engineer*

