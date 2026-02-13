# Password-less SSH Access from Jump Host to App Servers

## 📌 Lab Overview
The xFusionCorp system administration team configured password-less SSH
authentication from the jump host to all Nautilus application servers.
This setup ensures automation scripts can execute securely and
uninterrupted across servers without manual password entry.

---

## 🎯 Objectives
- Generate SSH key pair on jump host
- Distribute SSH public key to app servers
- Enable password-less SSH authentication
- Verify remote access using hostname checks

---

## 🧠 High-Level Logic

- LOGIN to jump host as thor

- IF SSH key does not exist:
  -  GENERATE RSA SSH key pair

- FOR each application server:
  -  COPY SSH public key to sudo user
  -  ACCEPT host authenticity prompt
  -  AUTHENTICATE once using password

- FOR each application server:
  -  SSH into server without password
  -  RUN hostname command to confirm access

## 🛠️ Implementation Steps

## Step 1: Login to Jump Host
- ssh thor@jump_host.stratos.xfusioncorp.com

📸 screenshot:

## Step 2: Generate SSH Key Pair
- ssh-keygen -t rsa

📸 screenshot:

## Step 3: Verify SSH Key Files
ls -l ~/.ssh/id_rsa*
📸 screenshots/ssh-key-files.png

## Step 4: Copy SSH Key to App Server 1
ssh-copy-id tony@172.16.238.10
Accept authenticity prompt

Enter user password once

📸 screenshots/ssh-copy-id-stapp01.png

## Step 5: Copy SSH Key to App Server 2
ssh-copy-id steve@172.16.238.11
📸 screenshots/ssh-copy-id-stapp02.png

## Step 6: Copy SSH Key to App Server 3
ssh-copy-id banner@172.16.238.12
📸 screenshots/ssh-copy-id-stapp03.png

## Step 7: Verify Password-less SSH Access
- App Server 1
  -  ssh tony@172.16.238.10 "hostname"

- App Server 2
  -  ssh steve@172.16.238.11 "hostname"

- App Server 3
  -  ssh banner@172.16.238.12 "hostname"

📸 screenshot:
<img width="1025" height="872" alt="image" src="https://github.com/user-attachments/assets/3f756692-115f-468e-a2fb-0d9ed14706c8" />

## ✅ Final Outcome

- SSH key-based authentication successfully configured

- Jump host accesses all app servers without passwords

- Hostname verification confirms correct server access

- Infrastructure ready for scheduled automation scripts

🏷️ Tags
`linux` `ssh` `authentication` `jump-host` `automation` `devops` `security`


<img width="1030" height="607" alt="image" src="https://github.com/user-attachments/assets/1c75275f-e725-4306-8a5a-fb573260cf23" />
<img width="919" height="662" alt="image" src="https://github.com/user-attachments/assets/18f2abe1-aaef-49e0-b596-6f8d2003ea1a" />
<img width="1030" height="860" alt="image" src="https://github.com/user-attachments/assets/486fe75b-b3a5-4b77-bd54-6b559757a4ec" />
<img width="1030" height="855" alt="image" src="https://github.com/user-attachments/assets/c2a3ce55-ddbd-44bd-b86e-abec1599d643" />
<img width="1029" height="865" alt="image" src="https://github.com/user-attachments/assets/abaed342-6e98-4d5d-b897-8c4cb1dbd8d9" />



