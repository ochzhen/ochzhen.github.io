---
layout: post
title:  "Reference New Or Existing Resource In Azure Bicep"
tags: bicep
---

In a traditional ARM template [reference function](/blog/reference-function-arm-template) provides capability to retrieve runtime state object of a resource. This might be useful to get FQDNs, properties, managed identity info.

Not surprisingly, we often want similar capabilities while working with Azure Bicep.

**Referencing existing resources in Bicep is achieved by declaring a symbolic name for the existing resource and using it to retrieve needed properties. ARM template `reference` function can also be used, however, it is less recommended.**

So, Bicep not only allows using the existing `reference` function but provides even more convenient and clear syntax to achieve that.

In this post we will discuss how to reference resources deployed in the **same or different resource group**, **same or different subscription**, touch on referencing **child resources** and resources deployed as part of a **module**.

**Contents:**
* TOC
{:toc}


## Symbolic Name and State Object of a Resource

Each resource in a Bicep file has a **symbolic name** which is used to get runtime state object of the resource.

**NOTE:** Bicep extension for Visual Studio Code knows returned object's structure based on the `apiVersion` and provides great code suggestions. This significantly simplifies the process of determining where needed properties are located.

In the following example `stg` is a symbolic name. After being declared, `stg` can be used to retrieve information about the resource.

```tsx
// Declaring symbolic name stg
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: 'stcontoso'
  ...
}

// Using stg to get property of a resource
output storageKind string = stg.kind
```

The above code gets compiled into the ARM template section below. Small notes:

- It uses `reference` function with ['Full' parameter](/blog/reference-function-arm-template#full-parameter) under the hood
- Symbolic name declaration from Bicep doesn't have any equivalent in the ARM template

```json
{
  "...": "...",
  "outputs": {
    "storageKind": {
      "type": "string",
      "value": "[reference(resourceId('Microsoft.Storage/storageAccounts', 'stcontoso'), '2021-02-01', 'full').kind]"
    }
  }
}
```


## Keyword *existing* and *scope* Property

As already mentioned, each resource in Bicep has a symbolic name which is used to reference the resource. This is obvious when we deploy the resource in the same Bicep file.

But when we are referencing an already existing resource, we should know about the following:

- Keyword `existing` is used when we want a symbolic name for a resource which is not deployed as part of the template but was already created.

- Property `scope` allows specifying where this existing resource lives. This comes into play when we want to reference a resource in a different scope.

Notes about `scope` property:

- It is optional, if not specified, default value will be applied
- Default value is the [scope of the bicep file](https://github.com/Azure/bicep/blob/main/docs/spec/resource-scopes.md#declaring-the-target-scope){:target="_blank"}
- Each resource type has its permitted scope, for example, storage account only accepts *resourceGroup* scope but not *subscription* or *managementGroup*

Some of the following sections will use scope property to correctly reference an existing resource.

```tsx
// Note the use of "existing" keyword
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: ...   // Required
  scope: ...  // Optional
}
```


## Reference Resource Deployed In The Same Template

Let's start with the most basic and simple case where we want to retrieve properties of a resource which is deployed in the same template.

Below is an example how to get the primary endpoint for blob of a storage account that we just deployed.

```tsx
param storageAccountName string = 'stcontoso'

// Declaring our resource here
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  kind: 'StorageV2'
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
}

// Retrieving resource property
output blobEndpoint string = stg.properties.primaryEndpoints.blob

// Returns https://stcontosoo.blob.core.windows.net/
```


## Reference Existing Resource In The Same Resource Group

In the previous section we deployed a simple storage account. Now, let's assume that we deploy a separate template in the scope of the same resource group and want to get blob primary endpoint.

In Bicep referencing existing resource in the same resource group is easy and clean:

- Symbolic name declaration contains keyword `existing`
- `name` property must be provided

```tsx
param storageAccountName string = 'stcontoso'

// Creating a symbolic name for an existing resource
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: storageAccountName
}

// Retrieving resource property
output blobEndpoint string = stg.properties.primaryEndpoints.blob
```


## Reference Existing Resource In a Different Resource Group

If we have another template which is deployed in the scope of another resource group but still in the same subscription, then we can use `resourceGroup` function to specify the correct scope.

```tsx
param storageAccountName string = 'stcontoso'
param storageResourceGroupName string = 'rg-bicep' // Resource group where the storage account exists

// Creating a symbolic name for an existing resource
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: storageAccountName
  scope: resourceGroup(storageResourceGroupName)
}

// Retrieving resource property
output blobEndpoint string = stg.properties.primaryEndpoints.blob
```

The bicep file above mainly boils down to the following expression which is significantly harder to understand.

```json
{
  "...": "...",
  "outputs": {
    "blobEndpoint": {
      "type": "string",
      "value": "[reference(extensionResourceId(format('/subscriptions/{0}/resourceGroups/{1}', subscription().subscriptionId, parameters('storageResourceGroupName')), 'Microsoft.Storage/storageAccounts', parameters('storageAccountName')), '2021-02-01').primaryEndpoints.blob]"
    }
  }
}
```


## Reference Existing Resource In a Different Resource Group and Subscription

If the resource group of the existing resource is located in a different subscription, then we can use another overload of `resourceGroup` function which accepts `subscriptionId`.

Please find an example below. The resulting ARM template is similar to the one from the previous section but now we specify subscriptionId explicitly.

```tsx
param storageAccountName string = 'stcontoso'
param storageResourceGroupName string = 'rg-bicep'
param storageSubscriptionId string = '23738f0d-c0e9-4531-ac0e-d0bbdb2796b1'

// Creating a symbolic name for an existing resource
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: storageAccountName
  scope: resourceGroup(storageSubscriptionId, storageResourceGroupName)
}

// Retrieving resource property
output blobEndpoint string = stg.properties.primaryEndpoints.blob
```


## Reference Existing Child Resource

In this section we will explore multiple ways how to reference an existing child resource in Bicep.

Let's illustrate this on an example of a Key Vault and a secret. The following examples assume that we have a Key Vault `kv-contoso` and a secret `someSecret` in it.

As a result, we want to return `secretUriWithVersion` in template deployment output.

### Child Resource Only

The most succinct way to reference a child resource is by specifying child's full type and including parent's name like shown in the code snippet below.

Just note that we as always use `existing` keyword and the name consists of **two sections** separated by slash `/`.

```tsx
param keyVaultName string = 'kv-contoso'
param secretName string = 'someSecret'

resource secret 'Microsoft.KeyVault/vaults/secrets@2019-09-01' existing = {
  name: '${keyVaultName}/${secretName}'
}

// https://kv-contoso.vault.azure.net/secrets/someSecret/2cdd92336f0a4a0a80bbbbdf9af8407d
output secretUri string = secret.properties.secretUriWithVersion
```

### Parent and Child Resource At Root Level

This approach leverages `parent` property which can be passed when declaring symbolic name for the child resource.

It might come in handy when we want to retrieve some properties both from parent and child resources like shown in the example below.

```tsx
resource keyvault 'Microsoft.KeyVault/vaults@2019-09-01' existing = {
  name: 'kv-contoso'
}

resource secret 'Microsoft.KeyVault/vaults/secrets@2019-09-01' existing = {
  name: 'someSecret'
  parent: keyvault
}

// https://kv-contoso.vault.azure.net/
output vaultUri string = keyvault.properties.vaultUri

// https://kv-contoso.vault.azure.net/secrets/someSecret/2cdd92336f0a4a0a80bbbbdf9af8407d
output secretUri string = secret.properties.secretUriWithVersion
```

### Parent and Child Resource Nested

Here is a slight variation of the previous case which leverages Bicep's feature of [declaring child resources inside of a parent](https://github.com/Azure/bicep/issues/127){:target="_blank"}.

Note that to access child resource symbolic name, we need to use `::` operator.

```tsx
resource keyvault 'Microsoft.KeyVault/vaults@2019-09-01' existing = {
  name: 'kv-contoso'

  resource secret 'secrets' existing = {
    name: 'someSecret'
  }
}

// https://kv-contoso.vault.azure.net/
output vaultUri string = keyvault.properties.vaultUri

// https://kv-contoso.vault.azure.net/secrets/someSecret/2cdd92336f0a4a0a80bbbbdf9af8407d
output secretUri string = keyvault::secret.properties.secretUriWithVersion
```


## Old Way Using reference() Function

Lastly, remember that any ARM template function is valid in Bicep, thus we still can use well-known `reference` function directly in Bicep code.

Moreover, under the hood Bicep just compiles all the examples above to the correct use of `reference` function.

I've written a post about [reference function](/blog/reference-function-arm-template) and there's a dedicated section to [referencing existing resources](/blog/reference-function-arm-template#examples-of-reference-function).

To illustrate this, take a look at the following example where reference function is directly used to retrieve needed property. Note that there are **no** `[]` around function invocation.

```tsx
param storageAccountName string = 'stcontoso'

output blobEndpoint string = reference(resourceId('Microsoft.Storage/storageAccounts', storageAccountName), '2021-02-01').primaryEndpoints.blob
```

The code above is equivalent to the bicep code we already discussed. I think there's no doubt that referencing resources through symbolic names is easier and handier.

```tsx
param storageAccountName string = 'stcontoso'

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: storageAccountName
}

output blobEndpoint string = stg.properties.primaryEndpoints.blob
```


## Related Posts
- [Parameters In Azure Bicep - Ultimate Guide With Examples](/blog/azure-bicep-parameters)
- [Reference() Function Explained With Examples - ARM Template](/blog/reference-function-arm-template)

## Useful Links
- [Bicep Spec: Resources](https://github.com/Azure/bicep/blob/main/docs/spec/resources.md){:target="_blank"}
- [Bicep Spec: Resource scopes](https://github.com/Azure/bicep/blob/main/docs/spec/resource-scopes.md){:target="_blank"}
- [Bicep Overview](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/bicep-overview){:target="_blank"}
