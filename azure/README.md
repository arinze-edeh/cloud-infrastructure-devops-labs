# Azure Labs

~ ➜  az account show
{
  "environmentName": "AzureCloud",
  "homeTenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "isDefault": true,
  "managedByTenants": [],
  "name": "Azure Free Labs",
  "state": "Enabled",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "user": {
    "name": "98335daa-ca9d-438a-b6bf-26b65f657583",
    "type": "servicePrincipal"
  }
}

~ ➜  az account list --output table
Name             CloudName    SubscriptionId                        TenantId                              State    IsDefault
---------------  -----------  ------------------------------------  ------------------------------------  -------  -----------
Azure Free Labs  AzureCloud   f0c3bcdd-5ce2-4fa0-8cf3-41559747512b  54c1a2d3-d100-453c-9636-3a109eb45552  Enabled  True

~ ➜  az vm list --query "[?name=='devops-vm'].{ResourceGroup:resourceGroup}"
[
  {
    "ResourceGroup": "KML_RG_MAIN-C849AEA3729A4D94"
  }
]

~ ➜  az vm update \
  --resource-group KML_RG_MAIN-C849AEA3729A4D94 \
  --name devops-vm \
  --set tags.Environment=dev
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
    "vmSize": "Standard_B1s",
    "vmSizeProperties": null
  },
  "host": null,
  "hostGroup": null,
  "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/KML_RG_MAIN-C849AEA3729A4D94/providers/Microsoft.Compute/virtualMachines/devops-vm",
  "identity": null,
  "instanceView": null,
  "licenseType": null,
  "location": "southcentralus",
  "managedBy": null,
  "name": "devops-vm",
  "networkProfile": {
    "networkApiVersion": null,
    "networkInterfaceConfigurations": null,
    "networkInterfaces": [
      {
        "deleteOption": null,
        "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-c849aea3729a4d94/providers/Microsoft.Network/networkInterfaces/devops-vmVMNic",
        "primary": null,
        "resourceGroup": "kml_rg_main-c849aea3729a4d94"
      }
    ]
  },
  "osProfile": {
    "adminPassword": null,
    "adminUsername": "azureuser",
    "allowExtensionOperations": true,
    "computerName": "devops-vm",
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
            "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCnvyiJ9er7iI8kvKPWERwK90PZlXiI87uI12Ac35nwOOhR/IG09EUzq9oBfra6JM4bRYbIEjGnBjOm8VvX69RIn2FuBbKI85YsDqsOxjupVUlSTPoxFiDm7D/ZK9ljeX0zCAO4M378XZHH2Ai/8bGAeTSgPxgQkSWe5y6kcAfGSnM90RF7d9IfJR/nd+P8B2FuNvYen/kcYUAAkOXz2UAqBemSd4sEmhR51s59oomcyb6AzVBPC/CNm17ltz1wAOp8/9B+gvRpaYM3nHYHw/hM5/K0ptb4vuPLTc7Io8o/ILMl+P8kze9d3KNHuN6RER11zPLvTdvg9AFXjH/MjaLn root@azure-client\n",
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
  "resourceGroup": "KML_RG_MAIN-C849AEA3729A4D94",
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
        "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-c849aea3729a4d94/providers/Microsoft.Compute/disks/devops-vm_disk1_76c484626c8a4b81bf834d8e12d81093",
        "resourceGroup": "kml_rg_main-c849aea3729a4d94",
        "securityProfile": null,
        "storageAccountType": "Standard_LRS"
      },
      "name": "devops-vm_disk1_76c484626c8a4b81bf834d8e12d81093",
      "osType": "Linux",
      "vhd": null,
      "writeAcceleratorEnabled": null
    }
  },
  "tags": {
    "Environment": "dev"
  },
  "timeCreated": "2026-02-19T01:57:53.773685+00:00",
  "type": "Microsoft.Compute/virtualMachines",
  "userData": null,
  "virtualMachineScaleSet": null,
  "vmId": "647651a4-2e5c-46dc-9903-792fcf53bdeb",
  "zones": null
}

~ ➜  az vm show \
  --resource-group KML_RG_MAIN-C849AEA3729A4D94 \
  --name devops-vm \
  --query tags
{
  "Environment": "dev"
}

~ ➜  


<img width="1027" height="849" alt="image" src="https://github.com/user-attachments/assets/79ce7653-1946-4a3d-9b85-b3adbffdc4db" />
<img width="1029" height="847" alt="image" src="https://github.com/user-attachments/assets/c4144f14-6073-4529-b12a-d49e7fe87403" />
<img width="1027" height="790" alt="image" src="https://github.com/user-attachments/assets/be960db0-9bf1-4250-9ce0-af28fb4e0fb1" />
<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/f27a2e76-65fa-46bd-b9b2-dd2199327d5b" />
<img width="1017" height="860" alt="image" src="https://github.com/user-attachments/assets/110218dc-3a5e-4ff2-a9de-674851beb824" />
<img width="1028" height="862" alt="image" src="https://github.com/user-attachments/assets/bfb9b7c6-4480-41a0-8a21-7f5ec658cdda" />
<img width="1028" height="849" alt="image" src="https://github.com/user-attachments/assets/364409ed-51ac-4d51-a744-458ba3e81428" />
<img width="1036" height="446" alt="image" src="https://github.com/user-attachments/assets/776cd497-40aa-4a77-aeda-d54ff0728790" />



