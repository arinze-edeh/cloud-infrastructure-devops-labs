# Azure Blob Storage File Upload – Nautilus DevOps Lab

## Project Overview

- This project documents the process of uploading a local file to an Azure Blob Storage container using the Azure CLI.

- The file `/tmp/xfusion.txt` was uploaded to the Blob container `xfusion-blob-29364` in the `westus region` under the storage account `xfusionst15842`.

- Objective: Validate end-to-end file upload and verification in Azure Blob Storage.

## Architecture Summary
          ┌─────────────────────────┐
          │  Azure Storage Account  │
          │      xfusionst15842     │
          └───────────┬────────────┘
                      │
          ┌───────────▼────────────┐
          │  Blob Container        │
          │  xfusion-blob-29364    │
          └───────────┬────────────┘
                      │
            ┌─────────▼─────────┐
            │  Block Blob File  │
            │  xfusion.txt      │
            └───────────────────┘

## 🔧 Technologies Used

- Azure CLI

- Azure Blob Storage

- Linux (Ubuntu/CentOS)

- Azure Active Directory (AD) Authentication

- Public Blob Containers

## 🚀 Implementation Steps

### Step 1️: Login to Azure

- `az login`

Check your account details:

- `az account show`

Expected Output:
````
{
  "environmentName": "AzureCloud",
  "homeTenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512",
  "isDefault": true,
  "name": "Azure Free Labs",
  "state": "Enabled",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "user": {
    "name": "f902723d-a08c-4008-9c35-56cc27e9a966",
    "type": "servicePrincipal"
  }
}
````

📸 Screenshots:
<img width="1031" height="569" alt="image" src="https://github.com/user-attachments/assets/488fecf2-3b9c-4ba1-aa0e-1f067bc76b65" />

### 2️⃣ Verify Local File
- `ls -l /tmp/xfusion.txt`

Expected Output:

- `-rw-r--r-- 1 root root 33 Feb 25 02:21 /tmp/xfusion.txt`

📸 Screenshots:
<img width="1029" height="587" alt="image" src="https://github.com/user-attachments/assets/018e6fe3-79c1-47eb-af59-a720025af1eb" />

### 3️⃣ List Storage Accounts
- `az storage account list --query "[].name" -o table`

Expected Output:
````
Result
--------------
xfusionst15842
````

📸 Screenshots:
<img width="1027" height="640" alt="image" src="https://github.com/user-attachments/assets/a583967d-4e89-47bb-9280-6f026aaf07a5" />

### 4️⃣ Upload File to Blob Container
````
az storage blob upload \
  --account-name xfusionst15842 \
  --container-name xfusion-blob-29364 \
  --name xfusion.txt \
  --file /tmp/xfusion.txt \
  --auth-mode login
````

Expected Output:
````
{
  "client_request_id": "5234cbe2-11f3-11f1-935a-da0a61c5debf",
  "content_md5": "Lu7zilatbGguzSz2Ecn5IQ==",
  "date": "2026-02-25T02:40:19+00:00",
  "etag": "\"0x8DE7417378C459D\"",
  "lastModified": "2026-02-25T02:40:19+00:00",
  "request_server_encrypted": true,
  "version": "2022-11-02"
}
````
📸 Screenshots:
<img width="1032" height="862" alt="image" src="https://github.com/user-attachments/assets/85fe9add-ba1a-4682-802c-0481995bca75" />

### 5️⃣ Verify Upload
````
az storage blob list \
  --account-name xfusionst15842 \
  --container-name xfusion-blob-29364 \
  --output table
````
Expected Output:

| Property       | Value                         |
|----------------|-------------------------------|
| Name           | xfusion.txt                   |
| Blob Type      | BlockBlob                     |
| Blob Tier      | Hot                           |
| Length         | 33                            |
| Content Type   | text/plain                    |
| Last Modified  | 2026-02-25T02:40:19+00:00    |

📸 Screenshots:
<img width="1036" height="855" alt="image" src="https://github.com/user-attachments/assets/a3af339f-51ec-420e-ae01-dc6776d8441c" />

### Validation Checklist

| Check                      | Status |
| -------------------------- | ------ |
| Azure login successful     | ✅      |
| Local file exists          | ✅      |
| Storage account exists     | ✅      |
| Blob uploaded successfully | ✅      |
| Blob visible in container  | ✅      |


### Final Outcome

- Successfully uploaded /tmp/xfusion.txt to Azure Blob Storage.

- Verified end-to-end file upload using Azure CLI.

- Data is stored in a public Blob container for further access.
