# Static Websites Deployment on App Server 2 – xFusionCorp Industries

## Project Overview

- This project documents the deployment of two static websites (ecommerce and demo) on App Server 2 (stapp02) in the Stratos Datacenter. Apache HTTP Server is installed and configured to serve the websites on custom port 8085.

- Website backups are initially stored on the jump_host and need to be migrated to stapp02 before deployment. The objective is to ensure local accessibility of the websites via curl commands.

## Architecture Summary

             ┌───────────────┐
             │ Jump Host     │
             │ (thor@jump)  │
             └───────┬──────┘
                     │ SCP Transfer
                     ▼
             ┌───────────────┐
             │ App Server 2  │
             │ stapp02       │
             │ Apache:8085   │
             └───────┬──────┘
                     │
      ┌──────────────┴──────────────┐
      │ Web Root (/var/www/html)    │
      │ ┌───────────┐  ┌─────────┐ │
      │ │ ecommerce │  │ demo    │ │
      │ └───────────┘  └─────────┘ │
      └────────────────────────────┘

## 🔧 Technologies Used

- Apache HTTP Server (httpd)

- CentOS Stream 9

- Linux Systemd Services

- SCP (Secure Copy Protocol)

- Bash Shell Scripting

## Step 1: Transfer Website Backups from Jump Host to App Server 2

### 1.1 Execute SCP Commands from Jump Host (thor@jump_host)

- Transfer the ecommerce backup
 
`scp -r /home/thor/ecommerce steve@172.16.238.11:/tmp/`

- Transfer the demo backup

`scp -r /home/thor/demo steve@172.16.238.11:/tmp/`

📸 Screenshot: `SCP transfer from jump_host to stapp02`
<img width="1028" height="593" alt="image" src="https://github.com/user-attachments/assets/3192f0e8-070b-4298-a452-e75709d4f092" />

### 1.2 SSH into App Server 2

`ssh steve@172.16.238.11`

📸 Screenshot: `SCP transfer from jump_host to stapp02`
<img width="1032" height="607" alt="image" src="https://github.com/user-attachments/assets/8db66fff-f303-4a49-a3ca-367abd851bab" />

## Step 2: Install Apache HTTP Server
- `sudo yum install -y httpd`

📸 Screenshot: `Apache installation`
<img width="1031" height="862" alt="image" src="https://github.com/user-attachments/assets/c88f4c76-b8d6-4624-8bef-3e85b080c884" />
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/a593cf76-4f93-4b93-97d8-4e90bb2b8231" />

## Step 3: Configure Apache to Listen on Port 8085

### Update the listening port from 80 to 8085
`sudo sed -i 's/Listen 80/Listen 8085/g' /etc/httpd/conf/httpd.conf`

### Verify the change
`grep "Listen 8085" /etc/httpd/conf/httpd.conf`

📸 Screenshot: `Apache port configuration`

<img width="1031" height="867" alt="image" src="https://github.com/user-attachments/assets/306bd313-3d6a-486b-aa65-295b0dbe76a6" />

## Step 4: Start and Enable Apache Service
````
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl restart httpd
````

📸 Screenshot:`Apache start and enable`

## Step 5: Move Websites to DocumentRoot
````
sudo mv /tmp/ecommerce /var/www/html/
sudo mv /tmp/demo /var/www/html/
````

## Step 6: Set Ownership and Permissions
````
sudo chown -R apache:apache /var/www/html/ecommerce
sudo chown -R apache:apache /var/www/html/demo
````

📸 Screenshot Placeholder:
![Move websites and set ownership](./screenshots/move_ownership.png)

## Step 7: Validate Local Access via Curl

### 7.1 Ecommerce Site
curl http://localhost:8085/ecommerce/

### 7.2 Demo Site
curl http://localhost:8085/demo/

Expected Output: HTML content of respective websites

📸 Screenshot: `Curl validation - ecommerce`
<img width="1030" height="565" alt="image" src="https://github.com/user-attachments/assets/1dced30a-cbc0-44f5-89f8-90c69b0780da" />

📸 Screenshot: `Curl validation - demo`
<img width="1025" height="755" alt="image" src="https://github.com/user-attachments/assets/1d252416-c8b9-4b3d-ad29-9e75cdfbf708" />

## ✅ Validation Checklist
| **Check**                      | **Status** |
| ------------------------------ | ---------- |
| Apache installed & running     | ✅          |
| Apache listening on port 8085  | ✅          |
| Website backups transferred    | ✅          |
| Websites moved to DocumentRoot | ✅          |
| Ownership & permissions set    | ✅          |
| Websites reachable via curl    | ✅          |


## Final Outcome

- Two static websites deployed on stapp02

Websites accessible locally on:

- `http://localhost:8085/ecommerce/`

- `http://localhost:8085/demo/`

- Apache configured on custom `port 8085`

- Ownership and permissions configured for proper access

- Ready for further integration with production environment

## Key Learnings

- SCP is essential for transferring files across servers

- Apache allows serving multiple websites via subdirectories

- Correct ownership and permissions prevent common access errors

- Custom ports require explicit configuration in `httpd.conf`






<img width="1036" height="472" alt="image" src="https://github.com/user-attachments/assets/578240d7-883c-4421-94f0-77db2aea525c" />
<img width="1036" height="429" alt="image" src="https://github.com/user-attachments/assets/3567b2fe-b0c4-4f96-b55b-6a13aef6835a" />
<img width="1033" height="566" alt="image" src="https://github.com/user-attachments/assets/3f9fefa7-6ffd-43dd-8a0a-80448f67a547" />




