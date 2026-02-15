# MariaDB Service Recovery – Nautilus Application (Stratos DC)

## 📌 Lab Overview
- A production outage was detected in the Nautilus application hosted in Stratos DC.
The application failed to connect to its database due to the MariaDB service being DOWN
on the Nautilus database server (stdb01).

- This task restores database availability by fixing permissions, restarting the service,
and validating successful recovery.

## 🎯 Objectives
- Access Nautilus database server via jump host
- Verify MariaDB data directory ownership
- Correct MariaDB file permissions
- Start and enable MariaDB service
- Confirm service is running successfully

## 🧠 High-Level Logic
- CONNECT to jump host
- SSH into database server (stdb01)

- IF MariaDB data directory ownership is incorrect:
  -  FIX ownership to mysql:mysql

- START MariaDB service
- ENABLE MariaDB to persist after reboot
- VERIFY MariaDB service status is active

- CONFIRM database service recovery

## 🛠️ Implementation Steps

## Step 1: Connect to Database Server (stdb01)
- LOGIN to jumphost as thor
- FROM jumphost:
  -  SSH into 172.16.239.10 as user peter
  -  AUTHENTICATE using provided credentials


screenshot: `db-server-ssh-login`
<img width="1034" height="549" alt="image" src="https://github.com/user-attachments/assets/ede259d2-c69d-4e5c-8cc0-057129e9f87a" />

## Step 2: Verify MariaDB Data Directory Ownership
- CHECK ownership of `/var/lib/mysql`

 -EXPECTED:
  -  owner = `mysql`
  -  group = `mysql`


screenshot: `mysql-directory-permissions`
<img width="1018" height="582" alt="image" src="https://github.com/user-attachments/assets/86beddd4-614f-4066-a307-22df36e6f12d" />

## Step 3: Fix MariaDB Directory Permissions
- IF ownership is incorrect:
  -  CHANGE ownership recursively to mysql:mysql

`sudo chown -R mysql:mysql /var/lib/mysql`


screenshot: `mysql-permission-fix`
<img width="1031" height="497" alt="image" src="https://github.com/user-attachments/assets/73ccd9b6-16fa-4dae-90ad-35b7f2ebae02" />

## Step 4: Start MariaDB Service
- START mariadb service using systemctl

`sudo systemctl start mariadb`


screenshot: 'mariadb-service-start'
<img width="1036" height="551" alt="image" src="https://github.com/user-attachments/assets/c2d018cd-76f3-4896-b5ce-b37d0cef8add" />

## Step 5: Enable MariaDB at Boot
- ENABLE mariadb service to persist across reboots

`sudo systemctl enable mariadb`

screenshot: 'mariadb-enable-service'
<img width="1036" height="551" alt="image" src="https://github.com/user-attachments/assets/c2d018cd-76f3-4896-b5ce-b37d0cef8add" />

## Step 6: Verify MariaDB Service Status
- CHECK mariadb service status

`sudo systemctl status mariadb`

- EXPECTED STATE:
  -  Active: active (running)


screenshot: 'mariadb-service-status'
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/17324216-b1f1-4a09-afe2-5d00c6054385" />

## ✅ Final Outcome

- MariaDB data directory ownership corrected
- MariaDB service successfully started
- Service enabled to persist on reboot
- Database server restored to operational state
- Nautilus application database connectivity recovered

## 🏷️ Tags
`linux`
`troubleshooting`
`mariadb`
`database-recovery`
`systemd`
`production-incident`
`devops`
`infrastructure-operations`
