# PostgreSQL User and Database Provisioning (Nautilus Infra)

## Overview
- This project documents the successful provisioning of a PostgreSQL database user and database on the Nautilus infrastructure database server.  
- The setup was performed to support a newly developed application requiring PostgreSQL access.

- All actions were executed **without restarting the PostgreSQL service**, in compliance with operational constraints.

---

## Infrastructure Context

| Component | Details |
|---------|--------|
| Environment | Nautilus Infrastructure |
| Database Server | `stdb01.stratos.xfusioncorp.com` |
| Access Method | Jump Host → DB Server |
| Database Engine | PostgreSQL |
| Privilege Scope | Database-level (full access) |

---

## Task Requirements

- Create a PostgreSQL user:
  - **Username:** `kodekloud_joy`
  - **Password:** `TmPcZjtRQx`
- Create a PostgreSQL database:
  - **Database Name:** `kodekloud_db4`
- Grant **full privileges** on the database to the user
- Do **NOT** restart the PostgreSQL service

---

## Execution Steps

## Step 1️: Connect to Database Server

- `ssh peter@stdb01.stratos.xfusioncorp.com`

📸 Screenshot:
<img width="1031" height="558" alt="image" src="https://github.com/user-attachments/assets/4b22da20-6c2c-45c2-bb61-60c417359241" />

## Step 2: Switch to PostgreSQL System User
- `sudo -i -u postgres`

📸 Screenshot:
<img width="1028" height="592" alt="image" src="https://github.com/user-attachments/assets/02b9e10f-ce9a-49d8-a791-3519db91f1a6" />

## Step 3: Create PostgreSQL User
- `psql -c "CREATE USER kodekloud_joy WITH PASSWORD 'TmPcZjtrQx';"`

📸 Screenshot:
<img width="1035" height="675" alt="image" src="https://github.com/user-attachments/assets/46c1ef45-0507-43e9-a54d-4fbda452526a" />

## Step 4: Create Database
- `psql -c "CREATE DATABASE kodekloud_db4;"`

📸 Screenshot:
<img width="1029" height="620" alt="image" src="https://github.com/user-attachments/assets/7f13cb10-fcd4-446f-a613-2553591ca77d" />

## Step 5: Grant Database Privileges
- `psql -c "GRANT ALL PRIVILEGES ON DATABASE kodekloud_db4 TO kodekloud_joy;"`

📸 Screenshot:
<img width="1036" height="729" alt="image" src="https://github.com/user-attachments/assets/81ed52da-892e-4d2e-9eaa-8940e8fd4f29" />

## Step 6: Verify Database Creation
- `psql -c "\l"`

📸 Screenshot:
<img width="1034" height="780" alt="image" src="https://github.com/user-attachments/assets/2729a174-9a48-4431-ae36-cdbe1bcf8ab5" />

## Step 7: Verify User Creation
- `psql -c "\du"`

📸 Screenshot:
<img width="1033" height="862" alt="image" src="https://github.com/user-attachments/assets/3a2d2f30-dcfe-4ba0-a1e4-d556c1b88cf3" />

## Step 8: Correct Password Case (Final Compliance Step)

- The required password contained an uppercase `R`, which was corrected using the command below.

- `psql -c "ALTER USER kodekloud_joy WITH PASSWORD 'TmPcZjtRQx';"`

📸 Screenshot:
<img width="1029" height="860" alt="image" src="https://github.com/user-attachments/assets/6713a9c5-662b-4bb2-821a-9f0189ecc1ef" />

## Validation Summary

| Check                        | Status            |
| ---------------------------- | ----------------- |
| User created                 | ✅                 |
| Database created             | ✅                 |
| Privileges granted           | ✅                 |
| Password matches requirement | ✅                 |
| PostgreSQL service restarted | ❌ (Not restarted) |

## Security & Best Practices

- Passwords are case-sensitive and securely stored as hashes by PostgreSQL

- Changes were applied live without impacting service availability

- Principle of least privilege respected (database-scoped access)

## Outcome

- The PostgreSQL user and database were provisioned successfully and validated against all requirements.
- The environment is now ready for application-level database integration.
