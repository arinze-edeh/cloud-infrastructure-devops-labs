# Azure Blob Container Access Hardening (Public → Private)

## Project Overview

This project documents a **security hardening operation** performed on an Azure Blob Storage container.
A publicly accessible blob container was converted to **private access** to restrict data exposure and
align the storage account with internal-only access requirements.

The operation was executed using the **Azure CLI**, leveraging the **storage account key fallback**
authentication mechanism when explicit credentials were not provided.

---

## Architecture Context

**Azure Subscription** 
- `Azure Free Labs`


**Storage Layer**

- Storage Account: `nautilusst17006`
- Region: `eastus`


**Blob Containers**

- `nautilus-container-11818` (PUBLIC → PRIVATE)

- `nautilus-priv-17871` (PRIVATE, unchanged)


---

## Security Objective

- Remove public access from a blob container
- Prevent anonymous read access
- Preserve existing private container configuration
- Ensure no data loss or container recreation

---

## Tooling & Environment

- Azure CLI
- Azure Cloud Environment
- Service Principal Authentication
- Storage Account Key (auto-queried by CLI)

---

## Step 1: Validate Active Azure Account

```
az account show
```

#### Expected Outcome

- Subscription is Enabled

- Correct tenant and subscription selected

- Service principal authenticated

📸 Screenshot:
<img width="1030" height="580" alt="image" src="https://github.com/user-attachments/assets/867e0c69-c3cf-4264-8787-5e1ef4f96237" />

## Step 2: Define Environment Variables
```
export STORAGE_ACCOUNT="nautilusst17006"
export TARGET_CONTAINER="nautilus-container-11818"
export REGION="eastus"
```

#### Purpose

- Avoid hard-coding values

- Improve command readability

- Enable reusability

📸 Screenshot:
<img width="1031" height="751" alt="image" src="https://github.com/user-attachments/assets/1122a816-f0af-4940-a243-b86c5fbacdff" />

## Step 3: Check Current Container Access Level
```
az storage container show \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --query "properties.publicAccess" \
  --output tsv
```

#### Observed Behavior

- CLI prompts for credentials

- Azure automatically queries storage account key

- Output returns container, confirming public access

📸 Screenshot:
<img width="1034" height="850" alt="image" src="https://github.com/user-attachments/assets/ca111580-4f2d-48db-abf4-b3081a07056c" />

## Step 4: Attempt RBAC Authentication (Observed Limitation)
```
az storage container set-permission \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --public-access off \
  --auth-mode login
```

#### Result

- Command fails

- `--auth-mode` login not supported in this context

- Allowed value defaults to key-based authentication

📸 Screenshot:
<img width="1032" height="672" alt="image" src="https://github.com/user-attachments/assets/e18a314d-1593-4b43-beb6-f45da6924d16" />

## Step 5: Convert Public Container to Private (Successful)
```
az storage container set-permission \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --public-access off
```

#### Outcome

- Azure CLI automatically retrieves storage account key

- Public access successfully disabled

- No impact on container contents

📸 Screenshot:
<img width="1032" height="510" alt="image" src="https://github.com/user-attachments/assets/1e648b06-5cb4-4352-a41a-f821935864ee" />

## Step 6: Verify Access Level After Change
```
az storage container show \
  --name $TARGET_CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --query "properties.publicAccess" \
  --output tsv
```
#### Expected Output

`null`

This confirms:

- No anonymous access

- Container is fully private

📸 Screenshot:
<img width="1033" height="539" alt="image" src="https://github.com/user-attachments/assets/0874306a-f075-47e8-89a7-4f54a2d5567a" />

## Step 7: List All Containers for Final Validation
```
az storage container list \
  --account-name $STORAGE_ACCOUNT \
  --query "[].{Name:name, PublicAccess:properties.publicAccess}" \
  --output table
```

#### Validation Results

- `nautilus-container-11818 → Private`

- `nautilus-priv-17871 → Private (unchanged)`

📸 Screenshot:
<img width="1039" height="425" alt="image" src="https://github.com/user-attachments/assets/7a391df4-3019-467c-953e-e311257ff107" />

## Final Outcome

- Public access successfully removed

- No container recreation

- No data loss

- Existing private container preserved

- Security posture improved

## Key Learnings

- Azure CLI will fallback to account keys if credentials are not explicitly supplied

- Not all storage operations support `--auth-mode login`

- Public blob access should be disabled unless explicitly required

- Validation steps are critical in security-sensitive operations


## Why This Matters

- Misconfigured public storage is one of the top cloud security risks.

This project demonstrates:

- Practical cloud governance

- Defensive security thinking

- Real-world CLI troubleshooting

- Verification-driven execution
