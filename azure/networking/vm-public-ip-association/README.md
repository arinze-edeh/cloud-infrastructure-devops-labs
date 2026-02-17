# Azure VM Public IP Attachment
---

## Project Overview
OBJECTIVE:

- Attach an existing Public IP to a VM NIC

- Ensure VM is accessible via Public IP


---

## Azure Environment Details
CLOUD PROVIDER:

`Microsoft Azure`

RESOURCES:

Virtual Machine: `datacenter-vm-pip`

Public IP: `datacenter-pip`

`Network Interface Card (NIC)`

ACCESS METHOD:

`Azure CLI`

---

## Implementation Steps

## Step 1: Authenticate
ACTION:

Login to Azure using provided credentials

screenshot:`azure-login`


---

## Step 2: Validate Resources
CONFIRM:

VM exists

Public IP exists

NIC exists

screenshot:`resource-validation`


---

## Step 3: Deallocate VM
COMMAND:

Stop VM before NIC modification

`az vm deallocate --resource-group <rg> --name datacenter-vm-pip`

screenshot:`vm-deallocated`


---

## Step 4: Attach Public IP to NIC
COMMAND:

Assign datacenter-pip to NIC

az network nic ip-config update
--resource-group <rg>
--nic-name <nic-name>
--name <ip-config-name>
--public-ip-address datacenter-pip

screenshot:`nic-public-ip-attachment`


---

## Step 5: Start VM
COMMAND:

Restart VM

`az vm start --resource-group <rg> --name datacenter-vm-pip`

screenshot:`vm-started`
<img width="1034" height="795" alt="image" src="https://github.com/user-attachments/assets/1b2fe9c1-d95f-4bc6-8400-aedc332749f7" />

---

## Step 6: Verification
CHECK:

VM has Public IP assigned

`az vm show -d --resource-group <rg> --name datacenter-vm-pip --query publicIps`

screenshot:`public-ip-confirmed`
<img width="1040" height="834" alt="image" src="https://github.com/user-attachments/assets/fcc044cd-7b7e-4fa6-adb9-29a8fe7047f9" />

---

## Validation Checklist
- Public IP attached successfully
- VM restarted successfully
- Public IP visible on VM
- Task completed without errors


---

## Notes
- VM must be deallocated before NIC changes

- Public IP must exist before attachment

- No resource recreation required

## Tags

`azure`
`networking`
`vm`
`public-ip`
`nic`
`azure-cli`

<img width="1030" height="696" alt="image" src="https://github.com/user-attachments/assets/d8acd204-f1a9-43bd-b85c-6df6fc14ff53" />
<img width="1035" height="716" alt="image" src="https://github.com/user-attachments/assets/6cbf15a4-f744-433f-aee9-145e9aa1d1f8" />
<img width="1034" height="685" alt="image" src="https://github.com/user-attachments/assets/148eb405-1bf8-4fd6-99bf-778d8c06b1e2" />
<img width="1033" height="558" alt="image" src="https://github.com/user-attachments/assets/f5bdaee4-17d4-44aa-96c4-7929c8e5a2df" />
<img width="1034" height="681" alt="image" src="https://github.com/user-attachments/assets/0d6a2286-b3ab-47a1-9adc-9e7b1048913b" />


