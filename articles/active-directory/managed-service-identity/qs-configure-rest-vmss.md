---
title: How to configure system and user assigned identities on an Azure VMSS using REST
description: Step by step instructions for configuring a system and user assigned identities on an Azure VMSS using CURL to make REST API calls.
services: active-directory
documentationcenter: 
author: daveba
manager: mtillman
editor: 

ms.service: active-directory
ms.component: msi
ms.devlang: na
ms.topic: conceptual
ms.tgt_pltfrm: na
ms.workload: identity
ms.date: 06/25/2018
ms.author: daveba
---

# Configure Managed Identity on a Virtual Machine Scale Set using REST API calls

[!INCLUDE[preview-notice](../../../includes/active-directory-msi-preview-notice.md)]

Managed Identity provides Azure services with an automatically managed system identity in Azure Active Directory. You can use this identity to authenticate to any service that supports Azure AD authentication, without having credentials in your code. 

In this article, you learn how to perform the following Managed Identity operations on an virtual machine scale set, using CURL to make calls to the Azure Resource Manager REST endpoint:

- Enable and disable the system assigned identity on an Azure virtual machine scale set
- Add and remove a user assigned identity on an Azure virtual machine scale set

## Prerequisites

- If you're unfamiliar with Managed Service Identity, check out the [overview section](overview.md). **Be sure to review the [difference between a system assigned and user assigned identity](overview.md#how-does-it-work)**.
- If you don't already have an Azure account, [sign up for a free account](https://azure.microsoft.com/free/) before continuing.
- To perform the management operations in this article, your account needs the following role assignments:
    - [Virtual Machine Contributor](/azure/role-based-access-control/built-in-roles#virtual-machine-contributor) to create a virtual machine scale set and enable and remove system and/or user assigned managed identity from an virtual machine scale set.
    - [Managed Identity Contributor](/azure/role-based-access-control/built-in-roles#managed-identity-contributor) role to create a user assigned identity.
    - [Managed Identity Operator](/azure/role-based-access-control/built-in-roles#managed-identity-operator) role to assign and remove a user assigned identity from and to a virtual machine scale set.
- If you are using Windows, install the [Windows Subsystem for Linux](https://msdn.microsoft.com/commandline/wsl/about) or use the [Azure Cloud Shell](../../cloud-shell/overview.md) in the Azure portal.
- [Install the Azure CLI local console](/azure/install-azure-cli), if you use the [Windows Subsystem for Linux](https://msdn.microsoft.com/commandline/wsl/about) or a [Linux distribution OS](/cli/azure/install-azure-cli-apt?view=azure-cli-latest).
- If you are using Azure CLI local console, sign in to Azure using `az login` with an account that is associated with the Azure subscription you would like to manage system or user assigned identities.


[!INCLUDE [cloud-shell-try-it.md](../../../includes/cloud-shell-try-it.md)]

## System assigned identity

In this section, you learn how to enable and disable system assigned identity on a virtual machine scale set using CURL to make calls to the Azure Resource Manager REST endpoint.

### Enable system assigned identity during creation of a virtual machine scale set

To create a virtual machine scale set with system assigned identity enabled, you need create a virtual machine scale set and retrieve an access token to use CURL to call the Resource Manager endpoint with the system assigned identity type value.

1. Create a [resource group](../../azure-resource-manager/resource-group-overview.md#terminology) for containment and deployment of your virtual machine scale set and its related resources, using [az group create](/cli/azure/group/#az-group-create). You can skip this step if you already have resource group you would like to use instead:

   ```azurecli-interactive 
   az group create --name myResourceGroup --location westus
   ```

2. Create a [network interface](/cli/azure/network/nic?view=azure-cli-latest#az-network-nic-create) for your virtual machine scale set:

   ```azurecli-interactive
    az network nic create -g myResourceGroup --vnet-name myVnet --subnet mySubnet -n myNic
   ```

3.  Retrieve a Bearer access token, which you will use in the next step in the Authorization header to create your virtual machine scale set with a system assigned managed identity.

   ```azurecli-interactive
   az account get-access-token
   ``` 

4. Create a virtual machine scale set using CURL to call the Azure Resource Manager REST endpoint. The following example creates a virtual machine scale set named *myVMSS* in the *myResourceGroup* with a system assigned identity, as identified in the request body by the value `"identity":{"type":"SystemAssigned"}`. Replace `<ACCESS TOKEN>` with the value you received in the previous step when you requested a Bearer access token and the `<SUBSCRIPTION ID>` value as appropriate for your environment.
 
    ```bash   
    curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PUT -d '{"sku":{"tier":"Standard","capacity":3,"name":"Standard_D1_v2"},"location":"eastus","identity":{"type":"SystemAssigned"},"properties":{"overprovision":true,"virtualMachineProfile":{"storageProfile":{"imageReference":{"sku":"2016-Datacenter","publisher":"MicrosoftWindowsServer","version":"latest","offer":"WindowsServer"},"osDisk":{"caching":"ReadWrite","managedDisk":{"storageAccountType":"Standard_LRS"},"createOption":"FromImage"}},"osProfile":{"computerNamePrefix":"myVMSS","adminUsername":"azureuser","adminPassword":"myPassword12"},"networkProfile":{"networkInterfaceConfigurations":[{"name":"myVMSS","properties":{"primary":true,"enableIPForwarding":true,"ipConfigurations":[{"name":"myVMSS","properties":{"subnet":{"id":"/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myVnet/subnets/mySubnet"}}}]}}]}},"upgradePolicy":{"mode":"Manual"}}}' -H "Content-Type: application/json" -H "Authorization: Bearer <ACCESS TOKEN>"
    ```

### Enable system assigned identity on an existing Azure virtual machine scale set

To enable system assigned identity on an existing virtual machine scale set, you need to acquire an access token and then use CURL to call the Resource Manager REST endpoint to update the identity type.

1. Retrieve a Bearer access token, which you will use in the next step in the Authorization header to create your virtual machine scale set with a system assigned managed identity.

   ```azurecli-interactive
   az account get-access-token
   ```

2. Use the following CURL command to call the Azure Resource Manager REST endpoint to enable system assigned identity on your virtual machine scale set as identified in the request body by the value `{"identity":{"type":"SystemAssigned"}` for a virtual machine scale set named *myVMSS*.  Replace `<ACCESS TOKEN>` with the value you received in the previous step when you requested a Bearer access token and the `<SUBSCRIPTION ID>` value as appropriate for your environment.
   
   > [!IMPORTANT]
   > To ensure you don't delete any existing user assigned managed identities that are assigned to the virtual machine scale set, you need to list the user assigned identities by using this CURL command: `curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.Compute/virtualMachineScaleSets/<VMSS NAME>?api-version=2018-06-01' -H "Authorization: Bearer <ACCESS TOKEN>"`. If you have any user assigned identities assigned to the virtual machine scale set as identified in the `identity` value in the response, skip to step 3 that shows you how to retain user assigned identities while enabling system assigned identity on your virtual machine scale set.

   ```bash
    curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PATCH -d '{"identity":{"type":"SystemAssigned"}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

3. To enable system assigned identity on a virtual machine scale set with existing user assigned identities, you need to add `SystemAssigned` to the `type` value.  
   
   For example, if your virtual machine scale set has the user assigned identities `ID1` and `ID2` assigned to it, and you would like to add system assigned identity to the virtual machine scale set, use the following CURL call. Replace `<ACCESS TOKEN>` and `<SUBSCRIPTION ID>` with values appropriate to your environment.

   API version `2018-06-01` stores user assigned identities in the `userAssignedIdentities` value in a dictionary format as opposed to the `identityIds` value in an array format used in API version `2017-12-01` and earlier versions.
   
   **API VERSION 2018-06-01**

   ```bash
   curl -v 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PATCH -d '{"identity":{"type":"SystemAssigned,UserAssigned", "userAssignedIdentities":{"/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1":{},"/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID2":{}}}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```
   
   **API VERSION 2017-12-01 and earlier**

   ```bash
   curl -v 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2017-12-01' -X PATCH -d '{"identity":{"type":"SystemAssigned","UserAssigned", "identityIds":["/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1","/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID2"]}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

### Disable system assigned identity from an Azure virtual machine scale set

To disable a system assigned identity on an existing virtual machine scale set, you need to acquire an access token and then use CURL to call the Resource Manager REST endpoint to update the identity type to `None`.

1. Retrieve a Bearer access token, which you will use in the next step in the Authorization header to create your virtual machine scale set with a system assigned managed identity.

   ```azurecli-interactive
   az account get-access-token
   ```

2. Update the virtual machine scale set using CURL to call the Azure Resource Manager REST endpoint to disable system assigned identity.  The following example disables system assigned identity as identified in the request body by the value `{"identity":{"type":"None"}}` from a virtual machine scale set named *myVMSS*.  Replace `<ACCESS TOKEN>` with the value you received in the previous step when you requested a Bearer access token and the `<SUBSCRIPTION ID>` value as appropriate for your environment.

   > [!IMPORTANT]
   > To ensure you don't delete any existing user assigned managed identities that are assigned to the virtual machine scale set, you need to list the user assigned identities by using this CURL command: `curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.Compute/virtualMachineScaleSets/<VMSS NAME>?api-version=2018-06-01' -H "Authorization: Bearer <ACCESS TOKEN>"`. If you have any user assigned identity assigned to the virtual machine scale set, skip to step 3 that shows you how retain the user assigned identities while removing the system assigned identity from your virtual machine scale set.

   ```bash
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PATCH -d '{"identity":{"type":"None"}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

3. To remove system assigned identity from a virtual machine scale set that has user assigned identities, remove `SystemAssigned` from the `{"identity":{"type:" "}}` value while keeping the `UserAssigned` value and the `userAssignedIdentities` dictionary values if you are using **API version 2018-06-01**. If you are using **API version 2017-12-01** or earlier, keep the `identityIds` array.

## User assigned identity

In this section, you learn how to add and remove user assigned identity on a virtual machine scale set using CURL to make calls to the Azure Resource Manager REST endpoint.

### Assign a user assigned identity during the creation of a virtual machine scale set

1. Retrieve a Bearer access token, which you will use in the next step in the Authorization header to create your virtual machine scale set with a system assigned managed identity.

   ```azurecli-interactive
   az account get-access-token
   ```

2. Create a [network interface](/cli/azure/network/nic?view=azure-cli-latest#az-network-nic-create) for your virtual machine scale set:

   ```azurecli-interactive
    az network nic create -g myResourceGroup --vnet-name myVnet --subnet mySubnet -n myNic
   ```

3.  Retrieve a Bearer access token, which you will use in the next step in the Authorization header to create your virtual machine scale set with a system assigned managed identity.

   ```azurecli-interactive
   az account get-access-token
   ``` 

4. Create a user assigned identity using the instructions found here: [Create a user assigned managed identity](how-to-manage-ua-identity-rest.md#create-a-user-assigned-managed-identity).

5. Create a virtual machine scale set using CURL to call the Azure Resource Manager REST endpoint. The following example creates a virtual machine scale set named *myVMSS* in the resource group *myResourceGroup* with a user assigned identity `ID1`, as identified in the request body by the value `"identity":{"type":"UserAssigned"}`. Replace `<ACCESS TOKEN>` with the value you received in the previous step when you requested a Bearer access token and the `<SUBSCRIPTION ID>` value as appropriate for your environment.
 
   **API VERSION 2018-06-01**

   ```bash   
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PUT -d '{"sku":{"tier":"Standard","capacity":3,"name":"Standard_D1_v2"},"location":"eastus",{"identity":{"type":"UserAssigned", "userAssignedIdentities":{"/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1":{}}}},"properties":{"overprovision":true,"virtualMachineProfile":{"storageProfile":{"imageReference":{"sku":"2016-Datacenter","publisher":"MicrosoftWindowsServer","version":"latest","offer":"WindowsServer"},"osDisk":{"caching":"ReadWrite","managedDisk":{"storageAccountType":"Standard_LRS"},"createOption":"FromImage"}},"osProfile":{"computerNamePrefix":"myVMSS","adminUsername":"azureuser","adminPassword":"myPassword12"},"networkProfile":{"networkInterfaceConfigurations":[{"name":"myVMSS","properties":{"primary":true,"enableIPForwarding":true,"ipConfigurations":[{"name":"myVMSS","properties":{"subnet":{"id":"/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myVnet/subnets/mySubnet"}}}]}}]}},"upgradePolicy":{"mode":"Manual"}}}' -H "Content-Type: application/json" -H "Authorization: Bearer <ACCESS TOKEN>"
   ```

   **API VERSION 2017-12-01 and earlier**

   ```bash   
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2017-12-01' -X PUT -d '{"sku":{"tier":"Standard","capacity":3,"name":"Standard_D1_v2"},"location":"eastus",{"identity":{"type":"UserAssigned", "identityIds":["/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1"]}},"properties":{"overprovision":true,"virtualMachineProfile":{"storageProfile":{"imageReference":{"sku":"2016-Datacenter","publisher":"MicrosoftWindowsServer","version":"latest","offer":"WindowsServer"},"osDisk":{"caching":"ReadWrite","managedDisk":{"storageAccountType":"Standard_LRS"},"createOption":"FromImage"}},"osProfile":{"computerNamePrefix":"myVMSS","adminUsername":"azureuser","adminPassword":"myPassword12"},"networkProfile":{"networkInterfaceConfigurations":[{"name":"myVMSS","properties":{"primary":true,"enableIPForwarding":true,"ipConfigurations":[{"name":"myVMSS","properties":{"subnet":{"id":"/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Network/virtualNetworks/myVnet/subnets/mySubnet"}}}]}}]}},"upgradePolicy":{"mode":"Manual"}}}' -H "Content-Type: application/json" -H "Authorization: Bearer <ACCESS TOKEN>"
   ```

### Assign a user assigned identity to an existing Azure virtual machine scale set

1. Retrieve a Bearer access token, which you will use in the next step in the Authorization header to create your virtual machine scale set with a system assigned managed identity.

   ```azurecli-interactive
   az account get-access-token
   ```

2.  Create a user assigned identity using the instructions found here, [Create a user assigned managed identity](how-to-manage-ua-identity-rest.md#create-a-user-assigned-managed-identity).

3. To ensure you don't delete existing user or system assigned managed identities that are assigned to the virtual machine scale set, you need to list the identity types assigned to the virtual machine scale set by using the following CURL command. If you have managed identities assigned to the virtual machine scale set, they are listed in the `identity` value.
 
   ```bash
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.Compute/virtualMachineScaleSets/<VMSS NAME>?api-version=2018-06-01' -H "Authorization: Bearer <ACCESS TOKEN>"
   ```

4. If you don't have any user or system assigned identities assigned to your virtual machine scale set, use the following CURL command to call the Azure Resource Manager REST endpoint to assign the first user assigned identity to the virtual machine scale set.  If you have a user or system assigned identity(s) assigned to the virtual machine scale set, skip to step 5 that shows you how to add multiple user assigned identities to a virtual machine scale set while also maintaining the system assigned identity.

   The following example assigns a user assigned identity, `ID1` to a virtual machine scale set named *myVMSS* in the resource group *myResourceGroup*.  Replace `<ACCESS TOKEN>` with the value you received in the previous step when you requested a Bearer access token and the `<SUBSCRIPTION ID>` value as appropriate for your environment.

   **API VERSION 2018-06-01**

    ```bash
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-12-01' -X PATCH -d '{"identity":{"type":"userAssigned", "userAssignedIdentities":{"/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1":{}}}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```   
    
   **API VERSION 2017-12-01 and earlier**

   ```bash
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2017-12-01' -X PATCH -d '{"identity":{"type":"userAssigned", "identityIds":["/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1"]}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

5. If you have an existing user assigned or system assigned identity assigned to your virtual machine scale set:
   
   **API VERSION 2018-06-01**

   Add the user assigned identity to the `userAssignedIdentities` dictionary value.

   For example, if you have system assigned identity and the user assigned identity `ID1` currently assigned to your virtual machine scale and would like to add the user assigned identity `ID2` to it:

   ```bash
   curl  'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PATCH -d '{"identity":{"type":"SystemAssigned, UserAssigned", "userAssignedIdentities":{"/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1":{},"/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID2":{}}}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

   **API VERSION 2017-12-01 and earlier**

   Retain the user assigned identities you would like to keep in the `identityIds` array value while adding the new user assigned identity.

   For example, if you have system assigned identity and the user assigned identity `ID1` currently assigned to your virtual machine scale set and would like to add the user identity `ID2` to it: 

   ```bash
   curl  'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2017-12-01' -X PATCH -d '{"identity":{"type":"SystemAssigned, UserAssigned", "identityIds":["/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1","/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID2"]}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

### Remove a user assigned identity from a virtual machine scale set

1. Retrieve a Bearer access token, which you will use in the next step in the Authorization header to create your virtual machine scale set with a system assigned managed identity.

   ```azurecli-interactive
   az account get-access-token
   ```

2. To ensure you don't delete any existing user assigned managed identities that you would like to keep assigned to the virtual machine scale set or remove the system assigned identity, you need to list the managed identities by using the following CURL command: 
   
   ```bash
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/<RESOURCE GROUP>/providers/Microsoft.Compute/virtualMachineScaleSets/<VMSS NAME>?api-version=2018-06-01' -H "Authorization: Bearer <ACCESS TOKEN>" 
   ```
   
   If you have managed identities assigned to the VM, they are listed in the response in the `identity` value. 
    
   For example, if you have user assigned identities `ID1` and `ID2` assigned to your virtual machine scale set, and you only want to keep `ID1` assigned and retain the system assigned identity:

   **API VERSION 2018-06-01**

   Add `null` to the user assigned identity you would like to remove:

   ```bash
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PATCH -d '{"identity":{"type":"SystemAssigned, UserAssigned", "userAssignedIdentities":{"/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1":{}}}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

   **API VERSION 2017-12-01 and earlier**

   Retain only the user assigned identity(s) you would like to keep in the `identityIds` array:   

   ```bash
   curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2017-12-01' -X PATCH -d '{"identity":{"type":"SystemAssigned","UserAssigned", "identityIds":["/subscriptions/<SUBSCRIPTION ID>/resourcegroups/myResourceGroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/ID1"]}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
   ```

If your virtual machine scale set has both system assigned and user assigned identities, you can remove all the user assigned identities by switching to use only system assigned using the following command:

```bash
curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PATCH -d '{"identity":{"type":"SystemAssigned"}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
```
    
If your virtual machine scale set has only user assigned identities and you would like to remove them all, use the following command:

```bash
curl 'https://management.azure.com/subscriptions/<SUBSCRIPTION ID>/resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myVMSS?api-version=2018-06-01' -X PATCH -d '{"identity":{"type":"None"}}' -H "Content-Type: application/json" -H Authorization:"Bearer <ACCESS TOKEN>"
```

## Next steps

For information on how to create, list, or delete user assigned using REST see:

- [Create, list or delete a user assigned identity using REST API calls](how-to-manage-ua-identity-rest.md)