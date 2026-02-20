# iptables Firewall Configuration for Application Servers

## ENVIRONMENT:
  -  Nautilus Infrastructure
  -  Stratos Datacenter

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

---

### Step 2: Switch to Root User

- ACTION:
    Gain administrative privileges

- COMMAND:
   - `sudo -i`

SCREENSHOT: `sudo root access`

---

### Step 3: Install iptables Services

- ACTION:
  -  Install iptables persistence service

- COMMAND:
  -  `yum install iptables-services -y`

- EXPECTED RESULT:
  -  iptables-services package installed successfully

SCREENSHOT: `iptables-services installation`

---

### Step 4: Enable and Start iptables Service

- ACTION:
  -  Ensure iptables starts automatically on boot

- COMMAND:
  -  `systemctl enable iptables`
  -  `systemctl start iptables`

SCREENSHOT: `iptables service enabled and started`

---

### Step 5: Flush Existing Firewall Rules

- ACTION:
  -  Clear any pre-existing firewall rules

- COMMAND:
  -  `iptables -F`

SCREENSHOT: `iptables rules flushed`

---

### Step 6: Allow SSH Access

- ACTION:
    `Ensure SSH access is not interrupted`

- COMMAND:
  -  `iptables -A INPUT -p tcp --dport 22 -j ACCEPT`

SCREENSHOT: `SSH allow rule added`

---

### Step 7: Allow LBR Host to Access Application Port

- ACTION:
  -  Permit application traffic from Load Balancer host only

- LBR IP:
    `172.16.238.14`

- COMMAND:
  -  `iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT`

SCREENSHOT: `LBR allow rule for port 5001`

---

### Step 8: Block All Other Access to Application Port

- ACTION:
  -  Deny application port access from all other sources

- COMMAND:
  -  `iptables -A INPUT -p tcp --dport 5001 -j DROP`

SCREENSHOT: `DROP rule for port 5001`

---

### Step 9: Save Firewall Rules

- ACTION:
  -  Persist firewall rules across system reboots

- COMMAND:
  -  service iptables save

- EXPECTED RESULT:
  -  Rules saved to `/etc/sysconfig/iptables`

SCREENSHOT: `iptables rules saved`

---

### Step 10: Repeat on All Application Servers

- ACTION:
    Repeat Steps 1–9 on:
        - stapp01
        - stapp02
        - stapp03

SCREENSHOT: `firewall applied on all app servers`

---

## Validation

TEST CASE 1:
    From LBR host:
        Access port 5001 → SUCCESS

TEST CASE 2:
    From any other host:
        Access port 5001 → BLOCKED

TEST CASE 3:
    Reboot server:
        Firewall rules persist → SUCCESS

SCREENSHOT:
    [Screenshot: LBR access successful]
    [Screenshot: non-LBR access blocked]

---

## Final Outcome

- iptables installed and enabled on all app servers
- Port 5001 secured and restricted to LBR host
- SSH access maintained
- Firewall rules persisted across reboots
- Infrastructure security posture improved
