# Secure EC2 Access – Key Pair Creation (AWS CLI)

## Overview
This lab demonstrates how to create and securely manage an Amazon EC2 key pair using the AWS CLI.  
Key pairs are required for SSH-based authentication into EC2 instances and are a foundational security component in AWS environments.

---

## Objectives
- Create an EC2 key pair using RSA encryption
- Store the private key securely
- Apply correct Linux file permissions
- Verify key pair creation via AWS CLI

---

## Tools & Services
- AWS EC2
- AWS CLI
- Linux Shell
- AWS IAM (preconfigured lab credentials)

---

## Architecture Context
- Region: `us-east-1`
- Authentication Method: SSH (Key Pair)
- Encryption Type: RSA

---

## Implementation Steps

### Step 1: Verify AWS Region
Confirmed the working region is set to `us-east-1` before creating resources.

```bash
aws configure get region

