# Linux Cron Job Scheduling (cronie)

## 📌 Lab Overview
The Nautilus system administration team is preparing to deploy
automation scripts across multiple application servers. Before
scheduling production jobs, a sample cron job was configured
to validate cron functionality across all servers.

---

## 🎯 Objectives
- Install the `cronie` package
- Start and enable the `crond` service
- Schedule a recurring cron job for the root user
- Verify cron execution output

---

## 🧱 Environment

- Jump Host: jumphost

- App Servers: `stapp01` `stapp02` `stapp03`

- OS: CentOS Stream 9

- Scheduler: cron (cronie package)

## 🧠 High-Level Logic
- CONNECT to jump host
- FOR each app server:
  -  INSTALL cron service
  -  START and ENABLE cron daemon
  -  CONFIGURE root cron job
  -  VERIFY cron execution

## 🛠️ Implementation Steps
⚠️ `The following steps were performed individually on each app server.`

## Step 1: SSH into the Server
- ssh tony@stapp01
- ssh steve@stapp02
- ssh banner@stapp03

📸 screenshots:
<img width="1034" height="666" alt="image" src="https://github.com/user-attachments/assets/19c263a8-0787-4e9e-981a-c0b45835cfe3" />
<img width="1039" height="866" alt="image" src="https://github.com/user-attachments/assets/aa017631-f3cd-49fd-bd94-e4547f322de7" />
<img width="1036" height="876" alt="image" src="https://github.com/user-attachments/assets/58f777cb-8792-4d6a-bea3-3e193bde2489" />

## Step 2: Install and Start Cron
- Install the Package
  -  sudo yum install -y cronie
- Start and Enable the Service
- `Start the crond service and enable it so it persists after a reboot`
  -  sudo systemctl start crond
  -  sudo systemctl enable crond

📸 screenshots:
<img width="1036" height="851" alt="image" src="https://github.com/user-attachments/assets/2f546048-df6a-4433-bb28-4366f66b7506" />
<img width="1034" height="853" alt="image" src="https://github.com/user-attachments/assets/58da6179-0709-467f-96fc-4f061833f44d" />

## Step 3: Add the Cron Job
- sudo yum install -y cronie

📸 screenshots:

## Step 4: Verification
- sudo systemctl start crond
- sudo systemctl enable crond

📸 screenshots:

## Step 5: Configure Root Cron Job
sudo crontab -e
Cron entry:

*/5 * * * * echo hello > /tmp/cron_text

📸 screenshots:

## Step 6: Verify Cron Execution
- sudo crontab -l
- ls -l /tmp/cron_text
- cat /tmp/cron_text

📸 screenshots:


## ✅ Final Outcome
- Cron service installed and running

- Root cron job scheduled every 5 minutes

- Output successfully written to /tmp/cron_text

- Configuration applied across all app servers

## 🏷️ Tags
`linux` `cron` `cronie` `system-administration` `automation`



<img width="1042" height="862" alt="image" src="https://github.com/user-attachments/assets/a715db10-e659-48f9-af15-884f9d19fa0d" />
<img width="1035" height="882" alt="image" src="https://github.com/user-attachments/assets/067a0da0-3b97-45a2-b0f2-1830500208c4" />
<img width="1026" height="859" alt="image" src="https://github.com/user-attachments/assets/7577a943-68d8-46c1-a55b-445a6f01d243" />



<img width="1036" height="869" alt="image" src="https://github.com/user-attachments/assets/7e5bc2a2-ab8f-49e0-8086-f22f134d5741" />
<img width="1038" height="890" alt="image" src="https://github.com/user-attachments/assets/cafad6e7-1a35-4a41-9509-41c779dcb83c" />
<img width="1039" height="871" alt="image" src="https://github.com/user-attachments/assets/a33c92bb-3fe2-4353-9e16-04dc3eda3b76" />



<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/3c8748f4-1a76-4898-a98c-3a8ce8ba05f1" />
<img width="1037" height="876" alt="image" src="https://github.com/user-attachments/assets/9401e24a-6951-4b21-a11f-f31d47906e4b" />
<img width="1027" height="857" alt="image" src="https://github.com/user-attachments/assets/4d9e2505-4efe-474f-af0d-fe9a95fcc839" />


