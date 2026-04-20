# Jenkins Centralised Log Collection: Apache Logs Pipeline from App Server to Storage Server

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Solution Design](#architecture-and-solution-design)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Access Jenkins and Log In](#step-1-access-jenkins-and-log-in)
  - [Step 2: Install Pending Plugin Updates](#step-2-install-pending-plugin-updates)
  - [Step 3: Install the Publish Over SSH Plugin](#step-3-install-the-publish-over-ssh-plugin)
  - [Step 4: Restart Jenkins After Plugin Installation](#step-4-restart-jenkins-after-plugin-installation)
  - [Step 5: Configure the SSH Server in Jenkins System Settings](#step-5-configure-the-ssh-server-in-jenkins-system-settings)
  - [Step 6: Resolve the Remote Directory Error on the Storage Server](#step-6-resolve-the-remote-directory-error-on-the-storage-server)
  - [Step 7: Validate the SSH Server Connection](#step-7-validate-the-ssh-server-connection)
  - [Step 8: Create the copy-logs Freestyle Job](#step-8-create-the-copy-logs-freestyle-job)
  - [Step 9: Configure the Build Trigger and Shell Build Step](#step-9-configure-the-build-trigger-and-shell-build-step)
  - [Step 10: Configure the Post-Build SSH Transfer Action](#step-10-configure-the-post-build-ssh-transfer-action)
  - [Step 11: Trigger the First Build Manually](#step-11-trigger-the-first-build-manually)
  - [Step 12: Verify Build 1 Console Output](#step-12-verify-build-1-console-output)
  - [Step 13: Verify Files on the Storage Server](#step-13-verify-files-on-the-storage-server)
  - [Step 14: Confirm Scheduled Build Execution](#step-14-confirm-scheduled-build-execution)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)

---

## Overview

This project implements an automated Apache log collection pipeline using Jenkins. A freestyle job named `copy-logs` is configured to periodically pull `access_log` and `error_log` from App Server 1 (`stapp01`) and deliver them to a designated directory on the Storage Server (`ststor01`) via SSH. The job runs every 3 minutes on a cron schedule and uses the **Publish Over SSH** plugin to perform the final file transfer.

| Attribute | Value |
|---|---|
| Jenkins Job Name | `copy-logs` |
| Job Type | Freestyle Project |
| Build Trigger | Build periodically (`*/3 * * * *`) |
| Source Server | `stapp01` (App Server 1) |
| Source Log Path | `/var/log/httpd/` |
| Source Files | `access_log`, `error_log` |
| Destination Server | `ststor01` (Storage Server) |
| Destination Path | `/usr/src/itadmin` |
| Transfer Mechanism | Publish Over SSH plugin + sshpass SCP (Build Step) |

---

## Problem Statement

The xFusionCorp Industries DevOps team is building a centralised logging management system. At least one application server is experiencing issues with Apache, and the team requires ongoing access to Apache logs (`access_log` and `error_log`) for troubleshooting. A manual log retrieval process is not sustainable. The solution must automate log collection on a recurring schedule, delivering logs from App Server 1 to the Storage Server for centralised analysis.

---

## Architecture and Solution Design

```
+-------------------+        sshpass + SCP         +---------------------+
|   stapp01         |  ---------------------------> | Jenkins Workspace   |
|  /var/log/httpd/  |   access_log, error_log       | /var/lib/jenkins/   |
|  access_log       |                               | workspace/copy-logs |
|  error_log        |                               +----------+----------+
+-------------------+                                          |
                                                               | Publish Over SSH
                                                               | (Post-build Action)
                                                               v
                                              +----------------+----------------+
                                              |   ststor01 (Storage Server)     |
                                              |   /usr/src/itadmin/             |
                                              |   access_log, error_log         |
                                              +---------------------------------+
```

**Two-phase transfer design:**

1. **Build Step (Shell):** Uses `sshpass` with `scp` to pull log files from `stapp01` into the Jenkins workspace on the controller node.
2. **Post-Build Action (Publish Over SSH):** Transfers the workspace files to the Storage Server `ststor01` using the pre-configured SSH server entry.

This separation of concerns makes each phase independently testable and auditable.

---

## Prerequisites

- Jenkins 2.x instance accessible at port 8080
- Admin credentials for Jenkins (`admin` / `Adm!n321`)
- Network connectivity from Jenkins controller to `stapp01` and `ststor01`
- `sshpass` installed on the Jenkins controller
- User `natasha` on `ststor01` with SSH access and write permissions to `/usr/src/itadmin`
- User `tony` on `stapp01` with read access to `/var/log/httpd/`

---

## Implementation Guide

### Step 1: Access Jenkins and Log In

Navigate to the Jenkins UI via the provided URL on port 8080. Log in using the admin credentials.

- **Username:** `admin`
- **Password:** `Adm!n321`

> Screenshot: Jenkins login page with admin username entered and password field populated


<img width="1917" height="1021" alt="image" src="https://github.com/user-attachments/assets/a3d4f355-837b-42c0-9f65-cc7ee35e985c" />

---

### Step 2: Install Pending Plugin Updates

Before installing new plugins, apply any available updates to keep the Jenkins instance stable.

1. Navigate to **Manage Jenkins** > **Plugins** > **Updates**.
2. Select all available updates (in this case, the **bouncycastle API** plugin had an available update).
3. Click **Update**.

> Screenshots: Plugin Updates page showing the bouncycastle API plugin selected for update

<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/b343282d-bd94-44da-bd24-21cd0fd0ef60" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/37f3560a-3ada-47f6-be26-9dee6effcfd6" />
<img width="1918" height="1019" alt="image" src="https://github.com/user-attachments/assets/34a2f279-674a-48fc-9bd9-c58ba9ad5c36" />

---

### Step 3: Install the Publish Over SSH Plugin

The `copy-logs` job requires the **Publish Over SSH** plugin to transfer files from the Jenkins workspace to the Storage Server.

1. Navigate to **Manage Jenkins** > **Plugins** > **Available plugins**.
2. Search for `publish over ssh`.
3. Select **Publish Over SSH** (version 390.vb_f56e7405751).
4. Click **Install**.

> Screenshot: Available plugins page with "Publish Over SSH" plugin found and selected for installation

<img width="1918" height="1018" alt="image" src="https://github.com/user-attachments/assets/f0dce57c-1807-4df5-9ee5-4e86d9b989e8" />


---

### Step 4: Restart Jenkins After Plugin Installation

After the plugin installation completes (all dependencies show **Success** on the Download Progress page), Jenkins must be restarted for the plugin to become active.

The restart was triggered by selecting **Restart Jenkins when installation is complete and no jobs are running** during the install process.

> Screenshots: Download progress page showing all plugin dependencies (JAXB, JSON Api, Jackson Annotations, Publish Over SSH, etc.) with Success status

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/9f590c52-a874-4071-9057-7ae0f3f7891a" />
<img width="1919" height="1015" alt="image" src="https://github.com/user-attachments/assets/d5be5c07-36c3-4845-bb0b-2cbb8d08767b" />

> Screenshot: Jenkins restarting screen showing "Jenkins is restarting" message with safe restart notice

<img width="1917" height="1020" alt="image" src="https://github.com/user-attachments/assets/99e587cf-9fb6-40ff-951d-afbec2f53564" />

---

### Step 5: Configure the SSH Server in Jenkins System Settings

After Jenkins restarts, configure the SSH connection to the Storage Server so the Publish Over SSH plugin can authenticate and transfer files.

1. Navigate to **Manage Jenkins** > **System**.
2. Scroll to the **SSH Servers** section (added by the Publish Over SSH plugin).
3. Click **Add** and populate the fields:

| Field | Value |
|---|---|
| Name | `ststor01` |
| Hostname | `ststor01` |
| Username | `natasha` |
| Remote Directory | `/usr/src/itadmin` |

4. Expand **Advanced** and check **Use password authentication, or use a different key**.
5. Enter the password for the `natasha` user in the **Passphrase / Password** field.
6. Click **Test Configuration**.

> Screenshots: Jenkins System configuration page showing the SSH Server section populated with ststor01 hostname, natasha username, and /usr/src/itadmin remote directory with password authentication enabled

<img width="1919" height="1016" alt="image" src="https://github.com/user-attachments/assets/79c22ced-d268-4ca9-93d7-1faaf93b6543" />
<img width="1919" height="1019" alt="image" src="https://github.com/user-attachments/assets/1e3b5425-59d3-4f26-8e27-1c5821138ba5" />

---

### Step 6: Resolve the Remote Directory Error on the Storage Server

The initial **Test Configuration** attempt failed with the following error:

```
Failed to connect or change directory
jenkins.plugins.publish_over.BapPublisherException: Failed to connect and initialize SSH connection.
Message: [Failed to change to remote directory [/usr/src/itadmin]]
```

**Root cause:** The directory `/usr/src/itadmin` did not exist on `ststor01`, and the `natasha` user did not have write ownership of the path.

**Resolution:** SSH into `ststor01` from the jump host, create the directory, and assign ownership to `natasha`.

```bash
ssh natasha@ststor01

sudo mkdir -p /usr/src/itadmin
sudo chown -R natasha:natasha /usr/src/itadmin

ls -ld /usr/src/itadmin
# drwxr-xr-x 2 natasha natasha 4096 Apr 20 03:31 /usr/src/itadmin

exit
```

> Screenshot: Terminal on jump host showing SSH session to ststor01, mkdir and chown commands executed, and ls -ld confirming directory exists with natasha ownership

<img width="1033" height="648" alt="image" src="https://github.com/user-attachments/assets/03f01295-ff95-42d2-acfa-5cb61cbb1c4c" />

---

### Step 7: Validate the SSH Server Connection

After creating the directory and setting permissions on `ststor01`, return to the Jenkins System configuration and click **Test Configuration** again.

The result now shows **Success**, confirming Jenkins can authenticate to `ststor01` as `natasha` and access the `/usr/src/itadmin` directory.

> Screenshot: Jenkins System configuration page showing "Success" result from Test Configuration for the ststor01 SSH server entry

<img width="1919" height="1018" alt="image" src="https://github.com/user-attachments/assets/a5bb360d-eff8-43cc-932f-fb59a9b00452" />

Save the system configuration.

---

### Step 8: Create the copy-logs Freestyle Job

1. From the Jenkins dashboard, click **New Item**.
2. Enter the item name: `copy-logs`.
3. Select **Freestyle project**.
4. Click **OK**.

> Screenshot: New Item page with "copy-logs" entered as the item name and Freestyle project selected

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/07f9a309-1d7d-4f04-8596-47c6bc06eae5" />

---

### Step 9: Configure the Build Trigger and Shell Build Step

Inside the `copy-logs` job configuration:

**Build Triggers:**

1. Check **Build periodically**.
2. In the **Schedule** field, enter: `*/3 * * * *`

This runs the job every 3 minutes. Jenkins displays a confirmation showing the next two scheduled run times.

**Build Steps:**

1. Click **Add build step** > **Execute shell**.
2. Enter the following shell commands to pull the Apache logs from `stapp01` into the Jenkins workspace:

```bash
sshpass -p 'Ir0nM@n' scp -o StrictHostKeyChecking=no \
    tony@stapp01:/var/log/httpd/access_log \
    $WORKSPACE/access_log

sshpass -p 'Ir0nM@n' scp -o StrictHostKeyChecking=no \
    tony@stapp01:/var/log/httpd/error_log \
    $WORKSPACE/error_log
```

These commands copy both log files from the default Apache log location on `stapp01` into the Jenkins workspace directory for the `copy-logs` job.

> Screenshot: Job Configure page showing Build Triggers section with "Build periodically" checked and schedule */3 * * * * entered, and the Execute shell build step populated with the two sshpass scp commands

<img width="1919" height="1021" alt="image" src="https://github.com/user-attachments/assets/cf11e15d-dad2-4d02-b550-717675ddbc7b" />

---

### Step 10: Configure the Post-Build SSH Transfer Action

1. Scroll down to the **Post-build Actions** section.
2. Click **Add post-build action** > **Send build artifacts over SSH**.
3. Configure the SSH Publisher:

| Field | Value |
|---|---|
| Name (SSH Server) | `ststor01` |
| Source files | `access_log, error_log` |
| Remove prefix | *(empty)* |
| Remote directory | *(empty, uses the base remote directory /usr/src/itadmin configured in system settings)* |
| Exec command | *(empty)* |

4. Click **Save**.

> Screenshot: Post-build Actions section showing "Send build artifacts over SSH" configured with ststor01 selected as the SSH server and "access_log, error_log" in the Source files field

<img width="1919" height="1024" alt="image" src="https://github.com/user-attachments/assets/fac9806a-029c-4762-927f-1d3141763d4d" />

---

### Step 11: Trigger the First Build Manually

From the `copy-logs` job status page, click **Build Now** to execute the first build immediately without waiting for the 3-minute cron trigger.

The bottom of the page shows a **"Build scheduled"** notification confirming the build has been queued.

> Screenshot: copy-logs job status page showing "Build scheduled" notification at the bottom after clicking Build Now

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/d4e25c26-8be8-4b44-9cb4-904bb57b19b2" />

---

### Step 12: Verify Build 1 Console Output

Navigate to **Build #1** > **Console Output**. The output confirms the full pipeline executed successfully:

```
Started by user admin
Running as SYSTEM
Building in workspace /var/lib/jenkins/workspace/copy-logs
[copy-logs] $ /bin/sh -xe /tmp/jenkins3492884625667986837.sh
+ sshpass -p Ir0nM@n scp -o StrictHostKeyChecking=no tony@stapp01:/var/log/httpd/access_log /var/lib/jenkins/workspace/copy-logs/access_log
Warning: Permanently added 'stapp01' (ED25519) to the list of known hosts.
+ sshpass -p Ir0nM@n scp -o StrictHostKeyChecking=no tony@stapp01:/var/log/httpd/error_log /var/lib/jenkins/workspace/copy-logs/error_log
SSH: Connecting from host [jenkins.stratos.xfusioncorp.com]
SSH: Connecting with configuration [ststor01] ...
SSH: Disconnecting configuration [ststor01] ...
SSH: Transferred 2 file(s)
Finished: SUCCESS
```

Key observations from the output:

- The SCP commands executed and pulled both log files from `stapp01`.
- The SSH post-build publisher connected to `ststor01` using the `ststor01` configuration.
- 2 files were successfully transferred to `/usr/src/itadmin` on the Storage Server.
- The build completed with **Finished: SUCCESS**.

> Screenshot: Console Output for Build #1 showing all execution steps and "Finished: SUCCESS" at the bottom

<img width="1919" height="1020" alt="image" src="https://github.com/user-attachments/assets/6273cdef-1eb8-45d0-9795-d83320c534e7" />

---

### Step 13: Verify Files on the Storage Server

SSH into `ststor01` from the jump host and list the contents of `/usr/src/itadmin` to confirm the log files arrived.

```bash
ssh natasha@ststor01

ls -lh /usr/src/itadmin/
# total 8.0K
# -rw-r--r-- 1 natasha natasha  41 Apr 20 03:42 access_log
# -rw-r--r-- 1 natasha natasha 733 Apr 20 03:42 error_log

exit
```

Both `access_log` and `error_log` are present with timestamps matching the build execution time, confirming end-to-end delivery.

> Screenshot: Terminal on jump host showing SSH session to ststor01 with ls -lh /usr/src/itadmin output displaying access_log (41 bytes) and error_log (733 bytes) with Apr 20 03:42 timestamps

<img width="1033" height="749" alt="image" src="https://github.com/user-attachments/assets/4f042519-be24-49bb-b22e-fc544f869cd5" />

---

### Step 14: Confirm Scheduled Build Execution

Return to the `copy-logs` job status page. The **Builds** panel shows **Build #2** was triggered automatically by the cron timer at 3:45 AM, 3 minutes after Build #1 at 3:42 AM. Both builds show green success indicators.

The Build #2 console output confirms the trigger was the scheduler (`Started by timer`) and the same two-phase pipeline completed successfully again, transferring 2 files.

> Screenshot: copy-logs job status page showing Build #2 (3:45 AM) and Build #1 (3:42 AM) both with green success icons in the Builds panel

<img width="1919" height="1009" alt="image" src="https://github.com/user-attachments/assets/a38daa61-ab1c-47af-92d6-4901c8186269" />

> Screenshot: Console Output for Build #2 showing "Started by timer" and "Finished: SUCCESS" confirming the scheduled execution worked correctly

<img width="1919" height="1012" alt="image" src="https://github.com/user-attachments/assets/ae4939eb-7b98-450b-8abe-602d35770bf2" />

---

## Errors Encountered and Resolutions

### Error 1: SSH Connection Failure Due to Missing Remote Directory

**Symptom:**

When testing the SSH Server configuration in Jenkins System settings, the Test Configuration button returned:

```
Failed to connect or change directory
jenkins.plugins.publish_over.BapPublisherException: Failed to connect and initialize SSH connection.
Message: [Failed to change to remote directory [/usr/src/itadmin]]
```

**Root Cause:**

The Publish Over SSH plugin attempts to change into the configured **Remote Directory** immediately upon connecting. The directory `/usr/src/itadmin` had not been created on `ststor01`, causing the connection initialization to fail.

**Resolution:**

SSH into `ststor01` and create the directory with correct ownership before retrying the connection test:

```bash
sudo mkdir -p /usr/src/itadmin
sudo chown -R natasha:natasha /usr/src/itadmin
```

After applying these changes, the Test Configuration returned **Success**.

**Prevention:**

Always pre-provision destination directories on remote servers before configuring Jenkins SSH publishers. Automating this as part of server provisioning (Ansible, Terraform remote-exec, or cloud-init) eliminates this class of failure entirely.

---

## Best Practices Applied

**Plugin management before job creation:** Plugin updates were applied and Jenkins was restarted before beginning job configuration. This prevents plugin version conflicts and ensures the UI reflects all available options.

**Two-phase transfer architecture:** Separating the log retrieval (SCP from stapp01 into workspace) from the final delivery (SSH publish to ststor01) makes each phase independently observable and debuggable. Build console output shows both phases explicitly.

**Password-based SSH authentication with isolated credentials:** The `sshpass` approach and the Publish Over SSH password configuration were scoped to specific service accounts (`tony` on stapp01, `natasha` on ststor01), following the principle of least privilege.

**Directory ownership pre-verification:** After creating `/usr/src/itadmin`, ownership was confirmed with `ls -ld` before returning to Jenkins, preventing a second round-trip failure.

**Manual build before relying on the scheduler:** Build #1 was triggered manually via **Build Now** to validate the full pipeline under controlled conditions before the cron timer took over. This is the correct approach for any production job: verify first, then automate.

**Console output review for both manual and scheduled builds:** Build #1 and Build #2 console outputs were both inspected. This confirmed that the `Started by timer` trigger produced identical pipeline behaviour to the manual trigger, validating the cron configuration.

**File presence verification on the destination server:** After the build succeeded, the destination directory on `ststor01` was inspected directly via SSH to confirm file arrival, file sizes, and timestamps. CI/CD success indicators alone are insufficient without out-of-band verification on first deployment.

---

## Lessons Learned

**The Publish Over SSH plugin validates the remote directory at connection time, not at transfer time.** This means a missing destination directory will cause the SSH server Test Configuration to fail, not just the file transfer step. Pre-creating the destination directory is a required setup step, not an optional optimisation.

**`StrictHostKeyChecking=no` is appropriate in controlled lab environments but should be replaced with known-hosts management in production.** The SSH warning `Permanently added 'stapp01' (ED25519) to the list of known hosts` in the console output indicates the host key was not pre-populated. In production, host keys should be pre-seeded into the Jenkins controller's `known_hosts` file as part of infrastructure provisioning.

**Cron expressions in Jenkins use the same five-field format as standard Unix cron, with an additional hash-based load distribution recommendation.** Jenkins warned about using `H/3 * * * *` instead of `*/3 * * * *` to spread load across the cluster. For single-server deployments the difference is negligible, but in multi-job environments the `H` hash modifier prevents thundering herd on the scheduler.

**Directory ownership must match the SSH user configured in the plugin, not just the sudo-capable user used during provisioning.** Running `sudo mkdir` creates a root-owned directory. The subsequent `chown` to `natasha:natasha` is a mandatory step, not a courtesy, because the plugin connects as `natasha` and requires write access to deposit files.

**Verifying the automated build trigger separately from the manual build is essential.** A job that succeeds manually but fails on a cron trigger typically indicates an environment variable, path, or credential issue that only surfaces when Jenkins runs as `SYSTEM` under the timer context. Observing Build #2 console output showing `Started by timer` with identical success confirmed the cron trigger was wired correctly.
