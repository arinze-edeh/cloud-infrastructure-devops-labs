# Secure Nginx HTTPS Deployment on App Server 1

## Project Overview
- This project demonstrates preparing a Linux server for production-grade web service deployment.  
- It involves installing and configuring **Nginx**, deploying a **self-signed SSL certificate**, and validating HTTPS access from a remote host.

## Key Outcomes:
- Nginx installed and configured on App Server 1
- SSL/TLS enabled using self-signed certificate
- Landing page deployed under Nginx document root
- Remote HTTPS access validated from jump host

---

## Infrastructure Details

| Component       | Details                          |
|-----------------|---------------------------------|
| Server Name     | App Server 1                    |
| Hostname        | stapp01.stratos.xfusioncorp.com |
| IP Address      | 172.16.238.10                   |
| OS User         | tony                             |
| Web Server      | Nginx 1.20.1                    |
| SSL Certificate | Self-Signed (nautilus.crt/key)  |
| Purpose         | Deploy and test secure web service |

---

## Step 1: SSH to App Server 1

- `ssh tony@172.16.238.10`

Accept host key

Authenticate with password

📸 Screenshot: SSH login
<img width="1028" height="632" alt="image" src="https://github.com/user-attachments/assets/304d9926-ff38-4586-b15d-21effecc1561" />

## Step 2: Install Nginx
- `sudo yum install -y nginx`

Verify installation:

`nginx -v`

📸 Screenshot: `Nginx version output`
<img width="1034" height="859" alt="image" src="https://github.com/user-attachments/assets/87826513-204d-415b-80d5-af62bedc0c57" />
<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/452cdda5-67ca-4a9c-9510-fbb7f3423977" />

## Step 3: Enable and Start Nginx
- `sudo systemctl enable nginx`
- `sudo systemctl start nginx`
- `sudo systemctl status nginx`

📸 Screenshot: `Nginx service running`

## Step 4: Deploy SSL Certificate

Create directories for certs and keys:

`sudo mkdir -p /etc/pki/tls/certs /etc/pki/tls/private`

Move the self-signed certificate and key:

`sudo mv /tmp/nautilus.crt /etc/pki/tls/certs/`
`sudo mv /tmp/nautilus.key /etc/pki/tls/private/`

📸 Screenshot: SSL files in place


## Step 5: Configure Nginx for HTTPS

Edit the main Nginx configuration file:

`sudo vi /etc/nginx/nginx.conf`

Add or update SSL server block:

server {
    listen 443 ssl;
    server_name stapp01.stratos.xfusioncorp.com;

    ssl_certificate /etc/pki/tls/certs/nautilus.crt;
    ssl_certificate_key /etc/pki/tls/private/nautilus.key;

    root /usr/share/nginx/html;
    index index.html;
}

Test configuration:

`sudo nginx -t`

📸 Screenshot: `Nginx configuration test successful`


## Step 6: Deploy Landing Page
`echo "Welcome!" | sudo tee /usr/share/nginx/html/index.html`

📸 Screenshot: `index.html content`


## Step 7: Restart Nginx
`sudo systemctl restart nginx`

📸 Screenshot: `Nginx restarted successfully`

## Step 8: Validate HTTPS Access from Jump Host

Exit server:

`exit`

From jump host, run:

`curl -Ik https://172.16.238.10/`

Expected Output:

HTTP/2 200
server: nginx/1.20.1
content-type: text/html
content-length: 9

📸 Screenshot: `curl HTTPS response headers`


## Outcome & Skills Demonstrated

- Linux Administration: `SSH, yum package management`

- Nginx Web Server Configuration

- SSL/TLS Deployment (self-signed)

- Systemd service management

- Remote Infrastructure Validation using `curl`



<img width="1037" height="692" alt="image" src="https://github.com/user-attachments/assets/0238cb83-3818-4b48-b8fa-d2ffccc5d8d7" />
<img width="1031" height="759" alt="image" src="https://github.com/user-attachments/assets/d4860fee-388e-4049-82ae-f74ab827139f" />
<img width="896" height="885" alt="image" src="https://github.com/user-attachments/assets/7f7a0a97-6f19-4e3d-8b4c-25775c212748" />
<img width="1036" height="824" alt="image" src="https://github.com/user-attachments/assets/20cd2bd9-d532-4aff-9cee-d142503add67" />
<img width="1031" height="391" alt="image" src="https://github.com/user-attachments/assets/8260e090-27be-4e7d-a344-18258052881b" />
<img width="1035" height="342" alt="image" src="https://github.com/user-attachments/assets/c22cedd0-3851-40c8-afd4-989b424278e5" />
<img width="1033" height="575" alt="image" src="https://github.com/user-attachments/assets/172cf54f-d96a-4767-9a85-8e8fd0ab8a30" />

