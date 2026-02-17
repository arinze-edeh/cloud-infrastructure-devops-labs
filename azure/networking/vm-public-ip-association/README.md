#Azure VM Public IP Attachment

## Project Overview

### OBJECTIVE:

- Attach an existing Public IP to a VM NIC.

- Ensure VM is accessible via Public IP.

### Azure Environment Details
CLOUD PROVIDER:

`Microsoft Azure.`

RESOURCES:

Resource Group: `kml_rg_main-bf56eb2fed794029.`

Virtual Machine: `datacenter-vm-pip (Location: eastus).`

Public IP: `datacenter-pip (Location: westus).`

Network Interface Card (NIC): `datacenter-vm-pipVMNic.`

ACCESS METHOD:

`Azure CLI.`

## Implementation Steps

## Step 1: Authenticate
ACTION:

Login to Azure using provided credentials.

## Step 2: Validate Resources & Discovery
CONFIRM:

- List resource groups and locations: `az group list --query "[].{Name:name, Location:location}" --output table.`

- Identify Target NIC: `az vm show --name datacenter-vm-pip --resource-group kml_rg_main-bf56eb2fed794029 --query "networkProfile.networkInterfaces[0].id".`

- Verify Public IP details: `az network public-ip list --resource-group kml_rg_main-bf56eb2fed794029 --output table.`

screenshots: `resource-validation`
<img width="1030" height="696" alt="image" src="https://github.com/user-attachments/assets/d8acd204-f1a9-43bd-b85c-6df6fc14ff53" />
<img width="1035" height="716" alt="image" src="https://github.com/user-attachments/assets/6cbf15a4-f744-433f-aee9-145e9aa1d1f8" />
<img width="1034" height="685" alt="image" src="https://github.com/user-attachments/assets/148eb405-1bf8-4fd6-99bf-778d8c06b1e2" />
<img width="1033" height="558" alt="image" src="https://github.com/user-attachments/assets/f5bdaee4-17d4-44aa-96c4-7929c8e5a2df" />

## Step 3: Troubleshoot & Deallocate
COMMAND:

`Initial attempt to update using generic name ipconfig1 failed with ResourceNotFoundError.`

Discovery: Found actual IP configuration name: `az network nic show --name datacenter-vm-pipVMNic --resource-group kml_rg_main-bf56eb2fed794029 --query "ipConfigurations[].name" (Result: ipconfigdatacenter-vm-pip).`

Optional: `az vm deallocate --resource-group kml_rg_main-bf56eb2fed794029 --name datacenter-vm-pip.`

screenshot: `vm-deallocated`

## Step 4: Attach Public IP to NIC
COMMAND:

Capture Public IP Resource ID: `PIP_ID=$(az network public-ip show --name datacenter-pip --resource-group kml_rg_main-bf56eb2fed794029 --query id -o tsv).`

Assign datacenter-pip using the correct configuration name:

- `az network nic ip-config update \`
  -  `--name "ipconfigdatacenter-vm-pip" \`
  -  `--nic-name datacenter-vm-pipVMNic \`
  -  `--resource-group kml_rg_main-bf56eb2fed794029 \`
  -  `--public-ip-address $PIP_ID`

screenshot: `nic-public-ip-attachment'`
<img width="1034" height="795" alt="image" src="https://github.com/user-attachments/assets/1b2fe9c1-d95f-4bc6-8400-aedc332749f7" />

## Step 5: Start VM
COMMAND:

Restart VM: `az vm start --resource-group kml_rg_main-bf56eb2fed794029 --name datacenter-vm-pip.`

screenshot: `vm-started`

## Step 6: Verification
CHECK:

Verify JSON response for "provisioningState": "Succeeded".

Confirm Public IP address is assigned: az vm list-ip-addresses --name datacenter-vm-pip --resource-group kml_rg_main-bf56eb2fed794029 --output table.

screenshot: `public-ip-confirmed`

<img width="1040" height="834" alt="image" src="https://github.com/user-attachments/assets/fcc044cd-7b7e-4fa6-adb9-29a8fe7047f9" />

## Validation Checklist

- Public IP ID captured successfully
- NIC IP configuration name correctly identified
- NIC updated to "Succeeded" status
- Public IP 23.100.42.173 linked to VM

## Notes
- Cross-Region Association: The Public IP was located in `westus` while the VM was in `eastus`; using the resource ID `($PIP_ID)` ensured proper mapping.

- Resource Names: IP configuration names may not follow standard naming conventions; always verify with az network nic show.

## Lessons Learned

- Avoid Default Assumptions
`During the initial implementation, the command failed because it assumed the IP configuration name was ipconfig1.`

`It is critical to use az network nic show to discover the actual resource names within the environment.`

- Handling Cross-Region Resources
`The Public IP (westus) and the Virtual Machine (eastus) were located in different Azure regions.`

`Using the Resource ID instead of just the resource name provides a more robust way to link assets across different locations and avoid naming conflicts.`

- Troubleshooting "Resource Not Found" Errors
`A ResourceNotFoundError in Azure CLI is often a sign of a naming mismatch rather than a missing resource.`

`Systematic "Discovery" commands (like list and show) should always precede "Update" commands to ensure the target environment state is accurately mapped.`

- Efficient Scripting with Variables
`Storing Resource IDs in variables (e.g., $PIP_ID) makes CLI commands cleaner and significantly easier to debug or automate in larger DevOps pipelines.`

## Tags
`azure`
`networking`
`vm`
`public-ip`
`nic`
`azure-cli`


<img width="1034" height="681" alt="image" src="https://github.com/user-attachments/assets/0d6a2286-b3ab-47a1-9adc-9e7b1048913b" />



