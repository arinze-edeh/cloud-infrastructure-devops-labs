# Automated Website Backup with SSH Key-Based Remote Transfer

A production-grade bash automation for archiving a static website and securely transferring backups to a remote server using passwordless SSH authentication.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure](#infrastructure)
- [Prerequisites](#prerequisites)
- [Step 1: Connect to the Application Server](#step-1-connect-to-the-application-server)
- [Step 2: Install Required Packages](#step-2-install-required-packages)
- [Step 3: Configure Passwordless SSH Authentication](#step-3-configure-passwordless-ssh-authentication)
- [Step 4: Prepare Required Directories](#step-4-prepare-required-directories)
- [Step 5: Create the Backup Script](#step-5-create-the-backup-script)
- [Step 6: Make the Script Executable](#step-6-make-the-script-executable)
- [Step 7: Execute the Backup Script](#step-7-execute-the-backup-script)
- [Step 8: Verify Local Backup](#step-8-verify-local-backup)
- [Step 9: Verify Remote Backup](#step-9-verify-remote-backup)
- [Validation Checklist](#validation-checklist)
- [Operational Considerations](#operational-considerations)
- [Tags](#tags)

---

## Overview

This document covers the end-to-end implementation of a website backup automation solution for a Linux-based application server. The script archives a static website directory into a ZIP file, stores it locally, and securely transfers it to a dedicated backup server using `scp` over SSH key-based authentication. No `sudo` privileges are used within the script itself, making it safe to schedule via cron under a non-privileged user.

---

## Problem Statement

Production websites require regular, automated backups to prevent data loss and support disaster recovery. Manual backup processes are error-prone and inconsistent. Additionally, password-based remote copy operations are insecure and cannot be automated without storing credentials in plaintext.

**Solution:** A self-contained bash script that:
- Creates a timestamped ZIP archive of the website directory
- Stores the backup locally on the application server
- Transfers the backup to a remote backup server using passwordless SSH (RSA key authentication)
- Requires no `sudo` within the script, enabling safe automation via cron

---

## Infrastructure

| Role | Hostname | IP Address | User |
|---|---|---|---|
| **Application Server** | `stapp02.stratos.xfusioncorp.com` | `172.16.238.11` | `steve` |
| **Backup Server** | `stbkp01.stratos.xfusioncorp.com` | `172.16.238.16` | `clint` |

| Resource | Path |
|---|---|
| **Website Directory** | `/var/www/html/official` |
| **Script Location** | `/scripts/official_backup.sh` |
| **Local Backup Directory** | `/backup` |
| **Remote Backup Directory** | `/backup` (on backup server) |
| **Archive Name** | `xfusioncorp_official.zip` |

---

## Prerequisites

- SSH access to the application server (`stapp02`) from the jump host
- `sudo` privileges on the application server for initial directory setup and package installation
- Network connectivity between the application server and the backup server on port 22
- `zip` package available via `yum` (installed in Step 2)

---

## Step 1: Connect to the Application Server

From the jump host (`thor@jumphost`), establish an SSH session to the application server as the `steve` user.

```bash
ssh steve@172.16.238.11
```

On first connection, SSH will prompt you to verify and accept the host fingerprint. Type `yes` to permanently add the host to your `known_hosts` file. Enter the user password when prompted.

**Screenshot: SSH connection to App Server 2**

![SSH connection to stapp02](https://github.com/user-attachments/assets/5179a768-429a-4af0-b5f1-0bd26fd0cfe9)

> **Operational Note:** In production environments, consider using an SSH config file (`~/.ssh/config`) to define aliases for frequently accessed hosts. This reduces the risk of connecting to the wrong host and simplifies automation scripts.

---

## Step 2: Install Required Packages

The `zip` utility is required for creating the backup archive. This installation step is performed manually outside the script, using `sudo`, so the script itself remains privilege-free.

Check if `zip` is already installed:

```bash
which zip
```

If not present, install it using `yum`:

```bash
sudo yum install zip -y
```

This command installs both `zip` and its dependency `unzip` from the CentOS Stream 9 BaseOS repository.

**Screenshot: zip installation initiated**

![zip package installation - transaction summary](https://github.com/user-attachments/assets/3423e8fc-6546-4a5f-8f0b-1168b60afe1d)

**Screenshot: zip installation completed**

![zip package installation - completed successfully](https://github.com/user-attachments/assets/86b041c3-e4fa-453e-8d27-7840d276c8ce)

> **Packages installed:**
> - `zip-3.0-35.el9.x86_64`
> - `unzip-6.0-59.el9.x86_64`

> **Best Practice:** In automated or immutable infrastructure pipelines, package installation should be handled at the image-build level (e.g., via Dockerfile or Packer). Installing packages at runtime is acceptable for mutable server setups but should be documented and tracked.

---

## Step 3: Configure Passwordless SSH Authentication

To allow the backup script to transfer files to the remote backup server without a password prompt, RSA key-based authentication must be configured between the application server and the backup server.

### 3a. Generate an RSA Key Pair

On the application server (`stapp02`), generate a new RSA key pair for the `steve` user:

```bash
ssh-keygen -t rsa
```

Accept the default file location (`/home/steve/.ssh/id_rsa`) and leave the passphrase empty to enable non-interactive authentication in automated scripts.

**Screenshot: SSH key generation**

![RSA key pair generated for steve on stapp02](https://github.com/user-attachments/assets/ca4095f2-42c1-4b83-ba49-f4c291fcd656)

### 3b. Copy the Public Key to the Backup Server

Use `ssh-copy-id` to install the public key on the backup server's `clint` user account:

```bash
ssh-copy-id clint@172.16.238.16
```

This appends `steve`'s public key to `/home/clint/.ssh/authorized_keys` on the backup server. You will be prompted for `clint`'s password exactly once. After this step, password-based authentication for this key is no longer required.

**Screenshot: Public key copied to backup server**

![ssh-copy-id successfully copying key to stbkp01](https://github.com/user-attachments/assets/65d0ab02-c335-472f-a9c3-2ba92fa300ed)

### 3c. Verify Passwordless Login

Confirm that SSH login from `stapp02` to `stbkp01` no longer requires a password:

```bash
ssh clint@172.16.238.16
```

A successful login without a password prompt confirms that key-based authentication is working correctly. Type `exit` to return to the application server.

**Screenshot: Passwordless SSH login confirmed**

![Passwordless SSH login to stbkp01 as clint - verified](https://github.com/user-attachments/assets/c25d7bf4-cd3b-4970-8955-b370e6def799)

> **Security Note:** Leaving a private key without a passphrase is a deliberate trade-off for automation. Restrict access to the key file (`chmod 600 ~/.ssh/id_rsa`) and limit the backup user (`clint`) to minimal required permissions on the backup server. Consider using dedicated keys for automation separate from interactive user keys.

> **Edge Case:** If `ssh-copy-id` fails due to a non-standard SSH port or strict host key checking policies, use the `-p` flag for port specification or pre-add the host fingerprint manually.

---

## Step 4: Prepare Required Directories

Two directories must exist before the script can run: one for the script itself and one for the local backup archive. These are created with `sudo` and ownership is immediately transferred to `steve` so the script can write to them without elevated privileges.

```bash
sudo mkdir -p /scripts
sudo chown steve:steve /scripts

sudo mkdir -p /backup
sudo chown steve:steve /backup
```

**Screenshot: Directories created and ownership assigned**

![/scripts and /backup directories created with correct ownership](https://github.com/user-attachments/assets/78b39b94-17b6-4855-879d-e11def40c121)

> **Rationale:** Separating script storage (`/scripts`) from backup output (`/backup`) follows the principle of separation of concerns. It also makes it straightforward to apply different retention or rotation policies to each directory independently.

---

## Step 5: Create the Backup Script

Create the backup script at `/scripts/official_backup.sh` using a text editor (e.g., `vi`):

```bash
vi /scripts/official_backup.sh
```

Enter the following script content:

```bash
#!/bin/bash

# Task A & B: Create a zip archive of /var/www/html/official and save to /backup
# The flag -r is for recursive (include all subfolders)
zip -r /backup/xfusioncorp_official.zip /var/www/html/official

# Task C: Copy the archive to Nautilus Backup Server
# Using scp with the clint user configured earlier
scp /backup/xfusioncorp_official.zip clint@172.16.238.16:/backup/
```

**Script logic breakdown:**

- **`zip -r /backup/xfusioncorp_official.zip /var/www/html/official`** -- Recursively archives the entire website directory into a single ZIP file stored in the local `/backup` directory.
- **`scp /backup/xfusioncorp_official.zip clint@172.16.238.16:/backup/`** -- Securely copies the archive to the remote backup server using the passwordless SSH key configured in Step 3.

**Screenshot: Backup script content in vi editor**

![official_backup.sh script content displayed in vi](https://github.com/user-attachments/assets/4d62259d-e92c-418e-8e57-48bf9ad09209)

> **Design Constraints:**
> - No `sudo` is used inside the script. Directory permissions were set in Step 4 to allow `steve` to write without elevation.
> - The archive name is static (`xfusioncorp_official.zip`). For production environments, consider appending a timestamp (e.g., `xfusioncorp_official_$(date +%Y%m%d_%H%M%S).zip`) to retain historical backups and avoid overwriting previous archives.

> **Risk:** A static filename means each run overwrites the previous backup. Implement a retention policy (e.g., keeping the last 7 backups) if historical recovery is a requirement.

---

## Step 6: Make the Script Executable

Apply the executable permission to the script so it can be invoked directly:

```bash
chmod +x /scripts/official_backup.sh
```

Verify the permission:

```bash
ls -l /scripts/official_backup.sh
```

The output should show `-rwxr-xr-x` or similar, confirming the execute bit is set.

**Screenshot: chmod applied to the backup script**

![chmod +x applied to official_backup.sh](https://github.com/user-attachments/assets/b2d592da-4ef8-4a19-bda5-950a9bb530f5)

---

## Step 7: Execute the Backup Script

Run the script using its absolute path:

```bash
/scripts/official_backup.sh
```

> **Note:** Running with `./scripts/official_backup.sh` from a different working directory will fail if the relative path does not resolve correctly. Always use the absolute path for scripts in production or cron jobs.

The script will output the files being added to the ZIP archive, followed by the SCP transfer progress, confirming both the local archive creation and the remote transfer.

**Screenshot: Script execution output**

![Backup script running - zip and scp output visible](https://github.com/user-attachments/assets/5c700f07-a37e-4752-a377-cadb9f17472e)

> **Observed output during execution:**
> ```
>   adding: var/www/html/official/ (stored 0%)
>   adding: var/www/html/official/.gitkeep (stored 0%)
>   adding: var/www/html/official/index.html (stored 0%)
> xfusioncorp_official.zip          100%  616    2.4MB/s   00:00
> ```

---

## Step 8: Verify Local Backup

Confirm that the ZIP archive was created successfully in the local `/backup` directory:

```bash
ls -l /backup/xfusioncorp_official.zip
```

**Expected output:**

```
-rw-r--r-- 1 steve steve 616 Feb 16 04:07 /backup/xfusioncorp_official.zip
```

This confirms the archive exists, is owned by `steve`, and has a non-zero file size.

**Screenshot: Local backup file verified**

![ls -l confirming xfusioncorp_official.zip exists in /backup on stapp02](https://github.com/user-attachments/assets/a6d3003d-4da9-49de-be9c-0784ce8598f3)

---

## Step 9: Verify Remote Backup

Confirm that the ZIP archive was successfully transferred to the backup server by running a remote `ls` command over SSH from the application server:

```bash
ssh clint@172.16.238.16 "ls -l /backup/xfusioncorp_official.zip"
```

**Expected output:**

```
-rw-r--r-- 1 clint clint 616 Feb 16 04:07 /backup/xfusioncorp_official.zip
```

This confirms the file exists on the backup server, is owned by `clint`, and matches the size of the local archive -- validating the integrity of the transfer.

**Screenshot: Remote backup verified on backup server**

![SSH remote ls confirming backup file exists on stbkp01](https://github.com/user-attachments/assets/2ba4286f-af62-4fbd-83f9-5ec983b10a8b)

---

## Validation Checklist

| Check | Status |
|---|---|
| `zip` package installed on app server | Complete |
| RSA key pair generated for `steve` on `stapp02` | Complete |
| Public key copied to `clint@stbkp01` via `ssh-copy-id` | Complete |
| Passwordless SSH login to backup server verified | Complete |
| `/scripts` directory created and owned by `steve` | Complete |
| `/backup` directory created and owned by `steve` | Complete |
| Script created at `/scripts/official_backup.sh` | Complete |
| Script made executable with `chmod +x` | Complete |
| Script runs without `sudo` | Complete |
| ZIP archive created at `/backup/xfusioncorp_official.zip` | Complete |
| Archive transferred to `/backup` on backup server | Complete |
| Local and remote file sizes match | Complete |

---

## Operational Considerations

**Scheduling with Cron:**
To automate this backup on a recurring schedule, add a cron entry for the `steve` user:

```bash
crontab -e
```

Example: run daily at 2:00 AM:

```
0 2 * * * /scripts/official_backup.sh >> /var/log/official_backup.log 2>&1
```

**Log output** is redirected to a log file for auditability. Consider implementing log rotation via `logrotate` for long-running deployments.

**Backup Retention:**
The current script overwrites the previous backup on every run. For environments requiring point-in-time recovery, modify the archive name to include a timestamp:

```bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
zip -r /backup/xfusioncorp_official_${TIMESTAMP}.zip /var/www/html/official
scp /backup/xfusioncorp_official_${TIMESTAMP}.zip clint@172.16.238.16:/backup/
```

Pair this with a cleanup routine to purge archives older than a defined retention window (e.g., 30 days).

**Network Failure Handling:**
The current script has no error handling. For production use, add exit code checks after each command:

```bash
zip -r /backup/xfusioncorp_official.zip /var/www/html/official || { echo "zip failed"; exit 1; }
scp /backup/xfusioncorp_official.zip clint@172.16.238.16:/backup/ || { echo "scp failed"; exit 1; }
```

**SSH Key Rotation:**
Establish a key rotation policy. When rotating keys, ensure the new public key is deployed to the backup server before revoking the old one to avoid disruption to automated backups.

---

## Tags

`linux` `bash` `backup` `ssh` `scp` `automation` `cron` `centos` `production-support` `devops`























# Website Backup Automation


## 📌 Project Overview
GOAL:
- Create a bash script to back up a static website
- Archive website files into a ZIP file
- Store backup locally and on a remote backup server
- Ensure passwordless copy (SSH key-based auth)
- Do NOT use sudo inside the script

## 🧱 Infrastructure Details
APP SERVER:
- Hostname: `stapp02.stratos.xfusioncorp.com`
- IP: `172.16.238.11`
- User: `steve`

BACKUP SERVER:
- Hostname: `stbkp01.stratos.xfusioncorp.com`
- IP: `172.16.238.16`
- User: `clint`

WEBSITE DIRECTORY:
- `/var/www/html/official`

📂 Script Location
`/scripts/official_backup.sh`

## 🛠 Step-by-Step Implementation (Pseudo-Code)

## Step 1: Connect to App Server 2
`ssh steve@172.16.238.11`

screenshot:`ssh-to-app-server`
<img width="1150" height="610" alt="image" src="https://github.com/user-attachments/assets/5179a768-429a-4af0-b5f1-0bd26fd0cfe9" />

## Step 2: Install Required Package (Outside Script)
IF zip is NOT installed:
  -  `sudo yum install zip -y`

screenshots:`zip-installation`
<img width="1158" height="857" alt="image" src="https://github.com/user-attachments/assets/3423e8fc-6546-4a5f-8f0b-1168b60afe1d" />
<img width="1153" height="865" alt="image" src="https://github.com/user-attachments/assets/86b041c3-e4fa-453e-8d27-7840d276c8ce" />

## Step 3: Configure Passwordless SSH to Backup Server
GENERATE SSH KEY:
    `ssh-keygen -t rsa`

COPY SSH KEY TO BACKUP SERVER:
  -  `ssh-copy-id clint@172.16.238.16`

VERIFY PASSWORDLESS LOGIN:
  -  `ssh clint@172.16.238.16`

screenshots:`ssh-key-auth-success`
<img width="1152" height="859" alt="image" src="https://github.com/user-attachments/assets/ca4095f2-42c1-4b83-ba49-f4c291fcd656" />
<img width="1155" height="871" alt="image" src="https://github.com/user-attachments/assets/65d0ab02-c335-472f-a9c3-2ba92fa300ed" />
<img width="1151" height="853" alt="image" src="https://github.com/user-attachments/assets/c25d7bf4-cd3b-4970-8955-b370e6def799" />

## Step 4: Prepare Required Directories
- CREATE /scripts DIRECTORY:
  -  `sudo mkdir -p /scripts`
  -  `sudo chown steve:steve /scripts`

- CREATE /backup DIRECTORY:
  -  `sudo mkdir -p /backup`
  -  `sudo chown steve:steve /backup`

screenshot:`directories-created`
<img width="1144" height="861" alt="image" src="https://github.com/user-attachments/assets/78b39b94-17b6-4855-879d-e11def40c121" />

## Step 5: Create Backup Script
CREATE FILE:
  -  `/scripts/official_backup.sh`

SCRIPT LOGIC:

- START
  -  SET SOURCE_DIR = `/var/www/html/official`
  -  SET ARCHIVE_NAME = `xfusioncorp_official.zip`
  -  SET LOCAL_BACKUP_DIR = `/backup`
  -  SET REMOTE_USER = `clint`
  -  SET REMOTE_HOST = `172.16.238.16`
  -  SET REMOTE_BACKUP_DIR = `/backup`

  -  CREATE ZIP ARCHIVE FROM SOURCE_DIR
      -  `zip -r /backup/xfusioncorp_official.zip /var/www/html/official`

  -  COPY ARCHIVE TO BACKUP SERVER
      -  `scp /backup/xfusioncorp_official.zip clint@172.16.238.16:/backup`

END

NOTE:
- NO sudo inside script
- Script must be executable

screenshot:`script-content`
<img width="1154" height="861" alt="image" src="https://github.com/user-attachments/assets/4d62259d-e92c-418e-8e57-48bf9ad09209" />

## Step 6: Make Script Executable
`chmod +x /scripts/official_backup.sh`

screenshot:`chmod-script`
<img width="1125" height="857" alt="image" src="https://github.com/user-attachments/assets/b2d592da-4ef8-4a19-bda5-950a9bb530f5" />

## Step 7: Execute Backup Script
RUN SCRIPT:
    `/scripts/official_backup.sh`

screenshot:`script-execution`
<img width="1150" height="865" alt="image" src="https://github.com/user-attachments/assets/5c700f07-a37e-4752-a377-cadb9f17472e" />

## Step 8: Verify Local Backup
CHECK FILE EXISTS:
    `ls -l /backup/xfusioncorp_official.zip`

screenshot:`local-backup-verified`
<img width="1133" height="857" alt="image" src="https://github.com/user-attachments/assets/a6d3003d-4da9-49de-be9c-0784ce8598f3" />

## Step 9: Verify Remote Backup
CHECK FILE ON BACKUP SERVER:
    `ssh clint@172.16.238.16 "ls -l /backup/xfusioncorp_official.zip"`

screenshot:`remote-backup-verified`
<img width="1158" height="857" alt="image" src="https://github.com/user-attachments/assets/2ba4286f-af62-4fbd-83f9-5ec983b10a8b" />

## ✅ Validation Checklist

- ZIP archive created successfully
- Backup stored in `/backup` on App Server 2
- Backup copied to Nautilus Backup Server
- No password prompt during SCP
- Script runs without sudo
- Script located under `/scripts`

## Tags
`linux`
`bash`
`backup`
`ssh`
`automation`
`production-support`
