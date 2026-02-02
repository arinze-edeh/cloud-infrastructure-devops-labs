# Linux User Management – Temporary User with Expiry Date

## Overview
This lab demonstrates how to create a temporary Linux user account with
a predefined expiry date. Expiry-based users are commonly used for
contractors, temporary developers, and short-term project assignments
to enforce automatic access revocation.

---

## Lab Context
Project Nautilus – xFusionCorp Industries  
Requirement: Grant temporary access to a developer assigned to the Nautilus project.

Target Server:
- App Server 3 (stapp03)

---

## Objective
- Create a user named `mariyam` (lowercase)
- Set an account expiry date of `2026-12-07`
- Ensure the account automatically disables after expiry
- Verify correct configuration

---

## Environment
- OS: Linux
- Server Role: Application Server
- Privileges: sudo access

---

## Step 1: Connect to Target Server

- CONNECT to jump host
- SSH into App Server 3 using provided credentials

📸 Screenshot:

<img width="1035" height="743" alt="image" src="https://github.com/user-attachments/assets/4fdaa99f-8114-4747-a020-95d183d34738" />
SSH session connected to stapp03

## Step 2: Create Temporary User with Expiry Date
- EXECUTE useradd command
- SET account expiry date
- USERNAME = mariyam
- EXPIRY_DATE = 2026-12-07
- sudo useradd -e 2026-12-07 mariyam

📸 Screenshot: 
<img width="1034" height="832" alt="image" src="https://github.com/user-attachments/assets/66e1854a-ccb3-44ea-9604-4421c8d85d89" />

useradd command execution

## Step 3: Verify Account Expiry Configuration
- QUERY account aging information
- CONFIRM expiry date is correctly applied
- sudo chage -l mariyam

Expected Output:

Account expires : Dec 07, 2026

📸 Screenshot:
<img width="1044" height="750" alt="image" src="https://github.com/user-attachments/assets/e42b3d22-a924-41d9-b5ab-4c9971dd5017" />
chage output showing expiry date

## Step 4: Validate User Creation
- CONFIRM user exists in system
- VERIFY username is lowercase
- getent passwd mariyam

📸 Screenshot:

<img width="1032" height="765" alt="image" src="https://github.com/user-attachments/assets/a5a58504-a67e-4fc4-ac76-fe00c147bb05" />

passwd entry for mariyam

## Result
- Temporary user created successfully

- Account expiry date enforced

- Automatic access revocation configured

## Security Best Practices
- Always set expiry dates for temporary users

- Review and audit expiring accounts regularly

- Combine expiry with least-privilege access

- Remove unused accounts promptly

## Real-World Relevance
This mirrors enterprise practices where:

- Contractors receive time-bound access

- Compliance requires automatic deprovisioning

- Security teams reduce orphaned accounts

## Skills Demonstrated
- Linux user lifecycle management

- Account expiration policies

- Secure access control

- Enterprise system administration







<img width="1040" height="784" alt="image" src="https://github.com/user-attachments/assets/5ab00ae9-4f77-4560-ae7b-969ca4f247a9" />
<img width="1028" height="840" alt="image" src="https://github.com/user-attachments/assets/590e2931-6957-4019-a427-007da6eccfea" />
<img width="1023" height="793" alt="image" src="https://github.com/user-attachments/assets/b7eaec4f-1a4a-4dcd-8cb5-e78e2f99de46" />

