---
title: Configure customer-managed keys for Azure API for FHIR
description: Bring your own key feature supported in Azure API for FHIR via Azure Cosmos DB
services: healthcare-apis
author: expekesheth
ms.service: azure-health-data-services
ms.subservice: fhir
ms.topic: overview
ms.date: 09/27/2023
ms.author: kesheth
ms.custom: devx-track-azurepowershell, devx-track-azurecli, devx-track-arm-template
ms.devlang: azurecli
---

# Configure customer-managed keys at rest

[!INCLUDE[retirement banner](../includes/healthcare-apis-azure-api-fhir-retirement.md)]

When you create a new Azure API for FHIR&reg; account, your data is encrypted using Microsoft-managed keys by default. Now you can add a second layer of encryption for the data using a key that you choose and manage yourself.

In Azure, this is typically accomplished using an encryption key in the customer's Azure Key Vault. Azure SQL, Azure Storage, and Azure Cosmos DB are some examples that provide this capability. Azure API for FHIR leverages this support from Azure Cosmos DB. When you create an account, you have the option to specify an Azure Key Vault key URI. This key is passed on to Azure Cosmos DB when the DB account is provisioned. When a Fast Healthcare Interoperability Resources (FHIR) request is made, Azure Cosmos DB fetches your key and uses it to encrypt/decrypt the data. 

To get started, refer to the following links:

- [Register the Azure Cosmos DB resource provider for your Azure subscription](/azure/cosmos-db/how-to-setup-cmk#register-resource-provider) 
- [Configure your Azure Key Vault instance](/azure/cosmos-db/how-to-setup-cmk#configure-your-azure-key-vault-instance)
- [Add an access policy to your Azure Key Vault instance](/azure/cosmos-db/how-to-setup-cmk#add-access-policy)
- [Generate a key in Azure Key Vault](/azure/cosmos-db/how-to-setup-cmk#generate-a-key-in-azure-key-vault)

## Using Azure portal

When creating your Azure API for FHIR account on Azure portal, you'll notice a **Data Encryption** configuration option under the **Database Settings** on the **Additional Settings** tab. By default, the service-managed key option is selected.

> [!Important]
> The data encryption option is only available when the Azure API for FHIR is created and cannot be changed afterwards. However, you can view and update the encryption key if the **Customer-managed key** option is selected. 


You can choose your key from the KeyPicker:

:::image type="content" source="media/bring-your-own-key/bring-your-own-key-keypicker.png" alt-text="KeyPicker":::

You can also specify your Azure Key Vault key here by selecting **Customer-managed key** option.
 
You can also enter the key URI as shown in the following.

:::image type="content" source="media/bring-your-own-key/bring-your-own-key-create.png" alt-text="Create Azure API for FHIR":::

> [!Important]
> Ensure all permissions for Azure Key Vault are set appropriately. For more information, see [Add an access policy to your Azure Key Vault instance](/azure/cosmos-db/how-to-setup-cmk#add-access-policy). 
Additionally, ensure that the soft delete is enabled in the properties of the Key Vault. Not completing these steps will result in a deployment error. For more information, see [Verify if soft delete is enabled on a key vault and enable soft delete](/azure/key-vault/general/key-vault-recovery?tabs=azure-portal#verify-if-soft-delete-is-enabled-on-a-key-vault-and-enable-soft-delete).

> [!NOTE]
> Using customer-managed keys in the Brazil South, East Asia, and Southeast Asia Azure regions requires an Enterprise Application ID generated by Microsoft. You can request an Enterprise Application ID by creating a one-time support ticket through the Azure portal. After receiving the Application ID, follow [the instructions to register the application](/azure/cosmos-db/how-to-setup-cross-tenant-customer-managed-keys?tabs=azure-portal#the-customer-grants-the-service-providers-app-access-to-the-key-in-the-key-vault).


For existing FHIR accounts, you can view the key encryption choice (**Service-managed key** or **Customer-managed key**) in the **Database** blade as follows. The configuration option can't be modified once it's selected. However, you can modify and update your key.

:::image type="content" source="media/bring-your-own-key/bring-your-own-key-database.png" alt-text="Database":::

In addition, you can create a new version of the specified key, after which your data gets encrypted with the new version without any service interruption. You can also remove access to the key to remove access to the data. When the key is disabled, queries will result in an error. If the key is re-enabled, queries will succeed again.

## Using Azure PowerShell

With your Azure Key Vault key URI, you can configure CMK using PowerShell by running the following PowerShell command.

```powershell
New-AzHealthcareApisService
    -Name "myService"
    -Kind "fhir-R4"
    -ResourceGroupName "myResourceGroup"
    -Location "westus2"
    -CosmosKeyVaultKeyUri "https://<my-vault>.vault.azure.net/keys/<my-key>"
```

## Using Azure CLI

As with PowerShell method, you can configure CMK by passing your Azure Key Vault key URI under the `key-vault-key-uri` parameter and running the following CLI command. 

```azurecli-interactive
az healthcareapis service create
    --resource-group "myResourceGroup"
    --resource-name "myResourceName"
    --kind "fhir-R4"
    --location "westus2"
    --cosmos-db-configuration key-vault-key-uri="https://<my-vault>.vault.azure.net/keys/<my-key>"

```
## Using Azure Resource Manager Template

With your Azure Key Vault key URI, you can configure CMK by passing it under the **keyVaultKeyUri** property in the **properties** object.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "services_myService_name": {
            "defaultValue": "myService",
            "type": "String"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.HealthcareApis/services",
            "apiVersion": "2020-03-30",
            "name": "[parameters('services_myService_name')]",
            "location": "westus2",
            "kind": "fhir-R4",
            "properties": {
                "accessPolicies": [],
                "cosmosDbConfiguration": {
                    "offerThroughput": 400,
                    "keyVaultKeyUri": "https://<my-vault>.vault.azure.net/keys/<my-key>"
                },
                "authenticationConfiguration": {
                    "authority": "https://login.microsoftonline.com/72f988bf-86f1-41af-91ab-2d7cd011db47",
                    "audience": "[concat('https://', parameters('services_myService_name'), '.azurehealthcareapis.com')]",
                    "smartProxyEnabled": false
                },
                "corsConfiguration": {
                    "origins": [],
                    "headers": [],
                    "methods": [],
                    "maxAge": 0,
                    "allowCredentials": false
                }
            }
        }
    ]
}
```

And you can deploy the template with the following PowerShell script.

```powershell
$resourceGroupName = "myResourceGroup"
$accountName = "mycosmosaccount"
$accountLocation = "West US 2"
$keyVaultKeyUri = "https://<my-vault>.vault.azure.net/keys/<my-key>"

New-AzResourceGroupDeployment `
    -ResourceGroupName $resourceGroupName `
    -TemplateFile "deploy.json" `
    -accountName $accountName `
    -location $accountLocation `
    -keyVaultKeyUri $keyVaultKeyUri
```
## Frequently asked questions

**Is the 'cosmosdb_key_vault_key_versionless_id' used in the FHIR API designed to connect to FHIR service managed Cosmos DB?**

Yes, when you are enabling customer-managed keys on FHIR APIs, selecting 'cosmosdb_key_vault_key_versionless_id' connects to FHIR service managed Cosmos DB.

## Next steps

In this article, you learned how to configure customer-managed keys at rest using the Azure portal, PowerShell, CLI, and Resource Manager Template. You can refer to the Azure Cosmos DB FAQ section for more information. 
 
>[!div class="nextstepaction"]
>[Azure Cosmos DB: how to setup CMK](/azure/cosmos-db/how-to-setup-cmk#frequently-asked-questions)

[!INCLUDE[FHIR trademark statement](../includes/healthcare-apis-fhir-trademark.md)]