# High Availability Nginx Load Balancer Setup (Nautilus – Stratos DC)

## Overview

- Due to increasing traffic and performance degradation on a production website managed by the Nautilus Production Support Team, the application was migrated to a high-availability architecture in the Stratos Data Center.

- This project focuses on the final migration step: configuring an Nginx-based HTTP Load Balancer (LBR) to distribute traffic across multiple Apache application servers without modifying existing backend ports, following real-world production constraints.

## Architecture

- Client
   |
   v
- Nginx Load Balancer (stlb01)
   |
   +-- App Server 1 (stapp01:5004)
   +-- App Server 2 (stapp02:5004)
   +-- App Server 3 (stapp03:5004)
   
## Infrastructure Details
| Role          | Hostname | IP Address    | Service            |
| ------------- | -------- | ------------- | ------------------ |
| Load Balancer | stlb01   | 172.16.238.14 | Nginx              |
| App Server 1  | stapp01  | 172.16.238.10 | Apache (Port 5004) |
| App Server 2  | stapp02  | 172.16.238.11 | Apache (Port 5004) |
| App Server 3  | stapp03  | 172.16.238.12 | Apache (Port 5004) |
| Jump Host     | jumphost | Dynamic       | SSH Access         |

## Objectives

- Install and configure Nginx on the Load Balancer

- Configure HTTP load balancing using the main config file:

`/etc/nginx/nginx.conf`

- Preserve existing Apache ports on backend servers

- Validate service health and traffic routing

- Access application via StaticApp URL

## Implementation Steps

## Step 1: Access the Load Balancer
`ssh loki@172.16.238.14`

📸 Screenshot: `SSH login to Load Balancer (stlb01)`
<img width="1021" height="183" alt="image" src="https://github.com/user-attachments/assets/31ddbf55-3e92-4f88-b5af-f1544a73e02c" />

## Step 2: Install & Enable Nginx
- sudo yum install -y nginx
- sudo systemctl enable nginx
- sudo systemctl start nginx
- sudo systemctl status nginx

📸 Screenshot: `Nginx installation and active service status`
<img width="1039" height="859" alt="image" src="https://github.com/user-attachments/assets/2f23828e-6ee0-4c17-b988-9de4310d0e9a" />
<img width="1028" height="853" alt="image" src="https://github.com/user-attachments/assets/568108e7-fd09-481d-87e5-a0ae00c39a33" />
<img width="1041" height="181" alt="image" src="https://github.com/user-attachments/assets/95b84ad9-7d70-464d-86b1-e342fbd8584e" />
<img width="1031" height="862" alt="image" src="https://github.com/user-attachments/assets/a7678bd7-bc2d-4896-81e9-93756376f015" />

## Step 3: Verify Apache on App Servers (No Port Changes)

- Apache was already configured to run on port 5004 and must not be modified.

- Verification performed on all app servers:

`sudo systemctl status httpd`
`sudo ss -lntp | grep httpd`

Expected output:

LISTEN 0 511 0.0.0.0:5004

📸 Screenshot: `Apache running and listening on port 5004`

## Step 4: Configure Nginx Load Balancing

⚠️ `Only /etc/nginx/nginx.conf was modified as required.`

`sudo vi /etc/nginx/nginx.conf`

## Final http {} Configuration (Relevant Section)

http {

    upstream app_servers {
        server 172.16.238.10:5004;
        server 172.16.238.11:5004;
        server 172.16.238.12:5004;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://app_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

📸 Screenshot: `Nginx upstream and server block configuration`

## Step 5: Validate & Reload Nginx
`sudo nginx -t`
`sudo systemctl reload nginx`

Expected result:

syntax is `ok`
test is successful

📸 Screenshot: `Successful Nginx configuration test`

## Application Validation

- The application was successfully accessed via the StaticApp URL provided by the environment:

- `http://80-port-<environment-id>.labs.kodekloud.com`

- Load balancer responded on port 80
- Requests routed to backend Apache servers
- No 502 / 504 errors observed

📸 Screenshot: `Application loading successfully via StaticApp URL`

## Key Troubleshooting Insight

- A 502 Bad Gateway error was initially encountered due to Nginx forwarding traffic to port 80, while Apache was running on port 5004.

Resolution:

- Validated Apache listening ports using ss -lntp

- Updated Nginx upstream servers to include the correct port

- Reloaded Nginx without downtime

## Final Outcome

- Nginx installed and running on Load Balancer

- Load balancing configured using main config file

- Apache ports preserved as required

- Traffic successfully distributed across app servers

- High-availability architecture fully operational

## Skills Demonstrated

- Linux system administration

- Nginx reverse proxy & load balancing

- Production-safe configuration management

- Network troubleshooting (502 Bad Gateway)

- HA infrastructure validation

- DevOps documentation best practices

## Notes

- This setup mirrors real-world enterprise environments where:

- Backend ports cannot be changed

- Bastion (jump) hosts are required

- Configuration changes must be validated before reloads




<img width="1038" height="861" alt="image" src="https://github.com/user-attachments/assets/2fc983e2-bc90-4232-b781-6a3e64de1d98" />
<img width="1033" height="854" alt="image" src="https://github.com/user-attachments/assets/355640a1-9ba2-4886-b73e-daf5055cc02c" />
<img width="1031" height="841" alt="image" src="https://github.com/user-attachments/assets/ffdbf250-7ddf-45ca-8223-3ebce461e9fa" />
<img width="1033" height="858" alt="image" src="https://github.com/user-attachments/assets/f815f010-ff60-435e-bb25-7691b40eae34" />
<img width="1038" height="487" alt="image" src="https://github.com/user-attachments/assets/feee1d48-774c-46ed-ae04-bf92109c97c5" />
<img width="1019" height="867" alt="image" src="https://github.com/user-attachments/assets/bc473fd7-2a06-4687-8c7e-e997243e7684" />
<img width="1034" height="862" alt="image" src="https://github.com/user-attachments/assets/2855ea06-b822-4561-a336-3b770e5da9ba" />
<img width="1030" height="699" alt="image" src="https://github.com/user-attachments/assets/0965d1d1-b11c-4cd9-8e62-b02cf011cce8" />
<img width="1664" height="1006" alt="image" src="https://github.com/user-attachments/assets/7ad29568-fd91-4d75-8b72-237f106617a0" />
