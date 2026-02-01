# Secure EC2 Access – Key Pair Creation (AWS CLI)

## Overview
This lab demonstrates how to create and securely manage an Amazon EC2 key pair using the AWS CLI.  
Key pairs are required for secure SSH access to EC2 instances and are a foundational security control in AWS environments.

---

## Objectives
- Create an EC2 key pair using RSA encryption
- Secure the private key with proper Linux permissions
- Verify key pair creation using AWS CLI
- Prepare credentials for secure EC2 access

---

## Tools & Services Used
- AWS EC2
- AWS CLI
- Linux Shell
- IAM (Preconfigured lab credentials)

---

## Environment Details
- AWS Region: `us-east-1`
- Authentication Method: SSH Key Pair
- Key Type: RSA

---

## Step 1: Confirm AWS Region

<img width="1041" height="880" alt="image" src="https://github.com/user-attachments/assets/f0f4b053-6032-4924-9152-d2b329c1f20d" />

## Step 2: Create EC2 Key Pair

- Created an RSA-based EC2 key pair named xfusion-kp using the AWS CLI

aws ec2 create-key-pair \
  --key-name xfusion-kp \
  --key-type rsa \
  --query 'KeyMaterial' \
  --output text > xfusion-kp.pem
<img width="1109" height="948" alt="image" src="https://github.com/user-attachments/assets/650da4fc-6985-47d3-aa1f-5526b7674019" />


## Step 3: Secure the Private Key

- Applied strict file permissions to protect the private key as required by SSH.

chmod 400 xfusion-kp.pem
<img width="1041" height="880" alt="image" src="https://github.com/user-attachments/assets/f0f4b053-6032-4924-9152-d2b329c1f20d" />

- Verified permissions

ls -l xfusion-kp.pem

<img width="1099" height="902" alt="image" src="https://github.com/user-attachments/assets/4dd0aabd-ebcf-45c8-a0c5-440fe9b40481" />

## Step 4: Verify Key Pair Creation in AWS

- Confirmed the key pair exists in AWS.

aws ec2 describe-key-pairs --key-names xfusion-kp
<img width="1068" height="911" alt="image" src="https://github.com/user-attachments/assets/edd63c96-09dd-4acb-b4fd-d3be90c7ec9b" />


## Result

- EC2 key pair successfully created

- Private key securely stored

- Key pair verified and ready for EC2 instance launches

## Security Best Practices

- Never commit private keys to GitHub

- Restrict key permissions to the owner only

- Rotate keys periodically in production environments

## Real-World Relevance

- This workflow reflects how Cloud and DevOps Engineers securely provision access to EC2 instances in real production environments using CLI-based automation.

## Skills Demonstrated

- AWS EC2 access management

- AWS CLI usage

- Linux file permission management

- Cloud security fundamentals

<img width="1035" height="861" alt="image" src="https://github.com/user-attachments/assets/68cb9cd9-87cd-40cd-9900-a5645d558212" />
<img width="1035" height="865" alt="image" src="https://github.com/user-attachments/assets/b0e78cff-d0a8-4919-b356-7463cdeb7f4d" />
<img width="1036" height="772" alt="image" src="https://github.com/user-attachments/assets/046f5460-03f7-404e-bd87-8adb6de5a423" />



