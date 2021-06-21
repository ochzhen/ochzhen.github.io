---
layout: post
title:  "Child Resources In Azure Bicep - 3 Ways To Declare, Loops, Conditions"
tags: azure-bicep
---

In this post we will discuss child resources in Bicep, these are the resources that exist only in the context of a parent resource. Also, sometimes they might be called nested resources.

In other words, a child resource cannot exist without its parent. **Nesting of resources** can be arbitrarily deep as long as it is supported by the schema.

Most likely, you've already encountered such resources in Azure. Let's take a look at some **common examples**:
- Virtual Machine or Scale Set extensions: `Microsoft.Compute/virtualMachines/extensions`,  `Microsoft.Compute/virtualMachineScaleSets/extensions`
- Key Vault secrets and access policies: `Microsoft.KeyVault/vaults/secrets`, `Microsoft.KeyVault/vaults/accessPolicies`
- Configuration for App Service or Azure Function: `Microsoft.Web/sites/config`
- Containers in Storage Account (note the deeper nesting): `Microsoft.Storage/storageAccounts/blobServices/containers`

So, as we can see, child resources are widely used in Azure, and that's why it is useful to know how to manage them using Bicep and Infrastructure As Code mindset.

**Contents:**
* TOC
{:toc}

## Overview

The first section [3 Ways To Declare Child Resources](#3-ways-to-declare-child-resources) covers variations on how to specify child resources in a Bicep file within/without and with/without parent resource definition.

If we choose to define resource within its parent, then [Accessing Nested Resource With :: Operator](#accessing-nested-resource-with--operator) comes in handy when we want to get some properties of the nested resource by its symbolic name.

Sometimes the decision whether to deploy a particular resource is based on some condition. This condition functionality has slightly different behavior depending on the way a child resource is declared, this is covered in [Conditional Deployment](#conditional-deployment) section.

[Loops: Creating Multiple Resources](#loops-creating-multiple-resources) describes how loops can be used in a combination with the approaches to declare child resource discussed in the beginning of the post.

[Example: Storage Account - Putting It All Together](#example-storage-account---putting-it-all-together) wraps up the post with an imaginary deployment of a storage account with containers where most of the concepts discussed throughout the post are applied.


## 3 Ways To Declare Child Resources

Obviously, when declaring a child resources, we need to provide information about its parent resource in some way.

Bicep provides multiple ways to declare child resources, all of them being quite handy and easy to use.

### Within Parent Resource

One of the ways to define child resources is to declare them inside of a parent resource as shown in the code snippet below.

**NOTES:**
- `resource` keyword (line **10**) is used as always for a resource declaration, multiple child resources can be declared in this way
- Child resource type can optionally have a version specified if it is different from the parent version (on line **10**)
- `name`('admin-password') and `type`('secrets') have only one segment because the resource is defined within a parent
- Password value is passed through a `secureString` (lines **2-3**) - just a good practice for working with secrets
- In generated ARM template the child resource will have dependency on the parent
- Child resource cannot be declared within a parent resource if the parent is declared with a loop (`for` expression), see [this section](#multiple-parent--one-child-resource-per-parent)

As a general rule, a child resource declared within its parent can access parent resource properties and siblings. At the same time, parent resource cannot access its child resources because it would lead to cyclic dependencies.

{% highlight tsx linenos %}
param keyVaultName string = 'kv-contoso'
@secure()
param adminPassword string

resource kv 'Microsoft.KeyVault/vaults@2019-09-01' = {
  name: keyVaultName
  // Other Key Vault properties are ommitted

  // Type could also contain version, e.g. 'secrets@2018-02-14'
  resource adminPwd 'secrets' = {
    name: 'admin-password'
    properties: {
      value: adminPassword
    }
  }

  // Additional child resources
  // resource userPwd 'secrets' = {
  //   name: 'user-password'
  //   ...
  // }
}
{% endhighlight %}

### Outside Parent With "parent" Property

Sometimes we want to be able to declare child resources separately, not within a parent resource. 

For this case Bicep has `parent` property where symbolic name of the parent resource can be passed. Basically, this allows Bicep to automatically infer name of the parent without us specifying multiple segments.

**NOTES:**

- Child resource type must be a full type with version (line **10**)
- `parent` property is set to the key vault in which context this secret exists (line **13**)
- `name` property contains only one segment because `parent` property is present (line **12**)
- During compilation Bicep adds a dependency on the parent in child's `dependsOn` section

{% highlight tsx linenos %}
param keyVaultName string = 'kv-contoso'
@secure()
param adminPassword string

resource kv 'Microsoft.KeyVault/vaults@2019-09-01' = {
  name: keyVaultName
  // Other Key Vault properties are ommitted
}

// Full type with version
resource adminPwd 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = {
  name: 'admin-password'   // Only one segment
  parent: kv               // Link to parent resource
  properties: {
    value: adminPassword
  }
}
{% endhighlight %}

### Outside Parent Without "parent" Property

Last but not least, a child resource can be declared in a standalone manner like it can be done in an ARM template.

**NOTES:**

- No need for a symbolic name of the parent resource
- Resource type is a full type with version (line **6**)
- `name` consists of two segments, parent resource name must be included (line **7**)
- The example below assumes that the parent Key Vault already exists

{% highlight tsx linenos %}
param keyVaultName string = 'kv-contoso'
@secure()
param adminPassword string

// Full type with version
resource adminPwd 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = {
  name: '${keyVaultName}/admin-password'  // Two segments
  properties: {
    value: adminPassword
  }
}
{% endhighlight %}


## Accessing Nested Resource With :: Operator

Operator `::` allows accessing a child resource declared within its parent resource by using symbolic names of the parent and child.

The problem is that the child resource symbolic name cannot be used directly in the rest of the template because this symbolic name only exists within the scope of a parent resource.

Let's take a look at an example of Key Vault and a secret, note how `::` operator is used to get `id` of the secret using symbolic names.

{% highlight tsx linenos %}
// Referencing existing Key Vault
resource kv 'Microsoft.KeyVault/vaults@2019-09-01' existing = {
  name: 'kv-contoso'

  resource adminPwd 'secrets' = {
    name: 'admin-password'
    properties: {
      value: 'somesecretvalue'
    }
  }
}

// Operator :: is used to access child resource
output secretId string = kv::adminPwd.id
{% endhighlight %}


## Conditional Deployment

Conditional deployment allows deciding whether a resource should be deployed based on a **condition evaluated at the start of the deployment**.

For example, we can introduce a parameter which controls whether our resource will be deployed, or we can compute the condition based on the state of other resources.

Just keep in mind a couple of things when using conditional deployment for a child resource or its parent:

- If a child resource is declared **within** the parent resource, and the **parent has condition**, then this condition **will be applied to the child** as well.
- If a child resource is declared **outside** the parent resource, and the **parent has condition**, then this condition **won't be applied to the child**.

So, the second use case requires attention because if a condition is only applied to the parent but not child, then a runtime error may occur since we try to deploy the child resource while the parent doesn't exist.

As always, let's illustrate the text with some code for better understanding.

{% highlight tsx linenos %}
param keyVaultName string = 'kv-contoso'
param shouldDeploy bool = true

resource kv 'Microsoft.KeyVault/vaults@2019-09-01' = if (shouldDeploy) {
  name: keyVaultName
  // Other Key Vault properties are ommitted

  // Within Parent Resource
  // Condition "shouldDeploy" will be applied
  resource secret1 'secrets' = {
    name: 'secret1'
    properties: {
      value: 'somesecretvalue1'
    }
  }
}

// Outside Parent With "parent" Property
// Condition "shouldDeploy" won't be applied
resource secret2 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = {
  name: 'secret2'
  parent: kv
  properties: {
    value: 'somesecretvalue2'
  }
}

// Outside Parent Without "parent" Property
// Condition "shouldDeploy" won't be applied
resource secret3 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = {
  name: '${keyVaultName}/secret3'
  properties: {
    value: 'somesecretvalue3'
  }
}
{% endhighlight %}


## Loops: Creating Multiple Resources

Loops are a very powerful functionality which allows creating multiple instances of a resource. We will cover different combinations of loops with parent and child resources.

### Single Parent & Multiple Child Resources

The first use case will be simple, assume there is **one Key Vault**, and we want to create **multiple secrets** in it.

**NOTE:** Any of the [3 Ways To Declare Child Resources](#3-ways-to-declare-child-resources) can be used, the following example declares child resources within its parent.

{% highlight tsx linenos %}
// Creating 1 Key Vault (parent)
resource kv 'Microsoft.KeyVault/vaults@2019-09-01' = {
  name: 'kv-contoso'
  // Other Key Vault properties are ommitted

  // Creating 3 secrets (children)
  resource secrets 'secrets' = [for i in range(0,3): {
    name: 'secret${i}'
    properties: {
      value: 'somesecretvalue${i}'
    }
  }]
}
{% endhighlight %}

### Multiple Parent & One Child Resource Per Parent

Slightly more complicated case is when we want to create **multiple Key Vaults**, each having **one secret**.

**NOTE:** Nested resource cannot appear inside of `for` expression. In other words, when parent resource is declared using a loop, then a child resource cannot be declared within parent.

To create vaults and secrets with 1 to 1 relationship, child resources should be declared **outside parent** like in the example below. Note that both ways with and without `parent` property would work.

Another even **better option is to use modules**, they will be covered in the [next section](#multiple-parent--multiple-child-resources-per-parent).

{% highlight tsx linenos %}
param keyVaultCount int = 2    // Number of Key Vaults to create

resource kvs 'Microsoft.KeyVault/vaults@2019-09-01' = [for i in range(0, keyVaultCount): {
  name: 'kv-contoso-${i}'
  // Other Key Vault properties are ommitted
}]

resource secrets 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = [for i in range(0, keyVaultCount): {
  name: 'secret${i}'
  parent: kvs[i]    // kvs is an array of resources, getting i-th element
  properties: {
    value: 'somesecretvalue${i}'
  }
}]
{% endhighlight %}

### Multiple Parent & Multiple Child Resources Per Parent

Let's say we want to have **multiple Key Vaults**, each having **multiple secrets**. This is not easily achievable with the approach we used above - nested loops are not supported.

This is where **modules** come in very handy, they allow writing elegant and easy to understand Bicep code.

In the resulting ARM template module will compile into nested deployments where each nested deployment deploys one vault and multiple secrets.

{% highlight tsx linenos %}
// ========== main.bicep file ==========

// Creating 2 Key Vaults using module
module kvs './keyvault.bicep' = [for i in range(0,2): {
  name: 'keyvault${i}'
  params: {
    keyVaultName: 'kv-contoso-${i}'
  }
}]
{% endhighlight %}

{% highlight tsx linenos %}
// ========== ./keyvault.bicep file ==========

param keyVaultName string

resource kv 'Microsoft.KeyVault/vaults@2019-09-01' = {
  name: keyVaultName
  // Other Key Vault properties are ommitted

  // Creating 3 secrets in Key Vault
  resource secrets 'secrets' = [for i in range(0,3): {
    name: 'secret${i}'
    properties: {
      value: 'somesecretvalue${i}'
    }
  }]
}
{% endhighlight %}


## Example: Storage Account - Putting It All Together

In the previous sections, a lot of things were covered. In all preceding examples we used Key Vault and secrets to illustrate parent-child relationship between resources.

Now, let's take a **storage account with containers** and apply the following things in one example:

- Declaration of child resources [within a parent resource](#within-parent-resource)
- Deeply nested child resources: `storageAccounts` > `blobServices` > `containers`
- [Conditional deployment](#conditional-deployment) of containers
- [One storage account with multiple containers](#single-parent--multiple-child-resources): `logs`, `data`, `backup`
- Outputting value of a child resource using [nested resource accessor](#accessing-nested-resource-with--operator) `::` and [ternary operators](https://github.com/Azure/bicep/blob/main/docs/spec/expressions.md#ternary-operator){:target="_blank"}

The code below illustrates all of the points mentioned above, also see inline comments.

{% highlight tsx linenos %}
param storageAccountName string = 'stcontoso'
param shouldCreateContainers bool = true
param containerNames array = [
  'logs'
  'data'
  'backup'
]

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  kind: 'StorageV2'
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }

  // Containers live inside of a blob service
  resource blobSvc 'blobServices' = {
    name: 'default'   // Always has value 'default'

    // Creating containers with provided names if contition is true
    resource containers 'containers' = [for name in containerNames: if(shouldCreateContainers) {
      name: name
      properties: {
        publicAccess: 'Blob'    // Just setting some property
      }
    }]
  }
}

// Checking if containers were created using ternary operator
// If yes then returning publicAccess property of the first container, else empty string
output containerPublicAccess string = shouldCreateContainers ? stg::blobSvc::containers[0].properties.publicAccess : ''
{% endhighlight %}


## Related Posts
- [Parameters In Azure Bicep - Ultimate Guide With Examples](/blog/azure-bicep-parameters)
- [Variables In Azure Bicep - From Basics To Advanced](/blog/azure-bicep-variables)
- [Reference New Or Existing Resource In Azure Bicep](/blog/reference-new-or-existing-resource-in-azure-bicep)
- [Reference() Function Explained With Examples - ARM Template](/blog/reference-function-arm-template)


## Useful Links
- [bicep/resources.md - Azure Bicep](https://github.com/Azure/bicep/blob/main/docs/spec/resources.md){:target="_blank"}
- [bicep/loops.md - Azure Bicep](https://github.com/Azure/bicep/blob/main/docs/spec/loops.md){:target="_blank"}
- [Child resources in templates - Azure Resource Manager](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/child-resource-name-type?tabs=bicep)