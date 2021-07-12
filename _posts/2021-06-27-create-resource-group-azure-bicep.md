---
layout: post
title:  "Create Resource Group With Azure Bicep and Deploy Resources In It"
tags: azure-bicep
---


In this short post we will discuss how to deploy a resource group and (optionally) create resources inside of this resource group all during one deployment.

Additionally, we cover different values of `targetScope` for the deployment: *subscription*, *managementGroup*, and *tenant*.

**To create a resource group, target scope of the deployment should be set to `subscription`, and a resource of type `Microsoft.Resources/resourceGroups` must be created. If target scope is not `subscription`, then resource group should be deployed using a module.**

The ability to create a resource group from a template is useful because it eliminates the need to perform creation of resource group manually and allows managing larger deployments.


**Contents:**
* TOC
{:toc}


## Overview

So, we want to **define a resource group in a bicep file** in order to have it deployed automatically. This is a basic case and it is covered in the [Minimal Example](#minimal-example) section.

Additionally, let's assume that we **also want to deploy some resources** into this resource group, in our case it will be a storage account. This scenario is discussed in [Deploying Resource Group and Storage Account](#deploying-resource-group-and-storage-account).

The post also goes briefly about [Deployment Target Scopes](#deployment-target-scopes) and how they relate to a resource group deployment. Later, this knowledge helps in understanding slightly more advanced cases of deploying at [subscription](#target-scope-subscription), [managementGroup and tenant](#target-scopes-managementgroup-and-tenant) target scopes.

In summary, this post talks about the deployment of the following resource types in a combination with different target scopes:
- `Microsoft.Resources/resourceGroups` - the resource group we create
- `Microsoft.Storage/storageAccounts` - some resource to deploy into the resource group


## Minimal Example

Deploying a resource group in a Bicep file is straightforward. The simplest template should contain an assignment of `subscription` target scope and definition of `Microsoft.Resources/resourceGroups` resource.

```tsx
// =========== main.bicep ===========
targetScope = 'subscription'

resource rg 'Microsoft.Resources/resourceGroups@2021-01-01' = {
  name: 'rg-contoso'
  location: 'westus2'
}
```


## Deployment Target Scopes

Each Bicep file has a [targetScope](https://github.com/Azure/bicep/blob/main/docs/spec/resource-scopes.md#declaring-the-target-scope){:target="_blank"} which is set implicitly or explicitly, it is used to perform validation and checks on the resources in the template.

When deploying a resource group, your target scope likely to be `subscription` or higher, because target scope `resourceGroup` makes less sense when creating a resource group in a template.

Target scope possible values are:

- `resourceGroup` (default)
- `subscription`
- `managementGroup`
- `tenant`

Setting target scope is done by using `targetScope` keyword and a scope name, for example:

```tsx
targetScope = 'subscription'
```

In the following sections we will cover two cases:

- Deploying main bicep file at the `subscription` target scope
- Deploying main bicep file at the `managementGroup` and `tenant` target scopes

## Deploying Resource Group and Storage Account

In the [Minimal Example](#minimal-example) we saw how to deploy just a resource group. In this part of the post, we are going to **also deploy a storage account** in the newly created resource group.

### Target Scope "subscription"

As already discussed, we need to deploy two resources: a resource group and a storage account.

- Resource group resource (`Microsoft.Resources/resourceGroups`) has to be deployed at the `subscription` scope. Since the scope of the bicep file is `subscription` as well, we can simply **deploy resource group in the main file**.
- Storage account resource has to be deployed at the `resourceGroup` scope. Therefore, we need to **use modules** to deploy it at a scope different from the main file.


#### Bicep Files

Our `main.bicep` file contains deployment of a resource group and a storage account module:

```tsx
// =========== main.bicep ===========

// Setting target scope
targetScope = 'subscription'

// Creating resource group
resource rg 'Microsoft.Resources/resourceGroups@2021-01-01' = {
  name: 'rg-contoso'
  location: 'westus2'
}

// Deploying storage account using module
module stg './storage.bicep' = {
  name: 'storageDeployment'
  scope: rg    // Deployed in the scope of resource group we created above
  params: {
    storageAccountName: 'stcontoso'
  }
}
```

```tsx
// =========== storage.bicep ===========

// targetScope = 'resourceGroup' - not needed since it is the default value

param storageAccountName string

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

#### Compiled ARM Template

Only if you are curious how Bicep files above look like when compiled into an ARM template, see JSON below.

**NOTES:**

- Storage account is deployed in a nested deployment `Microsoft.Resources/deployments` (line **12**)
- Scope of the nested deployment is set via `resourceGroup` property (line **15**)
- `$schema` property has two values: `../subscriptionDeploymentTemplate.json#` for the parent template (line **2**) and `../deploymentTemplate.json#` for nested deployment (line **27**)

{% highlight json linenos %}
{
  "$schema": "https://schema.management.azure.com/schemas/2018-05-01/subscriptionDeploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "resources": [
    {
      "type": "Microsoft.Resources/resourceGroups",
      "apiVersion": "2021-01-01",
      "name": "rg-contoso",
      "location": "westus2"
    },
    {
      "type": "Microsoft.Resources/deployments",
      "apiVersion": "2019-10-01",
      "name": "storageDeployment",
      "resourceGroup": "rg-contoso",
      "properties": {
        "expressionEvaluationOptions": {
          "scope": "inner"
        },
        "mode": "Incremental",
        "parameters": {
          "storageAccountName": {
            "value": "stcontoso"
          }
        },
        "template": {
          "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
          "contentVersion": "1.0.0.0",
          "parameters": {
            "storageAccountName": {
              "type": "string"
            }
          },
          "resources": [
            {
              "type": "Microsoft.Storage/storageAccounts",
              "apiVersion": "2021-02-01",
              "name": "[parameters('storageAccountName')]",
              "location": "[resourceGroup().location]",
              "sku": {
                "name": "Standard_LRS"
              },
              "kind": "StorageV2"
            }
          ]
        }
      },
      "dependsOn": [
        "[subscriptionResourceId('Microsoft.Resources/resourceGroups', 'rg-contoso')]"
      ]
    }
  ]
}
{% endhighlight %}


### Target Scopes "managementGroup" and "tenant"

Now, how to deploy a resource group if our **deployment targetScope is not subscription**? - **Use modules** to specify the right scope for a resource group.

Bicep modules have an optional `scope` property which can be used to specify a scope different from the bicep file where module is consumed. We've already used this functionality for the storage account in the previous section.

#### Modules: Resource Group and Storage Account

Since our main file's `targetScope` is not `subscription`, we need to put resource group into a separate module (just another bicep file).

These modules are the same both for `managementGroup` and `tenant` target scopes 

Below is our `resource-group.bicep` file, it deploys the resource group and a storage account module. This file is **identical** to `main.bicep` from the previous chapter where we deployed at the `subscription` target scope.

```tsx
// =========== resource-group.bicep ===========

targetScope = 'subscription'    // Resource group must be deployed under 'subscription' scope

param resourceGroupName string
param location string

resource rg 'Microsoft.Resources/resourceGroups@2021-01-01' = {
  name: resourceGroupName
  location: location
}

module stg './storage.bicep' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: 'stcontoso'
  }
  scope: rg
}
```

The `storage.bicep` file is the same as in the previous section - simple storage account declaration.

```tsx
// =========== storage.bicep ===========

// targetScope = 'resourceGroup' - not needed since it is the default value

param storageAccountName string

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

#### Deploying at "managementGroup" or "tenant" targetScopes

Now, we just need to consume `resource-group.bicep` module inside of our main bicep file.

Important point is to specify the correct scope for the module, this should be **subscription** for resource group.

Deploying at the `tenant` scope is almost identical to deploying at the management group scope. The only difference is the `targetScope` of the file, and that's it.

**NOTE:** You need permissions at the tenant level to be able to deploy at the `tenant` scope. Read more about the  [required access](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/deploy-to-tenant?tabs=azure-cli#required-access){:target="_blank"}.

```tsx
// =========== main.bicep ===========

targetScope = 'managementGroup'  // Setting target scope
// targetScope = 'tenant' - if deploying at the tenant scop

param subscriptionId string
param resourceGroupName string = 'rg-contoso'
param dateTime string = utcNow()  // Just to make resource group deployment name unique

// Deploying the resource group and a storage account inside of it
module rg './resource-group.bicep' = {
  name: 'resourceGroupDeployment-${dateTime}'
  params: {
    resourceGroupName: resourceGroupName
    location: 'westus2'
  }
  scope: subscription(subscriptionId)  // Passing subscription scope
}
```


## Related Posts

- [Parameters In Azure Bicep - Ultimate Guide With Examples](/blog/azure-bicep-parameters)
- [Variables In Azure Bicep - From Basics To Advanced](/blog/azure-bicep-variables)
- [Reference New Or Existing Resource In Azure Bicep](/blog/reference-new-or-existing-resource-in-azure-bicep)
- [Child Resources In Azure Bicep - 3 Ways To Declare, Loops, Conditions](/blog/child-resources-in-azure-bicep)
- [Deploy Azure Bicep In YAML and Classic Release Pipelines (CI/CD) - Azure DevOps](/blog/deploy-azure-bicep-in-cicd-pipeline)
- [Reference() Function Explained With Examples - ARM Template](/blog/reference-function-arm-template)


## Useful Links

- [resource-scopes.md - Azure Bicep](https://github.com/Azure/bicep/blob/main/docs/spec/resource-scopes.md){:target="_blank"}
- [Bicep modules - Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/bicep-modules){:target="_blank"}
- [Microsoft.Resources/resourceGroups](https://docs.microsoft.com/en-us/azure/templates/microsoft.resources/resourcegroups?tabs=bicep){:target="_blank"}