# AWS IAM User Creation

---

## Project Overview
OBJECTIVE:

Create an IAM user using AWS CLI

Verify successful creation

Perform actions in `us-east-1` region


---

## AWS Environment Details
SERVICE:

AWS Identity and Access Management (IAM)

REGION:

`us-east-1`

ACCESS METHOD:

AWS CLI on aws-client host


---

## Step-by-Step Implementation (CLI Based)

### Step 1: Confirm AWS Region
ENSURE AWS CLI IS OPERATING IN us-east-1

screenshot: `aws-cli-region-confirmation`
<img width="1026" height="679" alt="image" src="https://github.com/user-attachments/assets/1ef77d6b-5f26-4b10-b533-52dcf39f4abb" />

---

## Step 2: Create IAM User
COMMAND:

  -  Create IAM user named `iamuser_jim`

`aws iam create-user --user-name iamuser_jim`

EXPECTED OUTPUT:

  -  UserName: `iamuser_jim`

  -  ARN returned

  -  CreationDate populated

screenshot:`iamuser_jim-create-user`
<img width="1026" height="679" alt="image" src="https://github.com/user-attachments/assets/1ef77d6b-5f26-4b10-b533-52dcf39f4abb" />

---

### Step 3: List IAM Users to Verify Creation
COMMAND:

List all IAM users

aws iam list-users

VALIDATION:

`iamuser_jim` must appear in the users list

screenshot:`iamuser_jim-listed`
<img width="1025" height="826" alt="image" src="https://github.com/user-attachments/assets/85bdec04-80c2-4671-b362-b688e83131e4" />


---

## Validation Checklist
✔ IAM user iamuser_jim created successfully
✔ User visible in iam list-users output
✔ No errors returned by AWS CLI
✔ Task completed within required region


---


## Tags
`aws`
`iam`
`identity-and-security`
`user-management`
`aws-cli`


---

## Notes
- IAM is a global service but task was executed using us-east-1

- No policies were attached to the user

- No access keys were created

- CLI-based implementation only






