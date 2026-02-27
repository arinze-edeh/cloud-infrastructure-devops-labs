# Centralized Git Server Provisioning on Storage Node

## Project Overview
This project documents the provisioning of a centralized Git server on the Nautilus Storage Server within the Stratos Datacenter. The objective is to install Git using the yum package manager and initialize a bare Git repository to support collaborative source control workflows across distributed development teams.

---

## Environment Context
- Datacenter: Stratos DC
- Access Pattern: Jump Host → Storage Server
- Target Node: ststor01
- OS Family: CentOS Stream 9
- Repository Type: Bare Git Repository
- Repository Path: /opt/demo.git

---

## Prerequisites
- SSH access to Jump Host
- SSH access to Storage Server
- sudo privileges on Storage Server
- Network connectivity between Jump Host and Storage Server

---

## Step 1: Access Jump Host
```
ssh thor@jumphost
```
Screenshot: Successful login to Jump Host
<img width="1030" height="550" alt="image" src="https://github.com/user-attachments/assets/10e208a6-fdb4-4157-9132-4799bbda5c47" />

## Step 2: Connect to Storage Server
```
ssh natasha@172.16.238.15
```
Screenshot: SSH authenticity prompt and connection acceptance

<img width="1030" height="550" alt="image" src="https://github.com/user-attachments/assets/10e208a6-fdb4-4157-9132-4799bbda5c47" />

## Step 3: Install Git Using yum
```
sudo yum install -y git
```
Screenshot: yum resolving repositories and dependencies

Screenshot: Git installation completed successfully

## Step 4: Validate Git Installation
```
git --version
```
Screenshot: Git version output confirming installation
*

## Step 5: Initialize Bare Git Repository
```
sudo git init --bare /opt/demo.git
```
Screenshot: Bare repository initialization message

Screenshot: Repository initialized at /opt/demo.git

## Step 6: Verify Repository Structure
```
ls -l /opt/demo.git
```
Screenshot: Bare Git repository directory structure
<img width="1029" height="523" alt="image" src="https://github.com/user-attachments/assets/1c8b0329-7172-4862-a405-df1d08569f60" />

## Repository Validation Checklist

- Git installed successfully via yum

- Bare repository created at correct path

- Repository contains standard bare Git directories:

  -  HEAD

  -  config

  -  hooks

  -  info

  -  objects

  -  refs

## Outcome

- Centralized Git backend successfully provisioned

- Repository ready for remote push and pull operations

- Infrastructure aligned with enterprise source control standards

## Operational Notes

- Bare repositories do not contain working trees

- Intended strictly for shared access and collaboration

- Default branch initialized as master (can be changed globally if required)



<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/1f55ebae-8ec2-44ca-9876-e9b5285985d6" />
<img width="1035" height="861" alt="image" src="https://github.com/user-attachments/assets/ca30e46d-72f4-42a6-81e5-66b041ba54a2" />
<img width="1032" height="348" alt="image" src="https://github.com/user-attachments/assets/a2b2bed4-13dc-4da4-ba3c-e8959aa4bce8" />



