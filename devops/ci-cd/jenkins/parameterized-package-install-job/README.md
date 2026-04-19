# Jenkins Parameterized Job: Remote Package Installation via SSH on Storage Server

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Pre-flight Connectivity and Credential Verification via CLI SSH](#step-1-pre-flight-connectivity-and-credential-verification-via-cli-ssh)
  - [Step 2: Access the Jenkins UI](#step-2-access-the-jenkins-ui)
  - [Step 3: Check and Install Pending Plugin Updates](#step-3-check-and-install-pending-plugin-updates)
  - [Step 4: Install the Publish Over SSH Plugin](#step-4-install-the-publish-over-ssh-plugin)
  - [Step 5: Restart Jenkins After Plugin Installation](#step-5-restart-jenkins-after-plugin-installation)
  - [Step 6: Verify SSH Plugin Installation](#step-6-verify-ssh-plugin-installation)
  - [Step 7: Configure the SSH Server in Jenkins System Settings](#step-7-configure-the-ssh-server-in-jenkins-system-settings)
  - [Step 8: Create the Parameterized Jenkins Job](#step-8-create-the-parameterized-jenkins-job)
  - [Step 9: Configure the String Parameter](#step-9-configure-the-string-parameter)
  - [Step 10: Configure the Build Step to Execute the Remote Install Command](#step-10-configure-the-build-step-to-execute-the-remote-install-command)
  - [Step 11: Trigger the First Build with PACKAGE=vim-enhanced](#step-11-trigger-the-first-build-with-packagevim-enhanced)
  - [Step 12: Verify Build Success in Console Output](#step-12-verify-build-success-in-console-output)
  - [Step 13: Verify Package Installation on the Storage Server](#step-13-verify-package-installation-on-the-storage-server)
  - [Step 14: Run Repeat Builds to Confirm Reliability](#step-14-run-repeat-builds-to-confirm-reliability)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)

---

## Overview

This project documents the end-to-end configuration of a Jenkins parameterized freestyle job named `install-packages` that remotely installs system packages on the Stratos Datacenter storage server (`ststor01`) via SSH. The job accepts a `PACKAGE` string parameter at build time, enabling operators to install any named package on the storage server without requiring direct SSH access or manual intervention.

| Attribute | Value |
|---|---|
| Jenkins Job Name | `install-packages` |
| Job Type | Freestyle Project |
| Target Server | `ststor01` (Storage Server, Stratos Datacenter) |
| Remote User | `natasha` |
| Plugin Required | Publish Over SSH |
| Parameter Name | `PACKAGE` |
| Test Package | `vim-enhanced` |
| Jenkins Version | 2.541.2 |

---

## Problem Statement

The Nautilus DevOps team provisioned a new Jenkins server and required an automated mechanism to install and configure packages on the Stratos Datacenter storage server without direct manual SSH access. The solution needed to be parameterized so that any package name could be supplied at runtime, making the job reusable across operational scenarios.

The specific requirements were:

* Create a Jenkins job named `install-packages`
* Add a string parameter named `PACKAGE`
* Configure the job to install the package specified by `$PACKAGE` on `ststor01` via SSH
* Validate the solution by running the job at least once using `PACKAGE=vim-enhanced`
* Ensure the job runs reliably on repeated executions

---

## Solution Architecture

The solution uses Jenkins' **Publish Over SSH** plugin to establish an authenticated SSH connection from the Jenkins controller to `ststor01`. At build time, Jenkins substitutes the `$PACKAGE` environment variable into the remote exec command `sudo yum install -y $PACKAGE`, which runs on the storage server under the `natasha` user account. The SSH server configuration is stored in Jenkins System settings and referenced by name within the job's build step, keeping credentials centralized and the job configuration portable.

```
Jenkins Controller
      |
      |  SSH (Publish Over SSH Plugin)
      |  Exec: sudo yum install -y $PACKAGE
      v
ststor01 (Storage Server)
  natasha@ststor01
```

---

## Prerequisites

* Jenkins is running and accessible via the web UI
* The `natasha` user account exists on `ststor01` and has `sudo` privileges for `yum`
* Network connectivity exists between the Jenkins controller and `ststor01`
* Jenkins admin credentials: username `admin`, password `Adm!n321`
* SSH credentials for `natasha` on `ststor01`

---

## Implementation Guide

### Step 1: Pre-flight Connectivity and Credential Verification via CLI SSH

Before configuring Jenkins, connectivity to `ststor01` and the `natasha` user credentials were verified manually from the jump host. This step also served as an early confirmation that `sudo yum` was functional on the storage server and that the target package was installable.

From `thor@jumphost`, SSH into `ststor01` for the first time:

```bash
thor@jumphost ~$ ssh natasha@ststor01
```

Because this was the first connection to `ststor01`, the host key was not yet in the known hosts file. The authenticity prompt was reviewed and accepted:

```
The authenticity of host 'ststor01 (10.244.234.222)' can't be established.
ED25519 key fingerprint is SHA256:yEyN8qvzhNxfcKVE+H05zwQPmQMKCXj4JyGWuOP1HIg.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'ststor01' (ED25519) to the list of known hosts.
natasha@ststor01's password:
```

Once authenticated, a manual install of `vim-enhanced` was run directly on the storage server to confirm package manager availability and internet/repository connectivity:

```bash
[natasha@ststor01 ~]$ sudo yum install -y vim-enhanced
```

The following repositories were contacted and resolved dependencies successfully:

| Repository | Speed | Size |
|---|---|---|
| CentOS Stream 9 - BaseOS | 9.0 MB/s | 8.9 MB |
| CentOS Stream 9 - AppStream | 13 MB/s | 27 MB |
| CentOS Stream 9 - Extras packages | 13 kB/s | 21 kB |
| Docker CE Stable - x86_64 | 1.3 MB/s | 74 kB |
| Extra Packages for Enterprise Linux 9 - x86_64 | 55 MB/s | 21 MB |
| Extra Packages for Enterprise Linux 9 openh264 (From Cisco) | 1.8 MB/s | 2.5 kB |
| Extra Packages for Enterprise Linux 9 - Next | 320 kB/s | 260 kB |

The transaction resolved and installed 4 packages:

| Package | Architecture | Version | Repository | Size |
|---|---|---|---|---|
| vim-enhanced | x86_64 | 2:8.2.2637-27.el9 | appstream | 1.7 M |
| gpm-libs (dep) | x86_64 | 1.20.7-29.el9 | appstream | 21 k |
| vim-common (dep) | x86_64 | 2:8.2.2637-27.el9 | appstream | 7.0 M |
| vim-filesystem (dep) | noarch | 2:8.2.2637-27.el9 | baseos | 13 k |

Total download size: 8.8 M | Installed size: 34 M

The packages were downloaded at a combined rate of 13 MB/s and the transaction check succeeded, confirming the storage server environment was healthy before any Jenkins automation was introduced.

> Screenshots: Terminal output from thor@jumphost showing the initial SSH to ststor01, host key acceptance, sudo yum install -y vim-enhanced execution, repository metadata downloads, and the 4-package transaction summary including vim-enhanced, gpm-libs, vim-common, and vim-filesystem

<img width="1031" height="840" alt="image" src="https://github.com/user-attachments/assets/944422dd-fb97-4036-bf3e-2b192e0cc8c0" />
<img width="1030" height="733" alt="image" src="https://github.com/user-attachments/assets/9781d134-dd06-4c5d-8706-9f835f25748a" />

---

### Step 2: Access the Jenkins UI

Navigate to the Jenkins web interface using the **Jenkins** button in the top bar of the KodeKloud lab environment. Log in using the following credentials:

* **Username:** `admin`
* **Password:** `Adm!n321`

> Screenshot: Jenkins login page with username "admin" entered

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/a95a2ebc-3bd5-4e8b-aa69-2c03bcbebfd3" />

---

### Step 3: Check and Install Pending Plugin Updates

Before installing new plugins, apply any available plugin updates to keep the Jenkins environment current.

1. Go to **Manage Jenkins** > **Plugins** > **Updates**
2. Select the available update (the **bouncycastle API** plugin update was pending at health score 100)
3. Click **Update** to apply

> Screenshots: Jenkins Plugin Manager showing the bouncycastle API update available with health score 100

<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/304b1ebf-c6c6-4045-8871-ec8fd032695f" />
<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/9a597e85-726a-48ca-a1e1-6dc39b689d71" />
<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/67e25e1a-017d-4cb6-bf90-aec1b8012a81" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/1652fe84-61e4-4cf1-acc5-20fe7c67f204" />

Applying pending updates before adding new plugins prevents dependency conflicts and ensures the plugin subsystem is in a stable state prior to new installations.

---

### Step 4: Install the Publish Over SSH Plugin

The **Publish Over SSH** plugin is required to enable Jenkins to execute remote commands on `ststor01` over SSH.

1. Navigate to **Manage Jenkins** > **Plugins** > **Available plugins**
2. Search for **Publish Over SSH**
3. Check the checkbox next to **Publish Over SSH** (version `390.vb_f56e7405751`)
4. Click **Install**

> Screenshot: Available plugins tab showing "Publish Over SSH" selected with health score 96


<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/9fb617a7-68bf-4719-a29e-3ffd01347381" />

Jenkins will display a **Download progress** page. All dependency plugins must complete with a **Success** status before proceeding.

The dependency chain installed includes: JAXB, JSON Api, Jackson Annotations 2 API, Jakarta Activation API, SnakeYAML API, Jakarta XML Binding API, Woodstox Core API, Jackson 2 API, Infrastructure plugin for Publish Over X, Structs, EDDSA API, Gson API, Trilead API, Variant, commons-lang3 v3.x Jenkins API, Ionicons API, commons-text API, and others.

> Screenshots: Plugin download progress page showing all dependencies resolved with Success status

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/7b0cdc76-9556-497b-9947-da454d8f20f0" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/17d6884e-844d-421d-a0c8-213b9c26dc6a" />

---

### Step 5: Restart Jenkins After Plugin Installation

Once all plugin downloads complete, restart Jenkins to activate the newly installed plugins.

On the download progress page, select **Restart Jenkins when installation is complete and no jobs are running**.

Jenkins displays a restart screen confirming the service is restarting. The browser reloads automatically when Jenkins is ready.

> Screenshot: Jenkins restarting screen with spinner and "Your browser will reload automatically when Jenkins is ready" message


<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/30593a7f-3eb9-481d-934e-af94b19df8c2" />


---

### Step 6: Verify SSH Plugin Installation

After Jenkins restarts, confirm the Publish Over SSH plugin is active.

1. Navigate to **Manage Jenkins** > **Plugins** > **Installed plugins**
2. Search for `ssh`
3. Confirm **Publish Over SSH** is present and **Enabled** (blue toggle active)

The installed plugin search returns three SSH-related plugins:

| Plugin | Version | Enabled |
|---|---|---|
| JSch dependency plugin | 0.2.16-95.v3eecb_55fa_b_78 | Yes (greyed, dependency) |
| Publish Over SSH | 390.vb_f56e7405751 | **Yes (blue, active)** |
| SSH Credentials Plugin | 372.va_250881b_08cd | Yes (greyed, dependency) |

> Screenshot: Installed plugins tab filtered by "ssh" showing Publish Over SSH enabled

<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/90906356-9691-426d-8686-2bc6bffe069b" />

---

### Step 7: Configure the SSH Server in Jenkins System Settings

Register `ststor01` as a known SSH server so Jenkins can reference it by name in job build steps.

1. Navigate to **Manage Jenkins** > **System**
2. Scroll down to the **Publish over SSH** section
3. Under **SSH Servers**, click **Add**
4. Enter the following values:

| Field | Value |
|---|---|
| Name | `ststor01` |
| Hostname | `ststor01` |
| Username | `natasha` |
| Remote Directory | `/tmp` |

5. Expand **Advanced**
6. Check **Use password authentication, or use a different key**
7. Enter the SSH password for `natasha` in the **Passphrase / Password** field
8. Click **Save**

> Screenshot: Jenkins System configuration page showing SSH Server entry for ststor01 with natasha credentials and /tmp as remote directory

<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/f6b8dab4-a6f8-488e-91fe-6bc87a478798" />

---

### Step 8: Create the Parameterized Jenkins Job

Create a new freestyle job named `install-packages`.

1. From the Jenkins dashboard, click **New Item**
2. Enter the item name: `install-packages`
3. Select **Freestyle project**
4. Click **OK**

> Screenshot: New Item page with "install-packages" entered and Freestyle project selected

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/aa86d3b2-475d-45c7-8619-b517abe222fc" />

---

### Step 9: Configure the String Parameter

Enable build parameterization and add the `PACKAGE` string parameter.

1. In the job configuration page, check **This project is parameterized**
2. Click **Add Parameter** > **String Parameter**
3. Enter the following values:

| Field | Value |
|---|---|
| Name | `PACKAGE` |
| Default Value | *(leave blank)* |
| Description | `Package name to install on the storage server` |

4. Click **Apply**

> Screenshot: Job configuration showing "This project is parameterized" checked and PACKAGE String Parameter with description filled in

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/d8a56f68-697a-460c-af63-6a45721a4865" />

---

### Step 10: Configure the Build Step to Execute the Remote Install Command

Add a build step that sends the install command to `ststor01` over SSH.

1. Scroll down to **Build Steps**
2. Click **Add build step** > **Send files or execute commands over SSH**
3. Under **SSH Publishers** > **SSH Server**, select **Name**: `ststor01` from the dropdown
4. Under **Transfers** > **Transfer Set**, leave **Source files** blank (no files are being transferred)
5. In the **Exec command** field, enter:

```bash
sudo yum install -y $PACKAGE
```

6. Click **Save**

> Screenshots: Build Steps configuration showing "Send files or execute commands over SSH" step with ststor01 selected and "sudo yum install -y $PACKAGE" in the Exec command field

<img width="1918" height="1023" alt="image" src="https://github.com/user-attachments/assets/1fbc1e55-024b-4ee3-be6a-7272bed56bf6" />
<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/fdb4793d-a3b5-4a30-80cc-4592c34b5a2c" />

The `$PACKAGE` variable is automatically substituted by Jenkins with the value provided at build time. The Publish Over SSH plugin supports full Jenkins environment variable substitution in all transfer fields.

---

### Step 11: Trigger the First Build with PACKAGE=vim-enhanced

Run the job for the first time to validate the full pipeline.

1. From the `install-packages` job page, click **Build with Parameters**
2. In the **PACKAGE** field, enter: `vim-enhanced`
3. Click **Build**

> Screenshot: Build with Parameters page showing PACKAGE field populated with "vim-enhanced" and the Build button

<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/b93582e2-6061-48bd-bfc1-7495c4ed0c62" />

---

### Step 12: Verify Build Success in Console Output

After the build completes, inspect the console output to confirm the SSH execution succeeded.

Navigate to **install-packages** > **#1** > **Console Output**.

The expected output confirms a successful build:

```
Started by user admin
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/install-packages
SSH: Connecting from host [jenkins.stratos.xfusioncorp.com]
SSH: Connecting with configuration [ststor01] ...
SSH: EXEC: completed after 1,201 ms
SSH: Disconnecting configuration [ststor01] ...
SSH: Transferred 0 file(s)
Build step 'Send files or execute commands over SSH' changed build result to SUCCESS
Finished: SUCCESS
```

> Screenshot: Console Output for build #1 showing SSH connection to ststor01 and Finished: SUCCESS

The job status page also shows build **#1** completed successfully at 7:18 PM with a green checkmark.

> Screenshot: install-packages job status page showing build #1 with green success indicator

---

### Step 13: Verify Package Installation on the Storage Server

Confirm that `vim-enhanced` was installed on `ststor01` by SSHing into the storage server from the jump host and querying the RPM database.

**First verification session:**

```bash
thor@jumphost ~$ ssh natasha@ststor01
natasha@ststor01's password:
Last login: Sat Apr 18 19:01:33 2026 from 10.244.49.16
[natasha@ststor01 ~]$ rpm -q vim-enhanced
vim-enhanced-8.2.2637-27.el9.x86_64
[natasha@ststor01 ~]$ exit
logout
Connection to ststor01 closed.
```

> Screenshot: Terminal showing rpm -q vim-enhanced returning vim-enhanced-8.2.2637-27.el9.x86_64 after first build

The installed package version is `vim-enhanced-8.2.2637-27.el9.x86_64`, confirming the Jenkins job successfully executed `sudo yum install -y vim-enhanced` on the storage server.

The full install output captured during an earlier direct SSH session showed the following packages installed as part of the transaction:

| Package | Version | Repository | Size |
|---|---|---|---|
| vim-enhanced | 2:8.2.2637-27.el9 | appstream | 1.7 M |
| gpm-libs | 1.20.7-29.el9 | appstream | 21 k |
| vim-common | 2:8.2.2637-27.el9 | appstream | 7.0 M |
| vim-filesystem | 2:8.2.2637-27.el9 | baseos | 13 k |

> Screenshot: Terminal showing full yum install output for vim-enhanced with all 4 packages installed successfully

---

### Step 14: Run Repeat Builds to Confirm Reliability

To ensure the job behaves reliably on subsequent executions (idempotent behavior), the build was triggered additional times. `yum install` with the `-y` flag on an already-installed package exits cleanly without error, making the job safe to re-run.

**Second and third verification sessions on `ststor01`:**

```bash
thor@jumphost ~$ ssh natasha@ststor01
natasha@ststor01's password:
Last login: Sat Apr 18 19:21:30 2026 from 10.244.49.16
[natasha@ststor01 ~]$ rpm -q vim-enhanced
vim-enhanced-8.2.2637-27.el9.x86_64
[natasha@ststor01 ~]$ exit
logout
Connection to ststor01 closed.
```

> Screenshot: Terminal showing repeated ssh and rpm -q vim-enhanced verification across two sessions, both confirming package is installed

The package remained installed and the query returned the same version string across all verification sessions, confirming job reliability and idempotent execution.

---

## Errors and Resolutions

### Error: "Either Source files, Exec command or both must be supplied"

**Context:** When the Build Step was saved with the SSH transfer set left entirely blank (no source files and no exec command), Jenkins displayed a validation error on the build step.

**Root Cause:** The Publish Over SSH plugin requires at least one of `Source files` or `Exec command` to be populated. An empty transfer set is invalid.

**Resolution:** The **Exec command** field was populated with `sudo yum install -y $PACKAGE`. The **Source files** field was intentionally left blank because no files needed to be transferred; only a remote command execution was required.

> Screenshot: Build Steps page showing the validation error "Either Source files, Exec command or both must be supplied" before the exec command was entered

---

## Best Practices

* **Centralize SSH credentials in Jenkins System configuration** rather than embedding them per-job. The SSH server registered under `ststor01` in Manage Jenkins > System is reusable across multiple jobs, reducing credential sprawl.

* **Use parameterized builds for operational flexibility.** A single `install-packages` job handles any package by varying the `PACKAGE` parameter, eliminating the need to create separate jobs per package.

* **Leave Source files blank when only remote execution is needed.** The Publish Over SSH plugin supports exec-only build steps. Do not create dummy files simply to satisfy a source field.

* **Apply plugin updates before installing new plugins.** Updating existing plugins first ensures dependency libraries are at their latest compatible versions, reducing the risk of version conflicts with newly installed plugins.

* **Use `Restart Jenkins when installation is complete and no jobs are running`** rather than restarting manually mid-installation. This option waits for the download pipeline to complete before initiating restart, preventing partial installations.

* **Verify plugin installation after restart** by checking the Installed plugins tab filtered by a keyword. Do not assume a plugin is active without confirming its Enabled toggle is blue.

* **Validate installations via RPM query on the target server** (`rpm -q <package>`), not just by inspecting Jenkins console output. Console output confirms the SSH exec completed successfully, while the RPM query confirms the actual system state.

* **Run builds multiple times to confirm idempotency.** `yum install -y` is idempotent for already-installed packages, which makes the job safe for repeated triggering in operational and automation scenarios.

---

## Lessons Learned

* **The Publish Over SSH plugin installs a substantial dependency chain.** The installation triggered downloads for over a dozen library plugins (Jackson, JAXB, SnakeYAML, Woodstox, EDDSA, Gson, Trilead, and others). This is expected behavior for SSH-capable plugins in Jenkins and is not a cause for concern. Always review the Download progress page and confirm all entries show Success before proceeding.

* **SSH server names in Jenkins are resolved internally, not via DNS in all cases.** The hostname `ststor01` was entered as both the Name and Hostname fields, and Jenkins resolved it correctly within the Stratos Datacenter network. In environments where short hostnames are not resolvable, use the fully qualified domain name or IP address in the Hostname field while keeping the Name field as a human-readable label.

* **Jenkins environment variable substitution in the SSH exec command is reliable but case-sensitive.** The parameter was defined as `PACKAGE` (uppercase) and referenced as `$PACKAGE` in the exec command. Any mismatch in casing between the parameter name and the variable reference will result in an empty substitution and a failed or no-op yum command.

* **The Remote Directory field in the SSH server configuration (`/tmp`) does not affect exec-only build steps.** The remote directory is only relevant when transferring files. For jobs that only run remote commands, this field can be set to any valid path the remote user can access without impacting execution.

* **Build reliability should always be tested across multiple runs, not just validated on the first success.** The first build succeeds under ideal conditions. Subsequent builds expose edge cases such as idempotency behavior, connection timeouts, or lock contention on the package manager. Running the job two or three times before marking it production-ready is a sound operational standard.

* **Pre-verifying connectivity to the storage server before configuring Jenkins** was executed as the deliberate first step of this implementation, not as an afterthought. SSHing manually from the jump host as `natasha` and running `sudo yum install -y vim-enhanced` directly confirmed that credentials, sudo privileges, repository access, and internet connectivity were all functional before Jenkins was introduced as a variable. This separation of concerns makes troubleshooting significantly faster: if the Jenkins job had failed, the pre-flight evidence would have immediately ruled out network, credential, or package manager issues as the cause.



















<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/be5aaaea-a1f4-4d72-961c-94251a324c15" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/981fe58c-21d5-46b6-ba8e-912aa76312b9" />
<img width="1034" height="711" alt="image" src="https://github.com/user-attachments/assets/82dec9a9-1a89-402b-848e-ac7203d281c4" />
<img width="1032" height="611" alt="image" src="https://github.com/user-attachments/assets/02ea54e0-e28f-4bfc-84ff-a0e15b4c783d" />
<img width="1036" height="331" alt="image" src="https://github.com/user-attachments/assets/eefd2a26-7ace-4d79-9dc0-9f788e953d52" />
<img width="1037" height="387" alt="image" src="https://github.com/user-attachments/assets/5924b5f0-24db-4130-a0b8-976a587a45cd" />
<img width="1919" height="1022" alt="image" src="https://github.com/user-attachments/assets/b19d62d1-8ac1-4e35-8a9d-43b5c6441292" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/fccfc2fa-1009-4d95-b5d6-d742bf19b91c" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/4a7ab027-8011-43aa-b520-30007eaa376c" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/ed70d6af-87d6-4ee6-9c18-2f9514c23bee" />
<img width="1032" height="447" alt="image" src="https://github.com/user-attachments/assets/ef61a6b1-f8ea-4d92-8f21-3d695a4dc0ba" />
<img width="1035" height="487" alt="image" src="https://github.com/user-attachments/assets/b241da07-395d-4ab3-ab4c-0d5a6db00dea" />
<img width="1036" height="542" alt="image" src="https://github.com/user-attachments/assets/d0cb6b80-45e3-4e02-9a8b-789c6d5b6b3f" />




