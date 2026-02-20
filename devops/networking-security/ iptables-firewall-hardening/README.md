# iptables Firewall Configuration for Application Servers

---

## Project Overview

- This project secures application servers by configuring host-based firewall rules using iptables.

- The objective was to:
  -  Install and enable iptables on all application servers
  -  Restrict access to application port 5001
  -  Allow traffic only from the LBR host
  -  Persist firewall rules across reboots

- Servers configured:
  -  stapp01
  -  stapp02
  -  stapp03

---

## Security Requirement

- CURRENT STATE:
    - Application port 5001 was publicly accessible
    - No firewall rules were enforced

- TARGET STATE:
    - Port 5001 accessible ONLY from LBR host (172.16.238.14)
    - All other incoming traffic to port 5001 blocked
    - SSH access (port 22) preserved
    - Rules persistent after reboot

---

## Tools Used

- iptables
- iptables-services
- systemctl
- SSH
- CentOS Stream 9

---

## Implementation Workflow

---

### Step 1: Access Application Server

- ACTION:
  -  Connect to application server via jump host

- COMMAND:
  -  `ssh <user>@<stapp-server>`

SCREENSHOT: `SSH login to application server`
<img width="1027" height="622" alt="image" src="https://github.com/user-attachments/assets/a2e9bf81-345b-4b32-8ca5-66055266d94a" />

---

### Step 2: Switch to Root User

- ACTION:
    Gain administrative privileges

- COMMAND:
   - `sudo -i`

SCREENSHOT: `sudo root access`
<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/49f1d81f-da73-418e-a909-8b5974f3bae6" />

---

### Step 3: Install iptables Services

- ACTION:
  -  Install iptables persistence service

- COMMAND:
  -  `yum install iptables-services -y`

- EXPECTED RESULT:
  -  iptables-services package installed successfully

SCREENSHOT: `iptables-services installation`
<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/3d59cac2-ea71-4e19-8d47-f0b79e2f45ed" />
<img width="1033" height="605" alt="image" src="https://github.com/user-attachments/assets/b60f3bae-d04d-4563-b003-697c4f88cd08" />

---

### Step 4: Enable and Start iptables Service

- ACTION:
  -  Ensure iptables starts automatically on boot

- COMMAND:
  -  `systemctl enable iptables`
  -  `systemctl start iptables`

SCREENSHOT: `iptables service enabled and started`
<img width="1032" height="486" alt="image" src="https://github.com/user-attachments/assets/d3f0cfd9-cf2d-458e-8a77-6710f5fb58cf" />

---

### Step 5: Flush Existing Firewall Rules

- ACTION:
  -  Clear any pre-existing firewall rules

- COMMAND:
  -  `iptables -F`

SCREENSHOT: `iptables rules flushed`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/c04dff37-6e1d-4dd3-9a80-ee6f1a0ba3f7" />

---

### Step 6: Allow SSH Access

- ACTION:
    `Ensure SSH access is not interrupted`

- COMMAND:
  -  `iptables -A INPUT -p tcp --dport 22 -j ACCEPT`

SCREENSHOT: `SSH allow rule added`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/7aa8dcad-4807-46f9-b1bc-c2b6b480133a" />

---

### Step 7: Allow LBR Host to Access Application Port

- ACTION:
  -  Permit application traffic from Load Balancer host only

- LBR IP:
    `172.16.238.14`

- COMMAND:
  -  `iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT`

SCREENSHOT: `LBR allow rule for port 5001`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/42f8cad9-870d-4dd6-b811-2c063b8f1ebd" />

---

### Step 8: Block All Other Access to Application Port

- ACTION:
  -  Deny application port access from all other sources

- COMMAND:
  -  `iptables -A INPUT -p tcp --dport 5001 -j DROP`

SCREENSHOT: `DROP rule for port 5001`
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/e3120de9-c9c1-4dc6-85ed-f7614948fe03" />

---

### Step 9: Save Firewall Rules

- ACTION:
  -  Persist firewall rules across system reboots

- COMMAND:
  -  `service iptables save`

- EXPECTED RESULT:
  -  Rules saved to `/etc/sysconfig/iptables`

SCREENSHOT: `iptables rules saved`
<img width="1029" height="597" alt="image" src="https://github.com/user-attachments/assets/d4a5357e-210f-4a9c-82a7-03bd7880f305" />

---

### Step 10: Repeat on All Application Servers

- ACTION:
  -  Repeat Steps 1–9 on:
      -  stapp01
      -  stapp02
      -  stapp03

SCREENSHOTS: `firewall applied on all app servers`
<img width="1037" height="716" alt="image" src="https://github.com/user-attachments/assets/766ddd96-ad0d-41bc-9145-1f548bb9893e" />
<img width="1033" height="862" alt="image" src="https://github.com/user-attachments/assets/7a4c773f-2432-41af-99f6-9ce24f687176" />
<img width="1027" height="513" alt="image" src="https://github.com/user-attachments/assets/9f21b848-5041-4056-bb9a-704147583482" />
<img width="1036" height="566" alt="image" src="https://github.com/user-attachments/assets/aad83a94-248a-49af-bc06-f80af275989d" />
<img width="1038" height="607" alt="image" src="https://github.com/user-attachments/assets/46d656cf-8dcb-421d-9c27-3a567b02575d" />
<img width="1034" height="465" alt="image" src="https://github.com/user-attachments/assets/3467c84b-84c3-4616-90bb-17331a81459c" />
<img width="1029" height="659" alt="image" src="https://github.com/user-attachments/assets/7b99f563-46a3-463f-aa1a-5fb11fe3ba78" />
<img width="1026" height="863" alt="image" src="https://github.com/user-attachments/assets/dacde58b-4d4d-415d-b636-307965b81811" />
<img width="1033" height="616" alt="image" src="https://github.com/user-attachments/assets/665b84ee-a815-4eea-b4a8-50b7de1f885e" />
<img width="1031" height="483" alt="image" src="https://github.com/user-attachments/assets/5f3baf8d-20cc-4893-abc0-6f178f6386cb" />
<img width="1033" height="600" alt="image" src="https://github.com/user-attachments/assets/c746837e-10cc-44d9-9c05-f4614bb58ba0" />
<img width="1034" height="690" alt="image" src="https://github.com/user-attachments/assets/d5123376-a85c-4d7f-b938-81888e2539b9" />

---

## Final Outcome

- iptables installed and enabled on all app servers
- Port 5001 secured and restricted to LBR host
- SSH access maintained
- Firewall rules persisted across reboots
- Infrastructure security posture improved
