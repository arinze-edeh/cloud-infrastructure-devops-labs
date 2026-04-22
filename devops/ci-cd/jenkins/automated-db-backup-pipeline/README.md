# Jenkins Automated Database Backup Pipeline

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Infrastructure Components](#infrastructure-components)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Log In to Jenkins](#step-1-log-in-to-jenkins)
  - [Step 2: Update Existing Jenkins Plugins](#step-2-update-existing-jenkins-plugins)
  - [Step 3: Install the Publish Over SSH Plugin](#step-3-install-the-publish-over-ssh-plugin)
  - [Step 4: Establish SSH Access from Jenkins to the App Server](#step-4-establish-ssh-access-from-jenkins-to-the-app-server)
  - [Step 5: Establish SSH Trust from the App Server to the Storage Server](#step-5-establish-ssh-trust-from-the-app-server-to-the-storage-server)
  - [Step 6: Verify the Full SSH Chain](#step-6-verify-the-full-ssh-chain)
  - [Step 7: Create the Jenkins Freestyle Job](#step-7-create-the-jenkins-freestyle-job)
  - [Step 8: Configure the Build Trigger](#step-8-configure-the-build-trigger)
  - [Step 9: Write the Build Shell Script](#step-9-write-the-build-shell-script)
  - [Step 10: Save and Execute the First Build](#step-10-save-and-execute-the-first-build)
  - [Step 11: Verify the Backup File on the Storage Server](#step-11-verify-the-backup-file-on-the-storage-server)
- [Shell Script Reference](#shell-script-reference)
- [Best Practices Applied](#best-practices-applied)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Lessons Learned](#lessons-learned)

---

## Overview

This project implements a fully automated database backup pipeline using Jenkins. The pipeline connects a Jenkins controller to an application server (stapp01) via SSH, runs a `mysqldump` against a target MySQL database, and securely transfers the resulting dump file to a dedicated storage server (ststor01) using a chained SSH hop pattern. The job is scheduled to run every 10 minutes using a cron-style trigger.

---

## Problem Statement

The operations team required a repeatable, automated solution to back up the `kodekloud_db01` MySQL database hosted on the Stratos Datacenter application server (stapp01). Manual backup processes are error-prone and unsustainable at scale. The backup file must be timestamped, transferred to a centralized storage server (ststor01) under a designated path, and the entire process must run on a defined schedule without human intervention.

---

## Solution Architecture

```
Jenkins Controller
      |
      | SSH (key-based, jenkins user -> tony@stapp01)
      v
App Server (stapp01)
      |  mysqldump kodekloud_db01 -> /tmp/db_<date>.sql
      |
      | SCP (key-based, tony@stapp01 -> natasha@ststor01)
      v
Storage Server (ststor01)
      |
      /home/natasha/db_backups/db_<date>.sql
```

The Jenkins job orchestrates the entire chain using a single Execute Shell build step. The job uses an SSH key provisioned under the `jenkins` OS user to reach stapp01, from which a nested SSH/SCP command completes the transfer to ststor01. Temporary dump files are removed from stapp01 after transfer to prevent disk accumulation.

---

## Infrastructure Components

| Component | Hostname | User | Role |
|---|---|---|---|
| Jenkins Controller | jenkins | jenkins | Job orchestrator |
| Application Server | stapp01 | tony | MySQL database host |
| Storage Server | ststor01 | natasha | Backup destination |
| Database | kodekloud_db01 | kodekloud_roy | Target database |

---

## Prerequisites

* Jenkins 2.541.2 or later installed and accessible via web UI
* `mysqldump` installed on stapp01
* Network connectivity between all three servers
* The `/home/natasha/db_backups/` directory must exist on ststor01 prior to job execution
* The `jenkins` OS user must have a home directory at `/var/lib/jenkins`

---

## Implementation Guide

### Step 1: Log In to Jenkins

Navigate to the Jenkins web UI and authenticate using the administrator account.

* **URL:** `http://<jenkins-host>:8080`
* **Username:** `admin`
* **Password:** `Admin321`

Screenshot: Jenkins login page with credentials entered

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/898c8ff3-dc10-452f-a1d0-ac4a2eda279e" />

---

### Step 2: Update Existing Jenkins Plugins

Before installing new plugins, apply any pending updates to ensure dependency compatibility.

1. Navigate to **Manage Jenkins** > **Plugins** > **Updates**
2. Select the **bouncycastle API** plugin (the only pending update)
3. Click **Update**

Screenshot: Plugin Updates page showing bouncycastle API selected for update

The update page confirmed the plugin downloaded successfully and would activate on the next restart.

Screenshot: Download progress page showing "Downloaded Successfully. Will be activated during the next boot"

4. Check the **Restart Jenkins when installation is complete and no jobs are running** checkbox
5. Wait for Jenkins to restart automatically

Screenshot: Jenkins restarting screen ("Jenkins is restarting")

---

### Step 3: Install the Publish Over SSH Plugin

The Publish Over SSH plugin is required to enable SSH-based artifact transfer from Jenkins. Even though the final implementation uses a raw shell script approach rather than the plugin's post-build action, installing this plugin ensures all underlying SSH transport dependencies are available.

1. Navigate to **Manage Jenkins** > **Plugins** > **Available plugins**
2. Search for **Publish Over SSH**
3. Select the checkbox next to **Publish Over SSH** (version 390.vb_f56e7405751)
4. Click **Install**

Screenshot: Available plugins page with "Publish Over SSH" found and selected

The installation resolved and downloaded all required dependency plugins (JAXB, JSON Api, Jackson Annotations 2 API, Jakarta Activation API, SnakeYAML API, Jakarta XML Binding API, Woodstox Core API, Jackson 2 API, Infrastructure plugin for Publish Over X, Structs, EDDSA API, Gson API, Trilead API, Variant, commons-lang3 v3.x Jenkins API, Ionicons API, commons-text API, and more), all reporting **Success**.

Screenshot: Download progress page showing all dependency plugins installed with green "Success" status

5. Jenkins restarted again after the Publish Over SSH installation completed

Screenshot: Jenkins restarting screen ("Jenkins is restarting") following plugin install

---

### Step 4: Establish SSH Access from Jenkins to the App Server

All SSH operations in the build script run as the `jenkins` OS user. This step provisions a key pair for that user and distributes the public key to stapp01.

SSH into the Jenkins server from the jump host:

```bash
ssh jenkins@jenkins
```

Confirm the active user:

```bash
whoami
# jenkins
```

Check whether the `.ssh` directory already exists:

```bash
ls -la ~/.ssh/
# ls: cannot access '/var/lib/jenkins/.ssh/': No such file or directory
```

Create the directory and set correct permissions:

```bash
mkdir -p /var/lib/jenkins/.ssh
chmod 700 /var/lib/jenkins/.ssh
ls -la /var/lib/jenkins/.ssh/
```

Screenshot: Terminal showing .ssh directory created with correct drwx------ permissions

Generate an RSA 4096-bit key pair with no passphrase:

```bash
ssh-keygen -t rsa -b 4096 -f /var/lib/jenkins/.ssh/id_rsa -N ""
```

Verify the key pair was created:

```bash
ls -la /var/lib/jenkins/.ssh/
# -rw------- 1 jenkins jenkins 3414 Apr 21 19:49 id_rsa
# -rw-r--r-- 1 jenkins jenkins  765 Apr 21 19:49 id_rsa.pub
```

Screenshot: Terminal showing key generation output and both id_rsa and id_rsa.pub present

Copy the public key to stapp01 under the `tony` user account:

```bash
ssh-copy-id -i /var/lib/jenkins/.ssh/id_rsa.pub tony@stapp01
# Number of key(s) added: 1
```

Validate the connection is passwordless:

```bash
ssh -i /var/lib/jenkins/.ssh/id_rsa tony@stapp01 "echo Successfully connected to stapp01"
# Successfully connected to stapp01
```

Screenshot: Terminal confirming key copied to stapp01 and passwordless connection validated

---

### Step 5: Establish SSH Trust from the App Server to the Storage Server

The build script uses a nested SSH command (`tony@stapp01` executing `scp` to `natasha@ststor01`). This requires stapp01 to have passwordless SSH access to ststor01. This trust is configured directly on stapp01.

From the Jenkins server, SSH into stapp01 as `tony`:

```bash
ssh -i /var/lib/jenkins/.ssh/id_rsa tony@stapp01
```

On stapp01, generate a key pair for the `tony` user:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

Screenshot: Terminal on stapp01 showing key pair generated successfully

Verify the `.ssh` directory now contains both the `authorized_keys` file (from the earlier `ssh-copy-id`) and the new key pair:

```bash
ls -la ~/.ssh/
# drwx------ 2 tony tony 4096 Apr 21 19:57 .
# drwx------ 1 tony tony 4096 Apr 21 19:51 ..
# -rw------- 1 tony tony  765 Apr 21 19:51 authorized_keys
# -rw------- 1 tony tony 3381 Apr 21 19:57 id_rsa
# -rw-r--r-- 1 tony tony  738 Apr 21 19:57 id_rsa.pub
```

Copy the public key from stapp01 to ststor01 under the `natasha` user:

```bash
ssh-copy-id natasha@ststor01
# Number of key(s) added: 1
```

Verify the connection from stapp01 to ststor01 is passwordless:

```bash
ssh natasha@ststor01 "echo Successfully connected to ststor01"
# Successfully connected to ststor01
```

Create the backup destination directory on ststor01 and confirm it exists:

```bash
ssh natasha@ststor01 "mkdir -p /home/natasha/db_backups && ls -la /home/natasha/"
# drwxr-xr-x 2 natasha natasha 4096 Apr 21 19:34 db_backups
```

Screenshot: Terminal on stapp01 confirming ststor01 connection is passwordless and db_backups directory exists

Exit stapp01 to return to the Jenkins server:

```bash
exit
# Connection to stapp01 closed.
```

---

### Step 6: Verify the Full SSH Chain

Before creating the Jenkins job, validate the complete execution path end-to-end from the Jenkins server through stapp01 to ststor01.

```bash
ssh -i /var/lib/jenkins/.ssh/id_rsa tony@stapp01 "ssh natasha@ststor01 'echo Full chain working'"
# Full chain working
```

Screenshot: Terminal on Jenkins server showing "Full chain working" confirming the three-hop SSH chain is operational

---

### Step 7: Create the Jenkins Freestyle Job

1. In the Jenkins UI, click **New Item** from the dashboard
2. Enter the item name: `database-backup`
3. Select **Freestyle project**
4. Click **OK**

Screenshot: New Item page with "database-backup" entered and Freestyle project selected

---

### Step 8: Configure the Build Trigger

In the job configuration page, scroll to the **Triggers** section:

1. Check **Build periodically**
2. In the **Schedule** field, enter:

```
*/10 * * * *
```

Jenkins will display a warning suggesting `H/10 * * * *` for load spreading. The task specification requires `*/10 * * * *` exactly, so this warning is acknowledged and the required format is retained.

Screenshot: Triggers section showing "Build periodically" checked with schedule set to "*/10 * * * *"

---

### Step 9: Write the Build Shell Script

Scroll to the **Build Steps** section:

1. Click **Add build step** > **Execute shell**
2. Enter the following script:

```bash
#!/bin/bash

DUMP_FILE="db_$(date +%F).sql"

ssh -i /var/lib/jenkins/.ssh/id_rsa tony@stapp01 "mysqldump -u kodekloud_roy -pasdfgdsd kodekloud_db01 > /tmp/${DUMP_FILE}"

if [ $? -ne 0 ]; then
    echo "ERROR: mysqldump failed"
    exit 1
fi

echo "Dump created: /tmp/${DUMP_FILE}"

ssh -i /var/lib/jenkins/.ssh/id_rsa tony@stapp01 "scp /tmp/${DUMP_FILE} natasha@ststor01:/home/natasha/db_backups/${DUMP_FILE}"

if [ $? -ne 0 ]; then
    echo "ERROR: SCP to ststor01 failed"
    exit 1
fi

echo "Dump copied to ststor01:/home/natasha/db_backups/${DUMP_FILE}"

ssh -i /var/lib/jenkins/.ssh/id_rsa tony@stapp01 "rm -f /tmp/${DUMP_FILE}"

echo "Cleanup done. Backup completed successfully."
```

Screenshot: Build Steps section showing the complete shell script in the Execute shell command box

3. Click **Save**

---

### Step 10: Save and Execute the First Build

After saving, the job configuration page redirected to the `database-backup` job status page. Click **Build Now** to trigger the first manual execution.

Screenshot: database-backup job status page with Build Now option visible

Navigate to **Build #1** > **Console Output** to inspect the execution:

```
Started by user admin
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/database-backup
[database-backup] $ /bin/bash /tmp/jenkins10032617100517874706.sh
Dump created: /tmp/db_2026-04-21.sql
Dump copied to ststor01:/home/natasha/db_backups/db_2026-04-21.sql
Cleanup done. Backup completed successfully.
Finished: SUCCESS
```

Screenshot: Console Output for Build #1 showing all pipeline stages completed and "Finished: SUCCESS"

---

### Step 11: Verify the Backup File on the Storage Server

From the Jenkins server, confirm the backup file landed correctly on ststor01 by listing the remote directory via the SSH chain:

```bash
ssh -i /var/lib/jenkins/.ssh/id_rsa tony@stapp01 "ssh natasha@ststor01 'ls -lh /home/natasha/db_backups/'"
# total 4.0K
# -rw-r--r-- 1 natasha natasha 1.3K Apr 21 20:06 db_2026-04-21.sql
```

Screenshot: Terminal confirming db_2026-04-21.sql exists in /home/natasha/db_backups/ with correct size and ownership

The backup file `db_2026-04-21.sql` is present on ststor01 with a 1.3K size, owned by `natasha`, confirming end-to-end pipeline success.

---

## Shell Script Reference

| Variable | Value | Purpose |
|---|---|---|
| `DUMP_FILE` | `db_$(date +%F).sql` | Generates a date-stamped filename (e.g., `db_2026-04-21.sql`) |
| SSH identity file | `/var/lib/jenkins/.ssh/id_rsa` | Key used by Jenkins to reach stapp01 as `tony` |
| Database user | `kodekloud_roy` | MySQL user with read access to `kodekloud_db01` |
| Temp dump path | `/tmp/${DUMP_FILE}` | Staging path on stapp01, removed after transfer |
| Destination path | `/home/natasha/db_backups/${DUMP_FILE}` | Final resting place on ststor01 |

---

## Best Practices Applied

* **Key-based authentication only.** No passwords are used anywhere in the automated pipeline. All SSH connections rely on RSA 4096-bit key pairs with restricted permissions (`chmod 700` on directories, `chmod 600` on private keys).

* **Explicit exit code checking.** Each critical remote command (`mysqldump`, `scp`) is followed by an `if [ $? -ne 0 ]` guard with a descriptive error message and non-zero exit to ensure Jenkins marks the build as failed rather than silently proceeding after a partial failure.

* **Temporary file cleanup.** The dump file is removed from stapp01 after a successful transfer to ststor01, preventing unbounded disk consumption on the application server.

* **Timestamped filenames.** Using `date +%F` produces ISO 8601 formatted filenames (`YYYY-MM-DD`), making backups human-readable, sortable, and unambiguous across time zones.

* **Prerequisite validation before job creation.** The full SSH chain was verified end-to-end from the Jenkins server before the job was configured, ensuring that infrastructure issues would surface prior to job execution rather than during a scheduled build.

* **Plugin dependency management.** Pending plugin updates were applied before installing the new Publish Over SSH plugin to prevent version conflicts between plugin dependencies.

* **Cron format compliance.** The schedule `*/10 * * * *` was used exactly as required by the task specification. Jenkins' recommendation to use `H/10 * * * *` for load spreading was noted but intentionally not applied.

---

## Errors Encountered and Resolutions

### SSH Directory Missing for jenkins User

**Symptom:** Running `ls -la ~/.ssh/` as the `jenkins` user returned `ls: cannot access '/var/lib/jenkins/.ssh/': No such file or directory`.

**Root Cause:** The `jenkins` OS user's home directory was created during Jenkins installation but no SSH operations had ever been performed under that user, so the `.ssh` directory was never initialized.

**Resolution:** Manually created the directory and applied the required permissions:

```bash
mkdir -p /var/lib/jenkins/.ssh
chmod 700 /var/lib/jenkins/.ssh
```

---

### Jenkins UI Becomes Unresponsive After Plugin Restart

**Symptom:** After checking "Restart Jenkins when installation is complete," the browser displayed a blank or partially loaded page.

**Root Cause:** Jenkins backend restarts asynchronously. The browser may attempt to reload before the service is fully back online.

**Resolution:** Manually refreshed the browser tab after waiting approximately 30 to 60 seconds. The Jenkins login page reappeared normally after the service completed its restart cycle.

---

## Lessons Learned

* **Chain your SSH trust before writing a single line of script.** Validating `jenkins -> tony@stapp01 -> natasha@ststor01` with a simple `echo` command before touching the Jenkins job configuration saves significant debugging time. An untested chain will always fail silently in an automated context.

* **The `jenkins` OS user is not the same as the Jenkins web UI admin user.** SSH key provisioning must be done under the `jenkins` OS user account (typically homed at `/var/lib/jenkins`), not the account used to log in to the web UI. Confusing these two identities is a common source of authentication failures.

* **`mysqldump` password flags with no space are intentional.** The `-pasdfgdsd` syntax (password value immediately following `-p` with no space) is standard MySQL CLI behavior. A space between `-p` and the password value causes MySQL to prompt interactively, which breaks non-interactive scripts.

* **Nested SSH commands require the intermediate host to have its own key pair.** The `scp` command inside the SSH session on stapp01 initiates a new SSH connection originating from stapp01 to ststor01. This means stapp01 must have its own private key and ststor01 must have the corresponding public key in `authorized_keys`. Distributing only the Jenkins key to ststor01 directly is insufficient for this architecture.

* **Temporary files on application servers require explicit cleanup.** Database dumps can be large. Without the `rm -f` step, every scheduled execution accumulates a new dump file in `/tmp` on the application server. At a 10-minute interval, this adds up quickly and can fill the filesystem.

* **Plugin installation order matters.** Updating existing plugins before installing new ones prevents dependency resolution failures caused by stale plugin versions conflicting with newly installed packages.










<img width="1918" height="1016" alt="image" src="https://github.com/user-attachments/assets/f917d390-c640-4f2e-9463-5dfaaf4f61a3" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/74770ec7-b458-49f8-b2fb-cdb2c08632a4" />
<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/88b8f8d0-e670-402c-b849-2ad902b1e9c5" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/9ac890e3-5874-4764-b393-4b2ca5842c39" />
<img width="1919" height="1023" alt="image" src="https://github.com/user-attachments/assets/aab3d51c-94b3-4254-b5d9-28185753874e" />
<img width="1918" height="1018" alt="image" src="https://github.com/user-attachments/assets/69c7b8bf-da61-4f39-a636-e4aaaa6bf4f4" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/0ffb31e1-d3c6-423c-81a1-ad1deb115598" />
<img width="1035" height="778" alt="image" src="https://github.com/user-attachments/assets/8e17bacd-760b-47a0-8d90-ad8efe4e5eaf" />
<img width="1027" height="750" alt="image" src="https://github.com/user-attachments/assets/de5bcdea-1770-4925-9c6a-326a60abfe21" />
<img width="1033" height="780" alt="image" src="https://github.com/user-attachments/assets/be5037df-97c9-445e-9b26-919f3c4fc003" />
<img width="1030" height="857" alt="image" src="https://github.com/user-attachments/assets/d40dba52-2dbb-4fb8-91da-0725ff588aa1" />
<img width="1031" height="619" alt="image" src="https://github.com/user-attachments/assets/adfcf076-1c0f-4e05-8f0b-4381555249f6" />
<img width="1035" height="676" alt="image" src="https://github.com/user-attachments/assets/2b17569a-537c-4423-9ef2-0edaf0f778aa" />
<img width="1028" height="740" alt="image" src="https://github.com/user-attachments/assets/cd022d46-20f4-478b-bdac-6faaf4aaa639" />
<img width="1036" height="741" alt="image" src="https://github.com/user-attachments/assets/fad2055e-bd4d-4ad7-9f30-1c5649d0eb6b" />
<img width="1027" height="781" alt="image" src="https://github.com/user-attachments/assets/bf29ed0a-e12b-4d39-9c6e-12018523b2e1" />
<img width="1030" height="452" alt="image" src="https://github.com/user-attachments/assets/b578f211-e4d4-456e-9dc0-77254b3fd6ec" />
<img width="1031" height="402" alt="image" src="https://github.com/user-attachments/assets/4c577205-dce9-49ef-b3e2-301e33ea62d8" />
<img width="1030" height="613" alt="image" src="https://github.com/user-attachments/assets/aa075a6b-d18b-46a6-9b81-5240f0983d64" />
<img width="1033" height="752" alt="image" src="https://github.com/user-attachments/assets/a06fcc39-f83b-4c65-8d12-891faa07375b" />
<img width="1030" height="759" alt="image" src="https://github.com/user-attachments/assets/6be4beca-916b-4177-bbc7-d48b8b347157" />
<img width="1020" height="845" alt="image" src="https://github.com/user-attachments/assets/7084abd0-c7f5-426a-903c-df30b56f6d4e" />
<img width="1024" height="715" alt="image" src="https://github.com/user-attachments/assets/54ddf492-8f84-4cdf-9b42-92d43fe95edd" />
<img width="1028" height="857" alt="image" src="https://github.com/user-attachments/assets/f9440326-a706-497f-bafc-493b0cfdcde8" />
<img width="1031" height="450" alt="image" src="https://github.com/user-attachments/assets/c0c6ab7b-ede2-4cf6-9de7-78ca869625a4" />
<img width="1919" height="1013" alt="image" src="https://github.com/user-attachments/assets/c931777d-9d66-4233-89b0-f121f8a6d138" />
<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/ed891905-2a76-4508-b1a0-481b96341224" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/30274957-1341-4579-bf08-c89b15b783c6" />
<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/2ad4f4d5-2b7f-442c-9bdf-c607fb597dab" />
<img width="1919" height="1017" alt="image" src="https://github.com/user-attachments/assets/84f61f03-c7a3-47d0-ac5c-d8bc3b665405" />
<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/fe924fba-1e81-469a-ad0e-dbdf07619f40" />
<img width="1032" height="568" alt="image" src="https://github.com/user-attachments/assets/be5a8412-923c-4cac-807f-5ec0257f02df" />
