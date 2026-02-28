# Git Repository Deployment on Nautilus Storage Server

## Project Overview
This project documents the deployment of the **Apps Git repository** (`/opt/apps.git`) to the Nautilus Storage Server in the Stratos Datacenter.  
The repository, initially unused, was cloned to provide the development team with a ready-to-use workspace for managing the Nautilus application source code.

---

## Objectives
* Clone `/opt/apps.git` to `/usr/src/kodekloudrepos`  
* Use the `natasha` user account  
* Ensure no modifications are made to the repository or existing directories  
* Preserve the integrity of the repository and its structure  

---

## Pre-Requisites
* Access to Storage Server: `ststor01.stratos.xfusioncorp.com`  
* User: `natasha`  
* Password: `Bl@kW`  
* Git installed on the server  
* Target directory: `/usr/src/kodekloudrepos`  

---

## Steps to Clone Repository

### Step 1: SSH into Storage Server
```
ssh natasha@ststor01.stratos.xfusioncorp.com
# Password: ****
```

Screenshot:
<img width="1037" height="597" alt="image" src="https://github.com/user-attachments/assets/ef3ca39b-89aa-4681-8e42-a1e6300a8b32" />

## SStep 2: Verify Source Repository
```
ls -ld /opt/apps.git
ls /opt/apps.git
```
Screenshot: `Confirms that /opt/apps.git exists and contains Git objects (HEAD, branches, config, etc.)`
<img width="1036" height="532" alt="image" src="https://github.com/user-attachments/assets/107bd6af-454a-4155-b318-cdbf03e1d97e" />



## SStep 3: Verify / Create Target Directory
```
ls -ld /usr/src/kodekloudrepos
mkdir -p /usr/src/kodekloudrepos
```
Screenshot:

## SStep 4: Navigate to Target Directory
```
cd /usr/src/kodekloudrepos
pwd
```
Screenshot:

## SStep 5: Clone the Repository
```
git clone /opt/apps.git
```
Screenshot:

Note: The clone created a subfolder named apps. The repository is currently empty.

## SStep 6: Navigate into Cloned Repository and Verify
```
cd apps
git status
git remote -v
```
Screenshot:

- git status confirms On branch master and No commits yet

- git remote -v confirms the origin is set to /opt/apps.git

## Validation

- The repository now exists in `/usr/src/kodekloudrepos/apps`

- Remote origin is correctly set to `/opt/apps.git`

- No files were modified or deleted outside the repository

- Ready for the development team to add files and push commits




<img width="1033" height="592" alt="image" src="https://github.com/user-attachments/assets/61b719a7-5d66-4892-b663-e3039c5958bc" />

<img width="1036" height="560" alt="image" src="https://github.com/user-attachments/assets/9a6f97cd-b7cd-4507-a1d4-ecf9059f5fe0" />
<img width="1032" height="605" alt="image" src="https://github.com/user-attachments/assets/c55cf2f6-69d9-44f9-8064-46921d0abe7e" />
<img width="1036" height="718" alt="image" src="https://github.com/user-attachments/assets/916c568a-c7e2-4a96-9c9f-8b3165d33278" />
<img width="1039" height="653" alt="image" src="https://github.com/user-attachments/assets/1d7ab386-bb1f-445b-9fc2-ba4d606d66db" />
<img width="1020" height="715" alt="image" src="https://github.com/user-attachments/assets/0c39a5e7-daef-44d4-b4f0-abfe10307be8" />
<img width="1036" height="779" alt="image" src="https://github.com/user-attachments/assets/edbb2f62-7fe6-4bad-9de3-87e42a6bc402" />
<img width="1032" height="799" alt="image" src="https://github.com/user-attachments/assets/af4037e0-c55d-4be2-aed8-0b94196bb661" />
<img width="1036" height="838" alt="image" src="https://github.com/user-attachments/assets/e68d956a-7867-47fe-9f7e-c9fbb2aefd1c" />


