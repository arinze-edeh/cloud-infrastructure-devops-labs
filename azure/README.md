# Azure Cloud Labs

~ ➜  showcreds
╒═════════════════════════════╤════════════════════════════════════════════════════════════════════╕
│ Name                        │ Value                                                              │
╞═════════════════════════════╪════════════════════════════════════════════════════════════════════╡
│ Azure Console URL           │ https://portal.azure.com/azurefreekmlprod.onmicrosoft.com          │
├─────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Azure User Name             │ kk_lab_user_main-9642fe6e452040f1@azurefreekmlprod.onmicrosoft.com │
├─────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Azure Password              │ b7k3ZPVx                                                           │
├─────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Azure Application Client ID │ 4cfa7b3d-fe05-4f32-b714-30549b389a2c                               │
├─────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Azure Session End Time      │ Wed Feb 18 02:29:41 UTC 2026                                       │
╘═════════════════════════════╧════════════════════════════════════════════════════════════════════╛

~ ➜  az vm list --query "[].{VM_Name:name, ResourceGroup:resourceGroup, Location:location}" -o table
VM_Name     ResourceGroup                 Location
----------  ----------------------------  ----------
xfusion-vm  KML_RG_MAIN-9642FE6E452040F1  centralus

~ ➜  az vm show \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm \
  --query hardwareProfile.vmSize
"Standard_B1s"

~ ➜  az vm update \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm \
  --size Standard_B2s
Argument '--size' is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
{
  "additionalCapabilities": null,
  "applicationProfile": null,
  "availabilitySet": null,
  "billingProfile": null,
  "capacityReservation": null,
  "diagnosticsProfile": null,
  "etag": "\"2\"",
  "evictionPolicy": null,
  "extendedLocation": null,
  "extensionsTimeBudget": null,
  "hardwareProfile": {
    "vmSize": "Standard_B2s",
    "vmSizeProperties": null
  },
  "host": null,
  "hostGroup": null,
  "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/KML_RG_MAIN-9642FE6E452040F1/providers/Microsoft.Compute/virtualMachines/xfusion-vm",
  "identity": null,
  "instanceView": null,
  "licenseType": null,
  "location": "centralus",
  "managedBy": null,
  "name": "xfusion-vm",
  "networkProfile": {
    "networkApiVersion": null,
    "networkInterfaceConfigurations": null,
    "networkInterfaces": [
      {
        "deleteOption": null,
        "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-9642fe6e452040f1/providers/Microsoft.Network/networkInterfaces/xfusion-vmVMNic",
        "primary": null,
        "resourceGroup": "kml_rg_main-9642fe6e452040f1"
      }
    ]
  },
  "osProfile": {
    "adminPassword": null,
    "adminUsername": "azureuser",
    "allowExtensionOperations": true,
    "computerName": "xfusion-vm",
    "customData": null,
    "linuxConfiguration": {
      "disablePasswordAuthentication": true,
      "enableVmAgentPlatformUpdates": null,
      "patchSettings": {
        "assessmentMode": "ImageDefault",
        "automaticByPlatformSettings": null,
        "patchMode": "ImageDefault"
      },
      "provisionVmAgent": true,
      "ssh": {
        "publicKeys": [
          {
            "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDbva8jdWi3YgLrkR0bZgbxlOVleFREX7d3ThaDaf0uEojSdpFmE2/VwQyBIS5OKxS0Exaw0/+y2Em7D/z1q4+cYk2njQUNNtyneNtTXF3RVl+IgB5u0dti6HDRsolGatu24O+bokAkLJ3sKODd8hWelWl1W85xznoPSyJmVzUn0OwdVgPD8/OaLlt1npspH7SgtsujEbRyvBo0v/0+PEMKtrxqdT+hLCk6Wmn6Q2JWVrWon+DhAuBXzmnCsLMNOCOlQ+1jjYw51Cc1bVDtACz6ROYmyLCzhd/kRkHeZlzNfv7ZXQYlbVKaRFOX6b4m71HRu0fgiI9xHh/Eer/PCQpH root@azure-client\n",
            "path": "/home/azureuser/.ssh/authorized_keys"
          }
        ]
      }
    },
    "requireGuestProvisionSignal": true,
    "secrets": [],
    "windowsConfiguration": null
  },
  "plan": null,
  "platformFaultDomain": null,
  "priority": null,
  "provisioningState": "Succeeded",
  "proximityPlacementGroup": null,
  "resourceGroup": "KML_RG_MAIN-9642FE6E452040F1",
  "resources": null,
  "scheduledEventsPolicy": null,
  "scheduledEventsProfile": null,
  "securityProfile": {
    "encryptionAtHost": null,
    "encryptionIdentity": null,
    "proxyAgentSettings": null,
    "securityType": "TrustedLaunch",
    "uefiSettings": {
      "secureBootEnabled": true,
      "vTpmEnabled": true
    }
  },
  "storageProfile": {
    "dataDisks": [],
    "diskControllerType": "SCSI",
    "imageReference": {
      "communityGalleryImageId": null,
      "exactVersion": "22.04.202601310",
      "id": null,
      "offer": "0001-com-ubuntu-server-jammy",
      "publisher": "Canonical",
      "sharedGalleryImageId": null,
      "sku": "22_04-lts-gen2",
      "version": "latest"
    },
    "osDisk": {
      "caching": "ReadWrite",
      "createOption": "FromImage",
      "deleteOption": "Detach",
      "diffDiskSettings": null,
      "diskSizeGb": 30,
      "encryptionSettings": null,
      "image": null,
      "managedDisk": {
        "diskEncryptionSet": null,
        "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-9642fe6e452040f1/providers/Microsoft.Compute/disks/xfusion-vm_disk1_a5213dd8c1e04cfc8145c858463c4a11",
        "resourceGroup": "kml_rg_main-9642fe6e452040f1",
        "securityProfile": null,
        "storageAccountType": "Standard_LRS"
      },
      "name": "xfusion-vm_disk1_a5213dd8c1e04cfc8145c858463c4a11",
      "osType": "Linux",
      "vhd": null,
      "writeAcceleratorEnabled": null
    }
  },
  "tags": {},
  "timeCreated": "2026-02-18T01:30:26.185573+00:00",
  "type": "Microsoft.Compute/virtualMachines",
  "userData": null,
  "virtualMachineScaleSet": null,
  "vmId": "5cf8cda1-ea8f-44d7-bdfb-ae28733fe16b",
  "zones": null
}

~ ➜  az vm start \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm

~ ➜  az vm get-instance-view \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm \
  --query "{Size:hardwareProfile.vmSize, State:instanceView.statuses[1].displayStatus}"
{
  "Size": "Standard_B2s",
  "State": "VM running"
}

~ ➜  
<img width="1034" height="659" alt="image" src="https://github.com/user-attachments/assets/7355fc86-7d19-40c3-9455-d30adda8155a" />
<img width="1029" height="555" alt="image" src="https://github.com/user-attachments/assets/09466e95-8808-4ab4-bc4e-3a2f9ca970b0" />
<img width="1028" height="577" alt="image" src="https://github.com/user-attachments/assets/a67aa12e-9d6a-44a2-826a-66287742fb54" />
<img width="1037" height="825" alt="image" src="https://github.com/user-attachments/assets/3115eb1a-2257-40d4-a724-86617c7551e7" />
<img width="1035" height="826" alt="image" src="https://github.com/user-attachments/assets/93424d0b-a961-487a-b292-554a03f0bcb2" />
<img width="1037" height="829" alt="image" src="https://github.com/user-attachments/assets/9e8c714c-12d8-48e8-a0b5-6c623a16f935" />
<img width="1037" height="650" alt="image" src="https://github.com/user-attachments/assets/d5abc90e-4c9a-4575-a1ca-f1545543fae4" />
<img width="1031" height="867" alt="image" src="https://github.com/user-attachments/assets/38a6cbf7-df18-4384-89e8-8da11e630de8" />
<img width="1031" height="870" alt="image" src="https://github.com/user-attachments/assets/90502930-4c46-4749-9edb-f7f20fc9ef99" />



