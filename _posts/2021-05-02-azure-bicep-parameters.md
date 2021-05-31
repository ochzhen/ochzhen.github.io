---
layout: post
title:  "Parameters In Azure Bicep - Ultimate Guide With Examples"
tags: azure-bicep
---

In this post I would like to gather useful information that will help easily get started with parameters in Bicep language.

**Parameters in Azure Bicep are used to pass dynamic values into a template to make it flexible and reusable. Parameter declaration may contain default value and/or constraints on input value.**

Bicep is a domain-specific language which transpiles into ARM templates. In other words, it's a more convenient way to declare resources, and it works on top of ARM templates. Overview of bicep language is beyond the scope of this post.

You can read the post from start to finish or use navigation to jump to the section that interests you most.

**Contents:**
* TOC
{:toc}


## Should I Use Parameters?

Let's first discuss a simple but important topic when we actually need parameters, it may turn out that one doesn't need them at all :-).

Let's briefly cover when it makes sense to parametrize our template/resource file in case of bicep.

- **Don't** create parameters for values that can be computed based on other parameters - better avoid unnecessary parameters since it adds complexity we need to manage.
- **Do** have parameters when values can change between template usages, for example, when using the same template for different environments - e.g. development, test, production.
- **Do** use parameters to pass secrets - never store secrets as plain text in source code, better reference secrets from such services as Key Vault.

To sum up, use parameters to gain more flexibility but don't overuse it :D.


## Declaring Parameters

Every parameter in Bicep must have type specified, at this moment there are 5 types available to use in parameter declaration:

- `string`
- `object`
- `array`
- `int`
- `bool`

There are also plans to add `number` type to support floating point numbers. Read more about [types in Bicep](https://github.com/Azure/bicep/blob/main/docs/spec/types.md){:target="_blank"}.

### Basic Case

It is extremely simple to declare a parameter, the code snippet below shows how to do that for a simple scenario.

```tsx
// param myParam1 <type>
param myParam1 string

// param myParam2 <type> = <default_value>
param myParam2 string = 'foobar'
```

### Advanced Case

As a more advanced case I'd consider the **use of decorators** to enforce different constraints on input values.

In the last sections we will go over decorators for each data type and use case just to see when and how to use them with parameters.

However, I'd like to separately mention the option to provide a description for a parameter in two ways: using `@description('...')` or `@metadata({ description: '...' })` decorators (note that newlines are omitted in the latter case).

```tsx
// @decorator1()
// @decorator2()
// ...
// param myParam <type> = <default_value>

@minValue(3)
@maxValue(9)
@description('Number of virtual machines')
// OR:
// @metadata({
//   description: 'Number of virtual machines'
// })
param virtualMachineCount int = 3
```

The bicep code above transpiles into the following section of an ARM template:

```json
{
  "...": "...",
  "parameters": {
    "virtualMachineCount": {
      "type": "int",
      "defaultValue": 3,
      "metadata": {
        "description": "Number of virtual machines"
      },
      "maxValue": 9,
      "minValue": 3
    }
  },
  "...": "..."
}
```

### Constraints

There aren't many constraints on parameters but we should nevertheless know them:

- Name must be **unique** among other *parameters*, *variables*, and *resources*, but can be the same as *outputs*.
- Name is **case-insensitive** - even though Bicep allows creating two parameters whose names only differ in case, it will fail during template deployment telling that item with the same key has already been added.
- Maximum **256** parameters per file - it's a very generous limit, if you start getting many parameters then you should probably modularize your template.

### Where To Declare Parameters

In Bicep we **can declare parameters in any place**, it doesn't actually matter. Moreover, we can even declare parameter after the place where we use it.

The reason is quite simple - during compilation from bicep to ARM template, all parameters will be gathered in the parameters section of the ARM template.

As a result, we are free to declare them anywhere we want in bicep file.

```tsx
resource storageAccount 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  ...
}

param storageAccountName string
```

Resulting ARM template will looks like the following (metadata section is omitted):

```json
{
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
      "...": "..."
    }
  ]
}
```

## Using Declared Parameters In File

Using parameter values inside the code is very intuitive and doesn't need much explanation. Just refer to the parameter by its name and use dot `.` to drill into nested properties in case of objects.

For the sake of completeness, here is an example that illustrates the use of string and object parameters:

```tsx
param storageAccountName string = 'stcontoso'
param storageAccountSettings object = {
  location: resourceGroup().location
  kind: 'StorageV2'
  sku: 'Standard_LRS'
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-02-01' = {
  name: storageAccountName
  location: storageAccountSettings.location
  kind: storageAccountSettings.kind
  sku: {
    name: storageAccountSettings.sku
  }
}
```


## Default Value

Sometimes we want to have a default value that will be used in case user decided not to specify any. This can make user's life easier in many cases.

When parameter has a default value, then this parameter becomes not required during template deployment.

In the following subsections we'll cover the ways of specifying a default value.

### Literal Value

This is the most simple case - we can provide some constant value of the corresponding type as default.

```tsx
param myString string = 'foobar'
param myInt int = 28
param myBool bool = true
param myArr array = [
  'one'
  'two'
  'three'
]
param myObj object = {
  name: 'John'
  age: 42
}
```

### Another Parameter

In some cases we might want to **reuse** one parameter value to compose the default value for another one.

In the following example resource names are derived from the prefix parameter but at the same time user can override these defaults.

```tsx
param namePrefix string = ''
param storageAccountName string = '${namePrefix}staccount1'
param vmssName string = '${namePrefix}vmss1'
```

### Function/Expression

Additionally, different built-in functions can be used in default value composition. More generally, **it could be almost any expression** that evaluates to a value of the specified type.

One very common example is resource group location. The default location for resources is the same as the location of the resource group, however, user can override this since resources don't necessarily have to be in the same location as resource group.

Note that in Bicep expressions are *not enclosed* into square brackets `[...]` in contrast to ARM templates.

```tsx
param location string = resourceGroup().location
```

**NOTE:** All ARM template functions are valid in Bicep, so you might want to take a look at the [list of all ARM template functions](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions){:target="_blank"} available.

**IMPORTANT:** `reference` and `list*` functions cannot be used to specify default value, this is due to the fact that these functions retrieve the runtime state of a resource which not always makes sense in parameters section.


## Passing Parameters

Now given that our Bicep template has parameters, how do we actually pass them?

In this regard there's nothing new compared to ARM templates, we can either pass them **inline** or use a **parameter file**.

Small mention is that passing parameters that are not declared in the template results in a failing deployment due to a validation error.

**NOTE:** When passing value either through parameter file or inline, name is case-insensitive but it is still better to match the casing for consistency.

### Inline

During template deployment parameters can be passed inline, here's an example for Azure PowerShell:

```powershell
New-AzResourceGroupDeployment `
  -Name myBicepTemplateDeployment `
  -ResourceGroupName rg-contoso `
  -TemplateFile ./main.bicep `
  -storageAccountName stcontosobackup
```

### Parameter File

In Bicep we also can create parameter files like in plain old ARM templates, moreover, Bicep just uses ARM template parameter files and doesn't add anything new.

So, parameter files can be used in the same way, for example, we could have `main.bicep` and `main.parameters.json` files.

`main.bicep`:

```tsx
@minLength(3)
@maxLength(24)
param storageAccountName string
```

`main.parameters.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "stcontoso"
    }
  }
}
```

And then submit a deployment using PowerShell, for example:

```powershell
New-AzResourceGroupDeployment `
  -Name myBicepTemplateDeployment `
  -ResourceGroupName rg-contoso `
  -TemplateFile ./main.bicep `
  -TemplateParameterFile ./main.parameters.json
```


## Decorators: Constraints And Metadata

In Bicep decorators are used to specify constraints and metadata for a parameter. Here's a list of decorators in Bicep:

- `allowed`
- `secure`
- `minLength` & `maxLength`
- `minValue` & `maxValue`
- `description`
- `metadata`

In this section we will briefly go over the syntax and `description` & `metadata` decorators because they can be applied to any parameter independent of type.

The remaining sections will cover how to work with parameters of specific types and which decorators can or should be applied in those cases.

### Syntax

Multiple decorators can be applied to one parameter, the following code shows syntax as well as an example how to use them.

```tsx
// Syntax
// @expression1             // expression must be a function call
// @expression2
// ...
// param myParam <type>

// Example
@secure()
@minLength(8)
@maxLength(24)
param adminPassword string
```


### Metadata Decorator

Metadata object can be added to attach some information to a parameter. Resource Manager ignores it during deployment, so this metadata is mainly for humans.

We can specify any object we want but the convention is to have description property in it.

```tsx
@metadata({
  description: 'The number of virtual machines that will be created'
  myCustomField: 'Something important here'
})
param virtualMachineCount int
```

The Bicep code above gets transpiled into the following ARM template:

```json
{
  "...": "...",
  "parameters": {
    "virtualMachineCount": {
      "type": "int",
      "metadata": {
        "description": "The number of virtual machines that will be created",
        "myCustomField": "Something important here"
      }
    }
  },
  "...": "..."
}
```

### Description Decorator

Description decorator is a less verbose way to specify metadata object with description. If we only want to specify description, I'd rather use `description` decorator instead of `metadata`.

```tsx
@description('The number of virtual machines that will be created')
param virtualMachineCount int
```

The corresponding ARM template will look like:

```json
{
  "...": "...",
  "parameters": {
    "virtualMachineCount": {
      "type": "int",
      "metadata": {
        "description": "The number of virtual machines that will be created"
      }
    }
  },
  "...": "..."
}
```


## String Parameters

String parameters are probably the most used among all types. Here we discuss single and multi-line strings, **secure strings** will be covered in a separate blog post.

**NOTE:** Decorators [description](#description-decorator) and [metadata](#metadata-decorator) can be applied to any parameter including `string` parameters.

### String Interpolation

[String interpolation](https://en.wikipedia.org/wiki/String_interpolation){:target="_blank"} is a convenient and succinct way of combining literals with dynamic values.

In a single line string an expression should be put between curly braces of `${}` like this: `'${expression}'`.

In ARM templates we had to use [format](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-string?tabs=json#format){:target="_blank"} or [concat](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-string?tabs=json#concat){:target="_blank"} functions to achieve similar results, but string interpolation results in more readable code.


The following example illustrates the use of string interpolation and what it gets compiled into.

```tsx
param resourceTag string = '${resourceGroup().name}-${resourceGroup().location}-resource'
```

Compiled ARM template:

```json
{
  "...": "...",
  "parameters": {
    "resourceTag": {
      "type": "string",
      "defaultValue": "[format('{0}-{1}-resource', resourceGroup().name, resourceGroup().location)]"
    }
  },
  "...": "..."
}
```

### Multi line strings

In Bicep we can specify strings that span multiple lines by specifying our multiline string between three single quotes `'''string is here'''`.

This functionality allows writing more readable code compared to inserting `\n` in a single line string.

Here are a few points worth mentioning:

- If there's a newline right after opening `'''`, then it is ignored, all other newlines are processed
- The resulting value in ARM template is a single line string with `\n` instead of newlines
- Multiline strings do not support [string interpolation](#string-interpolation) (`'Hello, ${name}!'`) at this moment

Consider the following example:

```tsx
param myParam string = '''
First line
Second line
Third line'''
```

The resulting ARM template will look like this:

```json
{
  "...": "...",
  "parameters": {
    "myParam": {
      "type": "string",
      "defaultValue": "First line\nSecond line\nThird line"
    }
  },
  "...": "..."
}
```

### Controlling Length

Decorators `minLength` and `maxLength` allow constraining the minimum and maximum length of the parameter value respectively.

Use only one or both of them at the same time. Here's an example:

```tsx
@minLength(3)
@maxLength(24)
param storageAccountName string
```

### Providing Allowed Values

Use `allowed` decorator to limit the set of accepted values. For example, it may be useful when you want to allow only specific SKUs for a resource. However, don't overuse this constraint because it sometimes can limit the flexibility.

The `allowed` decorator accepts an array with *at least one element*. Also, at this moment array should be declared on multiple strings like the example below:

```tsx
@allowed([
  'Standard_ZRS'
  'Standard_GRS'
  'Standard_RAGRS'
])
param skuName string
```


##  Integer Parameters

Parameter of type `int` accepts integer values, so it could be both positive, negative numbers and zero. It is a 32-bit integer.

There is less to discuss about integer parameters compared to strings. But let's cover `minValue`, `maxValue`, and `allowed` decorators since they are applicable to integer parameters.

**NOTE:** Decorators [description](#description-decorator) and [metadata](#metadata-decorator) can be applied to any parameter including `int` parameters.

### Minimum & Maximum Value

Imagine that we want to limit the range of possible values that represent the number of virtual machines to create in a scale set.

This can be easily achieved using `minValue` and `maxValue` constraints.

```tsx
@minValue(3)
@maxValue(9)
param virtualMachineCount int = 5
```

### Specifying Allowed Values

Considering the previous example let's imagine that we want to allow creating only odd number of VMs, here's where `allowed` constraint comes into play.

```tsx
// At this moment array must be declared on multiple strings
@allowed([
  3
  5
  7
  9
])
param virtualMachineCount int
```


##  Boolean Parameters

Boolean parameters are extremely straightforward, there are only two possible values: true or false.

Decorator `@allowed([...])` could be applied to a bool parameter, however, it is not that useful as with strings or numbers since there are only two possible values.

**NOTE:** Decorators [description](#description-decorator) and [metadata](#metadata-decorator) can be applied to any parameter including `bool` parameters.

In the following example we set *isLargeScaleSet* to *true* if the number of VMs in a scale set greater than one hundred.

```tsx
param virtualMachineCount int
param isLargeScaleSet bool = virtualMachineCount > 100
```


##  Array Parameters

At the moment of writing, Bicep doesn't support single-line arrays as mentioned in [Known limitations](https://github.com/Azure/bicep#known-limitations){:target="_blank"}.

As a result, Bicep arrays must be declared on multiple lines, default value example:

```tsx
param myArray array = [
  1
  2
  3
]
```

**NOTE:** Decorators [description](#description-decorator) and [metadata](#metadata-decorator) can be applied to any parameter including `array` parameters.

### Min & Max Length

Decorators `minLength` and `maxLength` allow limiting the length of an incoming array value.

For example, a template can accept an array of storage account names and create/update resources with the corresponding names. If needed, we could limit the number of resource names that can be passed.

```tsx
@minLength(1)
@maxLength(5)
param storageAccountNames array
```

### Allowed Values

Specifying allowed values for an array parameter is a little bit trickier in my opinion.

I say that because intellisense doesn't enforce creating array of arrays while authoring template, but if we don't do that, we'll get an error when trying to deploy our Bicep file.

Also, it might get a bit verbose due to the limitation that elements of the array must be on different lines. However, this limitation should be removed in future versions of Bicep.

```tsx
@minLength(1)
@maxLength(5)
@allowed([
  [
    1
    2
    3
  ]
  [
    5
    6
  ]
])
param myArray array
```


##  Object Parameters

Objects have similar limitation to arrays in regards to declaration on multiple lines since Bicep uses newlines as a separator.

Below is an example of an object parameter with a default value.

```tsx
param storageAccountSettings object = {
  location: 'West US'
  sku: 'Standard_GRS'
  kind: 'StorageV2'
}
```

**NOTE:** Decorators [description](#description-decorator) and [metadata](#metadata-decorator) can be applied to any parameter including `object` parameters.

### Allowed Values

Decorator `allowed` can be applied to an object parameter, it might be not always that useful since it compares an entire object, but it is still worth knowing that this option exists.

In the following example we allow only predefined combinations of settings by specifying `allowed` constraint for object parameter.

```tsx
@allowed([
  {
    location: 'East US'
    sku: 'Standard_LRS'
  }
  {
    location: 'West US'
    sku: 'Standard_GRS'
  }
])
param storageAccountSettings object
```

## Related Posts

- [Variables In Azure Bicep - From Basics To Advanced](/blog/azure-bicep-variables)
- [Reference New Or Existing Resource In Azure Bicep](/blog/reference-new-or-existing-resource-in-azure-bicep)
- [Reference() Function Explained With Examples - ARM Template](/blog/reference-function-arm-template)

## Useful Links

- [What is Bicep?](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/bicep-overview){:target="_blank"}
- [What are ARM templates?](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview){:target="_blank"}
- [Bicep parameters spec](https://github.com/Azure/bicep/blob/main/docs/spec/parameters.md){:target="_blank"}
- [Tutorial: Use parameter files to deploy Azure Resource Manager Bicep file](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/bicep-tutorial-use-parameter-file?tabs=azure-powershell){:target="_blank"}
- [Add parameters to a Bicep template](https://github.com/Azure/bicep/blob/main/docs/tutorial/01-simple-template.md#add-parameters){:target="_blank"}
- [Bicep Playground](https://bicepdemo.z22.web.core.windows.net/){:target="_blank"}
