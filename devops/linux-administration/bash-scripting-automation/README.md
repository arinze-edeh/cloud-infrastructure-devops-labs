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

screenshot:`zip-installation`
<img width="1158" height="857" alt="image" src="https://github.com/user-attachments/assets/3423e8fc-6546-4a5f-8f0b-1168b60afe1d" />
<img width="1153" height="865" alt="image" src="https://github.com/user-attachments/assets/86b041c3-e4fa-453e-8d27-7840d276c8ce" />

## Step 3: Configure Passwordless SSH to Backup Server
GENERATE SSH KEY:
    `ssh-keygen -t rsa`

COPY SSH KEY TO BACKUP SERVER:
  -  `ssh-copy-id clint@172.16.238.16`

VERIFY PASSWORDLESS LOGIN:
  -  `ssh clint@172.16.238.16`

screenshot:`ssh-key-auth-success`

## Step 4: Prepare Required Directories
- CREATE /scripts DIRECTORY:
  -  sudo mkdir -p /scripts
  -  sudo chown steve:steve /scripts

- CREATE /backup DIRECTORY:
  -  sudo mkdir -p /backup
  -  sudo chown steve:steve /backup

screenshot:`directories-created`

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
      -  zip -r /backup/xfusioncorp_official.zip /var/www/html/official

  -  COPY ARCHIVE TO BACKUP SERVER
      -  scp /backup/xfusioncorp_official.zip clint@172.16.238.16:/backup

END

NOTE:
- NO sudo inside script
- Script must be executable

screenshot:`script-content`

## Step 6: Make Script Executable
`chmod +x /scripts/official_backup.sh`

screenshot:`chmod-script`

## Step 7: Execute Backup Script
RUN SCRIPT:
    `/scripts/official_backup.sh`

screenshot:`script-execution`

## Step 8: Verify Local Backup
CHECK FILE EXISTS:
    `ls -l /backup/xfusioncorp_official.zip`

screenshot:`local-backup-verified`

## Step 9: Verify Remote Backup
CHECK FILE ON BACKUP SERVER:
    `ssh clint@172.16.238.16 "ls -l /backup/xfusioncorp_official.zip"`

screenshot:`remote-backup-verified`
<img width="1158" height="857" alt="image" src="https://github.com/user-attachments/assets/2ba4286f-af62-4fbd-83f9-5ec983b10a8b" />

## ✅ Validation Checklist

- ZIP archive created successfully
- Backup stored in /backup on App Server 2
- Backup copied to Nautilus Backup Server
- No password prompt during SCP
- Script runs without sudo
- Script located under /scripts

## Tags
`linux`
`bash`
`backup`
`ssh`
`automation`
`production-support`





<img width="1152" height="859" alt="image" src="https://github.com/user-attachments/assets/ca4095f2-42c1-4b83-ba49-f4c291fcd656" />
<img width="1155" height="871" alt="image" src="https://github.com/user-attachments/assets/65d0ab02-c335-472f-a9c3-2ba92fa300ed" />
<img width="1151" height="853" alt="image" src="https://github.com/user-attachments/assets/c25d7bf4-cd3b-4970-8955-b370e6def799" />
<img width="1155" height="847" alt="image" src="https://github.com/user-attachments/assets/0e361e15-d992-4aee-9f68-124de8b5c051" />
<img width="1158" height="856" alt="image" src="https://github.com/user-attachments/assets/797d7c5b-0209-44ca-ba1c-f66d266b3ff8" />
<img width="1144" height="861" alt="image" src="https://github.com/user-attachments/assets/78b39b94-17b6-4855-879d-e11def40c121" />
<img width="1154" height="861" alt="image" src="https://github.com/user-attachments/assets/4d62259d-e92c-418e-8e57-48bf9ad09209" />
<img width="1125" height="857" alt="image" src="https://github.com/user-attachments/assets/b2d592da-4ef8-4a19-bda5-950a9bb530f5" />
<img width="1146" height="733" alt="image" src="https://github.com/user-attachments/assets/9ee47ab7-fc39-4494-9f15-b146460bdcce" />
<img width="1150" height="865" alt="image" src="https://github.com/user-attachments/assets/5c700f07-a37e-4752-a377-cadb9f17472e" />
<img width="1133" height="857" alt="image" src="https://github.com/user-attachments/assets/a6d3003d-4da9-49de-be9c-0784ce8598f3" />

