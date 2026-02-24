# WordPress Multi-Host Deployment on Stratos Datacenter

## Project Overview

This project documents the deployment of a **highly available WordPress application** across multiple application servers with a centralized MariaDB backend inside the **Stratos Datacenter**.
The infrastructure was preconfigured with a **shared NFS volume** mounted at `/var/www/html` across all application hosts, enabling consistent WordPress content delivery behind a Load Balancer (LBR).

The objective was to validate **end-to-end application-to-database connectivity** through the LBR endpoint.

---

## Architecture Summary

```
                  ┌───────────────┐
                  │ Load Balancer │
                  │   (LBR UI)    │
                  └───────┬───────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
 ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
 │ App Server  │   │ App Server  │   │ App Server  │
 │  stapp01    │   │  stapp02    │   │  stapp03    │
 │ Apache:3002 │   │ Apache:3002 │   │ Apache:3002 │
 └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
        │                 │                 │
        └────────── Shared NFS ─────────────┘
                   /var/www/html
                          │
                   ┌──────▼──────┐
                   │ DB Server   │
                   │ MariaDB     │
                   │ kodekloud_db5│
                   └─────────────┘
```

---

## 🔧 Technologies Used

* **Apache HTTP Server (httpd)**
* **PHP 8.x**
* **MariaDB 10.5**
* **CentOS Stream 9**
* **NFS Shared Storage**
* **Linux Systemd Services**
* **Load Balancer (LBR)**

---

## 📂 Shared Storage

* **Storage Server Path:** `/vaw/www/html`
* **Mounted On App Hosts:** `/var/www/html`
* Purpose: Single source of truth for WordPress files across all app servers

---

## 🚀 Implementation Steps

### 1️⃣ Database Server Configuration (stdb01)

#### Install & Enable MariaDB

```bash
sudo yum install -y mariadb-server
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

📸 **Screenshots:**
<img width="1028" height="486" alt="image" src="https://github.com/user-attachments/assets/6f26cdb4-4926-48b4-8417-3418eab7eec0" />
<img width="1036" height="859" alt="image" src="https://github.com/user-attachments/assets/0a44a21e-5397-432d-9309-e824aef942b5" />
<img width="1032" height="863" alt="image" src="https://github.com/user-attachments/assets/cffbbb38-fa55-4346-9b68-043da2c5774f" />
<img width="1032" height="655" alt="image" src="https://github.com/user-attachments/assets/f03b1e76-4ebf-44d2-98c1-81838af57a7b" />


---

#### Database & User Setup

```sql
CREATE DATABASE kodekloud_db5;
CREATE USER 'kodekloud_cap'@'%' IDENTIFIED BY 'ksH85UJjhb';
GRANT ALL PRIVILEGES ON kodekloud_db5.* TO 'kodekloud_cap'@'%';
FLUSH PRIVILEGES;
```

📸 **Screenshot:**
<img width="1032" height="598" alt="image" src="https://github.com/user-attachments/assets/60cdf792-9b8e-4be7-917e-fc385bfded1a" />

---

### 2️⃣ Application Servers Setup (stapp01, stapp02, stapp03)

#### Install Apache & PHP Dependencies

```bash
sudo yum install -y httpd php php-mysqlnd php-gd php-mbstring
```

📸 **Screenshot Placeholder:**
`docs/screenshots/php-httpd-install.png`

---

#### Configure Apache to Listen on Port 3002

```bash
sudo sed -i 's/Listen 80/Listen 3002/g' /etc/httpd/conf/httpd.conf
```

---

#### Start & Enable Apache

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

📸 **Screenshot Placeholder:**
`docs/screenshots/httpd-port-3002.png`

---

### 3️⃣ WordPress Database Configuration

Create `wp-config.php` inside the shared directory:

```bash
sudo tee /var/www/html/wp-config.php <<EOF
<?php
define( 'DB_NAME', 'kodekloud_db5' );
define( 'DB_USER', 'kodekloud_cap' );
define( 'DB_PASSWORD', 'ksH85UJjhb' );
define( 'DB_HOST', '172.16.239.10' );
EOF
```

📸 **Screenshot Placeholder:**
`docs/screenshots/wp-config.png`

---

### 4️⃣ Service Restart (All App Servers)

```bash
sudo systemctl restart httpd
```

---

## 🔍 Validation & Health Checks

### Apache Reachability

```bash
curl -I http://<app-server-ip>:3002
```

**Expected Result**

```
HTTP/1.1 200 OK
Server: Apache/2.4.x
X-Powered-By: PHP
```

📸 **Screenshot Placeholder:**
`docs/screenshots/httpd-healthcheck.png`

---

### LBR Application Verification

* Navigate to **LBR UI**
* Click **App** button on the top bar

✅ **Expected Message**

```
App is able to connect to the database using user kodekloud_cap
```

📸 **Screenshot:**

<img width="1919" height="1024" alt="image" src="https://github.com/user-attachments/assets/561d7ffd-54cd-4f6d-b188-a6297f75e704" />

---

## ✅ Validation Checklist

| Check                              | Status |
| ---------------------------------- | ------ |
| MariaDB installed & running        | ✅      |
| Database created                   | ✅      |
| DB user created & privileged       | ✅      |
| Apache installed on all app hosts  | ✅      |
| Apache listening on port 3002      | ✅      |
| Shared storage mounted             | ✅      |
| WordPress DB connectivity verified | ✅      |
| LBR access successful              | ✅      |

---

## 🎯 Final Outcome

* WordPress successfully deployed across **multiple app servers**
* Database centralized on a **dedicated MariaDB host**
* Application accessible via **Load Balancer**
* Verified **end-to-end database connectivity**
* Architecture supports **horizontal scaling & high availability**

---

## 🧠 Key Learnings

* Multi-tier WordPress deployments require strict **port consistency**
* Shared storage simplifies multi-host content synchronization
* Explicit database grants prevent silent WordPress failures
* LBR validation is the final proof of full-stack correctness
---




<img width="1033" height="853" alt="image" src="https://github.com/user-attachments/assets/2efd2453-0c8d-4708-8a9e-baedb72b0e7b" />
<img width="1031" height="859" alt="image" src="https://github.com/user-attachments/assets/9cc8556c-d09a-4002-84b7-244d31f5129e" />
<img width="1033" height="438" alt="image" src="https://github.com/user-attachments/assets/3422a69d-ced5-44b6-a0b0-4243c91ac126" />
<img width="1041" height="428" alt="image" src="https://github.com/user-attachments/assets/0ba7f6c2-44e2-44f1-8437-c83dd22781b6" />
<img width="1032" height="869" alt="image" src="https://github.com/user-attachments/assets/2ed76a89-04d1-4940-bee9-6845ca7ac8ff" />
<img width="1036" height="865" alt="image" src="https://github.com/user-attachments/assets/44f3586c-07ee-469f-bf6d-50b96c44f16a" />
<img width="1035" height="453" alt="image" src="https://github.com/user-attachments/assets/655b8912-5aa0-4cac-806b-0ee6326fc982" />
<img width="1036" height="867" alt="image" src="https://github.com/user-attachments/assets/17fd52d4-a11d-4952-9696-b2ac3d55d17e" />
<img width="1037" height="865" alt="image" src="https://github.com/user-attachments/assets/1e308fe0-3e38-4ef6-8f40-6a8371d8660d" />
<img width="1031" height="451" alt="image" src="https://github.com/user-attachments/assets/b9745257-3321-4834-b9ac-a16c7928229c" />
<img width="1035" height="467" alt="image" src="https://github.com/user-attachments/assets/c96d5254-0749-4c75-b87b-919f952e7b86" />
<img width="1032" height="293" alt="image" src="https://github.com/user-attachments/assets/27b88a05-1bba-4b23-9344-5345ae9a8388" />
<img width="1025" height="459" alt="image" src="https://github.com/user-attachments/assets/da73f4f8-28a4-4eff-ba16-3dc800b536ce" />
<img width="1034" height="786" alt="image" src="https://github.com/user-attachments/assets/8a3f2a43-11fa-4a36-a789-7c2a9516501f" />


