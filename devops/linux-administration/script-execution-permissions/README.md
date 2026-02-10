# Script Execution Permissions – Linux Administration Lab

## Lab Overview
This lab demonstrates how to grant executable permissions to a bash script on a remote Linux server while ensuring **all users** can execute the script.  
The task was performed within the **Stratos Datacenter** environment using a **jump host** for secure access.

---

## Environment Details

| Component        | Value |
|------------------|------|
| Jump Host        | jumphost.stratos.xfusioncorp.com |
| Target Server    | App Server 1 (stapp01) |
| Script Name      | xfusioncorp.sh |
| Script Location  | /tmp/xfusioncorp.sh |
| Required Access  | Execute permission for all users |

---

## Objective

- Grant executable permissions to `/tmp/xfusioncorp.sh`
- Ensure **owner, group, and others** can execute the script
- Validate permissions successfully

---

## High-Level Logic

CONNECT to jump host
AUTHENTICATE successfully

SSH into App Server 1
VERIFY script exists at target path

CHECK current permissions
IF script is not executable:
    APPLY execute permissions for all users

RE-VERIFY permissions
CONFIRM success

## Step-by-Step Implementation

## Step 1: Connect to Jump Host
ssh thor@jump_host.stratos.xfusioncorp.com
📸 Screenshot Placeholder:
![Jump Host Login](screenshots/jump-host-login.png)

## Step 2: SSH into App Server 1
ssh tony@stapp01.stratos.xfusioncorp.com
📸 Screenshot Placeholder:
![App Server Login](screenshots/app-server-login.png)

## Step 3: Verify Script Existence
ls -l /tmp/xfusioncorp.sh
Expected:

Script exists

Missing execute (x) permissions

📸 Screenshot Placeholder:
![Script Without Execute Permission](screenshots/script-missing-permissions.png)

## Step 4: Grant Execute Permission to All Users
sudo chmod a+x /tmp/xfusioncorp.sh
a+x ensures user, group, and others can execute the script.

📸 Screenshot Placeholder:
![chmod Execution](screenshots/chmod-execution.png)

## Step 5: Verify Final Permissions
ls -l /tmp/xfusioncorp.sh
Expected Output:

-r-xr-xr-x 1 root root ...
📸 Screenshot Placeholder:
![Final Permission Verification](screenshots/final-verification.png)

## Result
✔ Script is executable
✔ All users have execution permission
✔ Task completed successfully

## Key Linux Concepts Demonstrated
Linux file permission model (rwx)

chmod symbolic mode (a+x)

Secure SSH access via jump host

Permission verification using ls -l

##  Tags
linux chmod permissions devops bash system-administration




<img width="1026" height="556" alt="image" src="https://github.com/user-attachments/assets/400833d8-d281-4262-9834-bfab5a99dafd" />
<img width="1040" height="531" alt="image" src="https://github.com/user-attachments/assets/a752e1df-151a-476c-adc9-e82b0b8d609a" />
<img width="716" height="513" alt="image" src="https://github.com/user-attachments/assets/06281ee9-e830-4eb5-8fae-832ac18e9f8d" />
<img width="1009" height="802" alt="image" src="https://github.com/user-attachments/assets/b531f87c-1e4b-43c9-bb7c-7f08407f8feb" />
<img width="958" height="816" alt="image" src="https://github.com/user-attachments/assets/4a8be63b-45eb-466d-a64f-10e1a18c19c4" />
<img width="976" height="874" alt="image" src="https://github.com/user-attachments/assets/954d1bec-78b7-49ff-81fd-2a55a34d2499" />
<img width="1043" height="872" alt="image" src="https://github.com/user-attachments/assets/1d8bfc47-a2ec-48bc-a517-d4e5ab26dc93" />
<img width="965" height="877" alt="image" src="https://github.com/user-attachments/assets/17ca42ed-05af-4a64-87d7-b86bdf560499" />




