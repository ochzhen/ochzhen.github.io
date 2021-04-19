---
layout: post
title:  "Reference() Function Explained With Examples - ARM Template"
tags: azure arm-template
---

In this post, we will review `reference()` function which often comes handy when working with ARM templates. This function allows to retrieving runtime state of a resource which contains useful information such as URIs, names or other settings.

When I just started poking ARM templates, `reference()` function was a little bit unclear for me. That's why in this post I decided to discuss it and walk though some common use cases.

**Contents:**
* TOC
{:toc}


## Syntax

As defined in documentation, reference function can have from one to three parameters where only the first one is required.

`reference(resourceName or resourceIdentifier, [apiVersion], ['Full'])`

Actually, parameters are well described in the [official documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource?tabs=json#parameters-4){:target="_blank"} but here is a high level overview:
- `resourceName` - use it when referencing a resource in the same template
- `resourceIdentifier` - use it when referencing a resource not in the current template or when resource name is ambiguous. 
Use [resourceId() function](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource?tabs=json#resourceid){:target="_blank"} function to get the unique identifier of a resource.
- `apiVersion` - required only when the resource is not in the current template
- `'Full'` - use to retrieve full resource object, when omitted only properties object is returned


## 'Full' parameter

As already mentioned, 'Full' parameter should be used when we need information that is not in `properties` section.

Probably, the most common use case for using 'Full' option is to retrieve information about Managed Identity associated with a resource, it is returned as `identity` property.

**NOTE:** Without 'Full' parameter `reference()` function returns `properties` object, with 'Full' - the entire runtime state object.


## Determining The Return Value

The main inconvenience with `reference()` function is that properties of a return value are different for every resource type and they are not documented well. Additionally, even for the same resource schema could differ depending on the `apiVersion`.

Let's review some of the ways we can find out the schema of the return value for any resource (storage account, key vault, virtual machine, VMSS, AKS, Function App, etc.).


### [Best] Using ARM Template Outputs Section

This is a bulletproof method that will definitely work. The idea is simple - just output the return value from ARM template. This will allow you to view the actual properties that are being returned from the `reference()` function.

#### Creating ARM Template

Here is an ARM template that you can use, just modify parameter values according to your resources.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "resourceType": {
            "defaultValue": "Microsoft.Storage/storageAccounts",
            "type": "String"
        },
        "resourceName": {
            "defaultValue": "stcontoso",
            "type": "String"
        },
        "apiVersion": {
            "defaultValue": "2019-06-01",
            "type": "String"
        }
    },
    "resources": [],
    "outputs": {
        "resource": {
            "type": "Object",
            "value": "[reference(resourceId(parameters('resourceType'), parameters('resourceName')), parameters('apiVersion'))]"
        },
        "resourceFull": {
            "type": "Object",
            "value": "[reference(resourceId(parameters('resourceType'), parameters('resourceName')), parameters('apiVersion'), 'Full')]"
        }
    }
}
```

#### Applying Template

There are many ways to apply an ARM template file. In this case I would prefer to use "[Custom Template Deployment In Azure Portal](/blog/azure-custom-script-extension-windows#custom-template-deployment-in-azure-portal)".

You should get results quite fast, for me it took less than 10 seconds. Output values are under the corresponding tab, just copy and prettify the JSON.

[![Template deployment outputs](/assets/img/reference-function-arm-templates/deployment-output.png "Template deployment outputs") _Template deployment outputs_](/assets/img/reference-function-arm-templates/deployment-output.png){:target="_blank"}

For a storage account you will get output similar to this:

```json
{
    "apiVersion": "2019-06-01",
    "location": "westus2",
    "sku": {
        "name": "Standard_RAGRS",
        "tier": "Standard"
    },
    "tags": {},
    "kind": "StorageV2",
    "properties": {
        "privateEndpointConnections": [],
        "minimumTlsVersion": "TLS1_2",
        "allowBlobPublicAccess": true,
        "allowSharedKeyAccess": true,
        "networkAcls": {
            "bypass": "AzureServices",
            "virtualNetworkRules": [],
            "ipRules": [],
            "defaultAction": "Allow"
        },
        "supportsHttpsTrafficOnly": true,
        "encryption": {
            "services": {
                "file": {
                    "keyType": "Account",
                    "enabled": true,
                    "lastEnabledTime": "2021-01-28T16:18:46.5479110Z"
                },
                "blob": {
                    "keyType": "Account",
                    "enabled": true,
                    "lastEnabledTime": "2021-01-28T16:18:46.5479110Z"
                }
            },
            "keySource": "Microsoft.Storage"
        },
        "accessTier": "Hot",
        "provisioningState": "Succeeded",
        "creationTime": "2021-01-28T16:18:46.4697950Z",
        "primaryEndpoints": {
            "dfs": "https://stcontoso.dfs.core.windows.net/",
            "web": "https://stcontoso.z5.web.core.windows.net/",
            "blob": "https://stcontoso.blob.core.windows.net/",
            "queue": "https://stcontoso.queue.core.windows.net/",
            "table": "https://stcontoso.table.core.windows.net/",
            "file": "https://stcontoso.file.core.windows.net/"
        },
        "primaryLocation": "westus2",
        "statusOfPrimary": "available",
        "secondaryLocation": "westcentralus",
        "statusOfSecondary": "available",
        "secondaryEndpoints": {
            "dfs": "https://stcontoso-secondary.dfs.core.windows.net/",
            "web": "https://stcontoso-secondary.z5.web.core.windows.net/",
            "blob": "https://stcontoso-secondary.blob.core.windows.net/",
            "queue": "https://stcontoso-secondary.queue.core.windows.net/",
            "table": "https://stcontoso-secondary.table.core.windows.net/"
        }
    },
    "subscriptionId": "8e9b92ce-07d7-45d9-9544-cbfd1d3f1270",
    "resourceGroupName": "rg-contoso",
    "scope": "",
    "resourceId": "Microsoft.Storage/storageAccounts/stcontoso",
    "referenceApiVersion": "2019-06-01",
    "condition": true,
    "isConditionTrue": true,
    "isTemplateResource": false,
    "isAction": false,
    "provisioningOperation": "Read"
}
```


### Azure CLI Command

Another good method to view resource state is to use [`az resource show` command](https://docs.microsoft.com/en-us/cli/azure/resource?view=azure-cli-latest#az_resource_show){:target="_blank"}. You can run this command in Azure Portal Cloud Shell.

**NOTES:**
- Compared to the previous method this one doesn't return additional properties related to ARM template when using 'Full' option and could differ in terms of other fields. But most likely it won't be critical for you use case.
- Need to pass `--id` and optionally `--api-version` arguments, `id` can be found under `Properties` tab of resource page in Azure Portal

```bash
az resource show --id /subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Storage/storageAccounts/{storageAccountName} --api-version 2018-04-01
```

For example, for a storage account you'll get something similar to the following:

```json
{
  "id": "/subscriptions/8e9b92ce-07d7-45d9-9544-cbfd1d3f1270/resourceGroups/rg-contoso/providers/Microsoft.Storage/storageAccounts/stcontoso",
  "identity": null,
  "kind": "StorageV2",
  "location": "westus2",
  "managedBy": null,
  "name": "stcontoso",
  "plan": null,
  "properties": {
    "accessTier": "Hot",
    "allowBlobPublicAccess": true,
    "allowSharedKeyAccess": true,
    "creationTime": "2021-01-28T16:18:46.4697950Z",
    "encryption": {
      "keySource": "Microsoft.Storage",
      "services": {
        "blob": {
          "enabled": true,
          "keyType": "Account",
          "lastEnabledTime": "2021-01-28T16:18:46.5479110Z"
        },
        "file": {
          "enabled": true,
          "keyType": "Account",
          "lastEnabledTime": "2021-01-28T16:18:46.5479110Z"
        }
      }
    },
    "minimumTlsVersion": "TLS1_2",
    "networkAcls": {
      "bypass": "AzureServices",
      "defaultAction": "Allow",
      "ipRules": [],
      "virtualNetworkRules": []
    },
    "primaryEndpoints": {
      "blob": "https://stcontoso.blob.core.windows.net/",
      "dfs": "https://stcontoso.dfs.core.windows.net/",
      "file": "https://stcontoso.file.core.windows.net/",
      "queue": "https://stcontoso.queue.core.windows.net/",
      "table": "https://stcontoso.table.core.windows.net/",
      "web": "https://stcontoso.z5.web.core.windows.net/"
    },
    "primaryLocation": "westus2",
    "privateEndpointConnections": [],
    "provisioningState": "Succeeded",
    "secondaryEndpoints": {
      "blob": "https://stcontoso-secondary.blob.core.windows.net/",
      "dfs": "https://stcontoso-secondary.dfs.core.windows.net/",
      "queue": "https://stcontoso-secondary.queue.core.windows.net/",
      "table": "https://stcontoso-secondary.table.core.windows.net/",
      "web": "https://stcontoso-secondary.z5.web.core.windows.net/"
    },
    "secondaryLocation": "westcentralus",
    "statusOfPrimary": "available",
    "statusOfSecondary": "available",
    "supportsHttpsTrafficOnly": true
  },
  "resourceGroup": "rg-contoso",
  "sku": {
    "capacity": null,
    "family": null,
    "model": null,
    "name": "Standard_RAGRS",
    "size": null,
    "tier": "Standard"
  },
  "tags": {},
  "type": "Microsoft.Storage/storageAccounts"
}
```

### Resource Explorer

Resource Explorer is an alternative way to view the state of a resource through Azure Portal. Search for "Resource Explorer" in the top bar, then select the resource you want to view.

**NOTE:** The main downside is that it uses ***fixed*** `apiVersion` and there is no option to select another one. However, feel free to use it if this version returns properties you are interested in.

[![Resource Explorer](/assets/img/reference-function-arm-templates/resource-explorer.png "Resource Explorer") _Resource Explorer_](/assets/img/reference-function-arm-templates/resource-explorer.png){:target="_blank"}

### ARM Template Reference

Just to mention that some information about the properties you might get from ARM template reference, for example, [Microsoft.Storage storageAccounts template reference](https://docs.microsoft.com/en-us/azure/templates/microsoft.storage/storageaccounts){:target="_blank"}. However, be aware that it doesn't have exactly the same schema since it's used for authoring resources but not retrieving their state, therefore, it might be not suitable for your needs.


## Examples Of reference() Function

In the following subsections let's go over some common use cases you might encounter.

### Reference Existing Resource In The Same Resource Group

To get a resource that is not part of the current template we have to invoke `reference()` function with at least two parameters:

`reference(resourceIdentifier, apiVersion, ['Full'])`

**IMPORTANT:** Parameter `apiVersion` is required.

Resource identifier could be easily retrieved using `resourceId` function. Below there is an example for a storage account, you could specify `'Full'` parameter if needed.

```json
"[reference(resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName')), '2019-06-01')]"
```

### Reference Existing Resource In Another Resource Group

If the resource we want to reference is in a different resource group, the only modification we need to make compared to the previous example is to pass optional `resourceGroupName` parameter to the `resourceId` function.

For a storage account it will look like this:

```json
"[reference(resourceId(parameters('resourceGroupName'), 'Microsoft.Storage/storageAccounts', parameters('storageAccountName')), '2019-06-01')]"
```

### Reference Managed Identity

To retrieve managed identity associated with a resource, simply invoke `reference()` function for this resource with 'Full' parameter. Identity information will be returned inside of `identity` field.

For example, referencing VMSS managed identity `principalId` can be done like this:

```json
"[reference(resourceId('Microsoft.Compute/virtualMachineScaleSets', parameters('vmssName')), '2020-06-01', 'Full').identity.principalId]"
```

The return object will have the following schema:

```json
{
    "apiVersion": "2020-06-01",
    "...": "..."
    "identity": {
        "principalId": "23d6c0c2-adf2-4958-a2da-6fc2d437e0c6",
        "tenantId": "283a576f-45dd-454b-99c0-b2c2680460e7",
        "type": "SystemAssigned"
    },
    "properties": {
         "...": "..."
    }
}
```

### Reference Nested Deployment Output

Let's assume that we have a `Microsoft.Resources/deployments` resource where we do some nested deployment and return information about created resource. Next, we want to get this info in the parent resource.

It is easily achievable using `reference()` function in a way shown below:

```json
"[reference('myNestedDeployment').outputs.someNestedResource.value.name]"
```

**NOTE:** Property `expressionEvaluationOptions` scope must be set to `inner` if we want to use reference function in the nested template. I've included this setting in the example below even though it's not strictly necessary here. [https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates?tabs=azure-powershell#expression-evaluation-scope-in-nested-templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/linked-templates?tabs=azure-powershell#expression-evaluation-scope-in-nested-templates)

Here is an example of a template that references output of a nested deployment:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "resources": [
        {
            "type": "Microsoft.Resources/deployments",
            "apiVersion": "2019-10-01",
            "name": "myNestedDeployment",
            "properties": {
                "mode": "Incremental",
                "expressionEvaluationOptions": {
                    "scope": "inner"
                },
                "template": {
                    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
                    "contentVersion": "1.0.0.0",
                    "resources": [],
                    "outputs": {
                        "someNestedResource": {
                            "type": "object",
                            "value": {
                                "type": "foo",
                                "name": "bar"
                            }
                        }
                    }
                }
            }
        }
    ],
    "outputs": {
        "resource": {
            "type": "Object",
            "value": "[reference('myNestedDeployment')]"
        },
        "resourceFull": {
            "type": "Object",
            "value": "[reference('myNestedDeployment', '2019-10-01', 'Full')]"
        }
    }
}
```

The output from a template above for `resourceFull` object is shown below. Actually, non-Full version is sufficient but I'm including full object just for illustration purposes.

```json
{
    "apiVersion": "2019-10-01",
    "properties": {
        "templateHash": "16948081250938308320",
        "mode": "Incremental",
        "provisioningState": "Succeeded",
        "timestamp": "2021-02-01T16:45:07.8440489Z",
        "duration": "PT0.5157196S",
        "correlationId": "6c6e2fbb-6d45-4f0d-889e-e8cf8d57796d",
        "providers": [],
        "dependencies": [],
        "outputs": {
            "someNestedResource": {
                "type": "Object",
                "value": {
                    "type": "foo",
                    "name": "bar"
                }
            }
        },
        "outputResources": []
    },
    "subscriptionId": "ad51e962-25df-4b3f-bf46-e148c71d33e0",
    "resourceGroupName": "rg-contoso",
    "scope": "",
    "resourceId": "Microsoft.Resources/deployments/myNestedDeployment",
    "referenceApiVersion": "2019-10-01",
    "condition": true,
    "isConditionTrue": true,
    "isTemplateResource": false,
    "isAction": false,
    "provisioningOperation": "Read"
}
```


## Useful Links
- [Reference function documentation](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource?tabs=json#reference){:target="_blank"}
- [ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/){:target="_blank"}
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/what-is-azure-cli){:target="_blank"}
- [https://resources.azure.com/](https://resources.azure.com/){:target="_blank"}
