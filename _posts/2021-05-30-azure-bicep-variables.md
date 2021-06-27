---
layout: post
title:  "Variables In Azure Bicep - From Basics To Advanced"
tags: azure-bicep
---


In this post I wanted to gather useful information about Bicep variables and illustrate it with simple-to-understand examples. Let's go!

**Variables in Azure Bicep allow specifying complex expressions that can be used in other parts of the code without duplication. In contrast to ARM templates, `reference` and `list*` functions can be used in Bicep variables.**

Luckily, variables in Bicep are a quite simple and intuitive concept. However, let's explore some examples and see how Bicep compiles them into ARM templates.

**Contents:**
* TOC
{:toc}


## Overview

We begin with a short discussion on when it is a good idea to use variables: [When To Use Variables?](#when-to-use-variables).

The next section [Declaration And Initialization](#declaration-and-initialization) includes explanation of the syntax and what elements of a bicep file can be used to *define a variable*: parameters, variables, resources, modules, functions, loops.

We will also briefly touch on existing [constraints and limitations](#constraints--limitations) that one should keep in mind while using variables in Bicep, followed by a short section about [how to use a variable](#using-variable-in-template).

Section [How Bicep Variables Work](#how-bicep-variables-work) explains differences between Bicep and ARM template variables and illustrates with code samples.

The last section is about [using `reference` and `list*` functions in a variable](#using-reference-and-list-functions-in-a-variable) - functionality not available in ARM templates but present in Bicep.


## When To Use Variables?

First of all, let's briefly discuss when it makes sense to use variables. Here are some thoughts when bicep variables may be useful:

- Reuse the same value in multiple places in the file
- Be able to easily change value used in multiple places
- Declare a complex expression in a variable to improve readability

Next, let's talk about how to declare and use variables in a template.


## Declaration And Initialization

Declaring a variable and specifying its value is extremely simple in Bicep. Intuitive syntax and ability to leverage functions make variables a very powerful and useful tool.

### Syntax

Quite simple and similar to many other languages, just some notes in addition to the code sample below:

- No need to specify type - it is inferred automatically from the assigned value
- On the right side of the assignment can be any expression
- Value can be `null` as well

```tsx
// var <name> = <expression>
var myString = 'some string value'
var myNull = null
var location = resourceGroup().location
```

### Declare Anywhere In File

In ARM templates we had to specify all variables in a specific section of a template.

In contrast to ARM templates, variables in Bicep can be declared anywhere in a file, even after they are referenced.

The reason for this behavior is simple: Bicep file is declarative and, as a result, during compilation **Bicep will bring variables together into the corresponding section or embed expression inline**.

To illustrate this, consider this imaginary example and `resourceLocation` variable:

```tsx
param storageAccountName string

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  kind: 'StorageV2'
  location: resourceLocation
  sku: {
    name: 'Standard_LRS'
  }
}

// Even though it is possible to declare variable anywhere
// Still worth declaring it before the place it is used for better readability
var resourceLocation = resourceGroup().location
```

The compiled ARM template will have our variable `resourceLocation` in the `variables` section:

```json
{
  "...": "...",
  "parameters": {
    "storageAccountName": {
      "type": "string"
    }
  },
  "variables": {
    "resourceLocation": "[resourceGroup().location]"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-02-01",
      "name": "[parameters('storageAccountName')]",
      "kind": "StorageV2",
      "location": "[variables('resourceLocation')]",
      "sku": {
        "name": "Standard_LRS"
      }
    }
  ]
}
```

### Use Parameters, Resources, Modules, or Another Variables

Variables are so powerful because it is possible to combine different Bicep elements during a variable creation.

#### Parameters

A quite common case is using parameters in variable declaration.

```tsx
param name string
var greeting = 'Hello, ${name}!'  // string interpolation
```

#### Variables

Other variables can also be used on the right side of the declaration. The only note here is that we cannot introduce cycles in variables usage which is an obvious constraint.

```tsx
var name = 'John'
var greeting = 'Hello, ${name}!'
```

#### Resources

In addition to parameters and variables, one can use resource properties to create a variable. Please find an example below.

**NOTE:** Read more about `existing` keyword in [Reference New Or Existing Resource In Azure Bicep](/blog/reference-new-or-existing-resource-in-azure-bicep#keyword-existing-and-scope-property)

```tsx
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: 'stcontoso'
}

var myTag = '${stg.type}-${stg.kind}-${stg.sku.name}'
```

#### Modules

If a module returns some values in `outputs` section, they can also be used in variable declaration.

In the example below module returns full storage account object in outputs which we then use in the main template.

```tsx
// ===== main.bicep =====

module stg './storage.bicep' = {
  name: 'storageDeployment'
  params: {
    storageAccountName: 'stcontoso'
  }
}

// Using module outputs to create a variable
var myTag = '${stg.outputs.storageAccount.kind}-${stg.outputs.storageAccount.sku.name}'

// ===== ./storage.bicep =====

param storageAccountName string

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  location: resourceGroup().location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
}

output storageAccount object = stg
```

### Use Functions

Any [ARM template function](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions){:target="_blank"} is **valid in Bicep**, and they can also be used in variables if needed.

We can combine multiple functions, parameters, resources, etc. to come up with a value for a variable. Luckily, Bicep will handle the complexity and make authoring easy for us.

To illustrate it, consider the following imaginary example of creating `myTag` variable.

```tsx
param prefix string
var location = resourceGroup().location

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: 'stcontoso'
}

var myTag = '${prefix}-${location}-${stg.sku}'
```

### Use Loops

Another useful feature is the ability to use looping to initialize a variable. It might come in handy when we want to **extract complexity** out of resource declaration.

In the example below, `range(0, 3)` is used but any other enumerable expression can be there, for example, parameter of type `array`.

```tsx
// Creating a variable using a for-loop
var secretsValues = [for i in range(0, 3): {
  name: 'secret${i}'
  value: 'supersecretvalue${i}'
}]

// Assuming that a key vault already exists
resource kv 'Microsoft.KeyVault/vaults@2019-09-01' existing = {
  name: 'kv-contoso'
}

// Using variable to create multiple resources
resource secrets 'Microsoft.KeyVault/vaults/secrets@2019-09-01' = [for secret in secretsValues: {
  name: secret.name
  parent: kv
  properties: {
    value: secret.value
  }
}]
```


## Constraints & Limitations

We've already discussed how useful variables are, however, it is worth reviewing some limitations that come with the use of variables.

- Name must be unique among parameters, resources, modules, and other variables, but it can be the same as output name
- Cannot reassign another value to a variable (assign only once)
- Declaration and initialization can't be separated


## Using Variable In Template

Including this small section only for completeness since using variable value is extremely simple - just reference it by name like it is done in many other programming languages.

```tsx
var storageAccountName = 'mysuperstorage'

// ...

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  // ...
}
```


## How Bicep Variables Work

Interestingly, variables in Bicep behave slightly different from variables in ARM template due to the fact that there's one more level of abstraction.

There are two ways how Bicep variable is handled in the final ARM template:

- Variable is placed into [variables section](#variables-section) - default behavior, works with Bicep "static" variables which use parameters, other variables or constants in the declaration
- Value is [embedded inline](#inline-embedding) where variable is used - this is applied to "dynamic" variables which use resource or module properties in the declaration

As a result of the second behavior, the difference between ARM template and Bicep variables is that Bicep provides **more flexibility** and solves the problem of [using `reference` and `list*` functions in a variable](#using-reference-and-list-functions-in-a-variable).

Next, let's take a look at the examples of the two cases described above.

### Variables Section

If variable declaration is simple and uses static information, then it can be compiled into a corresponding entry inside of `variables` section.

In the following example we have a `storageAccountName` variable which is declared using a parameter.

```tsx
param projectName string
var storageAccountName = 'st${projectName}'

resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  location: resourceGroup().location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
}
```

As a result, ARM template has this variable in the corresponding section as shown below.

```json
{
  "...": "...",
  "parameters": {
    "projectName": {
      "type": "string"
    }
  },
  "variables": {
    "storageAccountName": "[format('st{0}', parameters('projectName'))]"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-02-01",
      "name": "[variables('storageAccountName')]",
      "location": "[resourceGroup().location]",
      "kind": "StorageV2",
      "sku": {
        "name": "Standard_LRS"
      }
    }
  ]
}
```

### Inline Embedding

If variable declaration uses **dynamic state** of the resources, then it cannot be placed into the variables section. In this case, Bicep takes variable expression and puts it in all occurrences, as simple as that.

Here is an imaginary example where the same variable is used twice in outputs.

```tsx
resource stg 'Microsoft.Storage/storageAccounts@2021-02-01' existing = {
  name: 'stcontoso'
}

var storageKind = 'tag-${stg.kind}'

output output1 string = storageKind
output output2 string = storageKind
```

Note that there was **no variable created** in `variables` section since Bicep variable uses the runtime of a resource.

```json
{
  "...": "...",
  "resources": [],
  "outputs": {
    "output1": {
      "type": "string",
      "value": "[format('tag-{0}', reference(resourceId('Microsoft.Storage/storageAccounts', 'stcontoso'), '2021-02-01', 'full').kind)]"
    },
    "output2": {
      "type": "string",
      "value": "[format('tag-{0}', reference(resourceId('Microsoft.Storage/storageAccounts', 'stcontoso'), '2021-02-01', 'full').kind)]"
    }
  }
}
```


## Using reference and list* functions in a variable

Now equipped with the knowledge of how Bicep variables work, we can discuss `reference` and `list*` functions.

**In Bicep variables `reference` and `list*` functions can be used in contrast to ARM templates where they cannot.**

Why is it possible? Because during compilation into ARM template, [Bicep embeds the expressions inline](#inline-embedding) without using `variables` section.

Let's illustrate this behavior using `listKeys` function for storage account. For example, it might be useful when we want to create a connection string.

**NOTE:** Read more about referencing resources in Bicep in [Reference New Or Existing Resource In Azure Bicep](/blog/reference-new-or-existing-resource-in-azure-bicep).

```tsx
param storageAccountName string = 'stcontoso'

// This Bicep variable is not compiled into an ARM template variable,
// but instead expression is inserted in every place where it's used.
var keysObj = listKeys(resourceId('Microsoft.Storage/storageAccounts', storageAccountName), '2021-02-01')

output key1 string = keysObj.keys[0].value
output key2 string = keysObj.keys[1].value
```

The Bicep code above is compiled into the following ARM template. Note that `listKeys` function is used twice, and there is no variable in the template.

```json
{
  "...": "...",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "defaultValue": "stcontoso"
    }
  },
  "outputs": {
    "key1": {
      "type": "string",
      "value": "[listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName')), '2021-02-01').keys[0].value]"
    },
    "key2": {
      "type": "string",
      "value": "[listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName')), '2021-02-01').keys[1].value]"
    }
  }
}
```


## Related Posts

- [Parameters In Azure Bicep - Ultimate Guide With Examples](/blog/azure-bicep-parameters)
- [Reference New Or Existing Resource In Azure Bicep](/blog/reference-new-or-existing-resource-in-azure-bicep)
- [Child Resources In Azure Bicep - 3 Ways To Declare, Loops, Conditions](/blog/child-resources-in-azure-bicep)
- [Create Resource Group With Azure Bicep and Deploy Resources In It](/blog/create-resource-group-azure-bicep)
- [Reference() Function Explained With Examples - ARM Template](/blog/reference-function-arm-template)

## Useful Links

- [Variables in ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-variables?tabs=bicep){:target="_blank"}
- [Variables - Azure Bicep Spec](https://github.com/Azure/bicep/blob/main/docs/spec/variables.md){:target="_blank"}
- [ARM Template limits](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-best-practices#template-limits){:target="_blank"}
