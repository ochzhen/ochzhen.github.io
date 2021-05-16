---
layout: post
title:  "Use Azure PowerShell Module in Azure Functions - Quick Guide"
tags: azure-function powershell
---


Thanks to the PowerShell Core support in [Azure Functions](https://azure.microsoft.com/en-us/updates/powershell-support-in-azure-functions-is-now-generally-available/){:target="_blank"} we can now also use [PowerShell Az module](https://docs.microsoft.com/en-us/powershell/azure/?view=azps-5.7.0){:target="_blank"} in Function Apps.

**To make PowerShell Az module available in an Azure Function, `managedDependency` property has to be enabled in `host.json` file, and `Az='5.*'` module version included in `requirements.psd1` file.**

It is great that now Azure Functions can be also used for infrastructure management and scripting. For example, recently I used it with Azure PowerShell to write a function that retrieves information about traffic managers in a subscription to power a simple dashboard.

**IMPORTANT:** This post is about using Az PowerShell module **inside** of an Azure Function with PowerShell runtime. It is NOT about managing Azure Functions with PowerShell.

First two steps are necessary to make Az module available in the function runtime, the following optional steps go further by describing how to configure managed identity, grant permissions and connect to a subscription inside of the script.

**Contents:**
* TOC
{:toc}


## [Required] Enable managedDependency property in host.json

We need to verify that managedDependency is enabled in [host.json](https://docs.microsoft.com/en-us/azure/azure-functions/functions-host-json){:target="_blank"}, it is set to true by default when a PowerShell functions project is created.

[PowerShell gallery](https://www.powershellgallery.com/){:target="_blank"} is used to manage dependencies, and the list of required modules is taken from `requirements.psd1`.

```json
{
  "...": "...",
  "managedDependency": {
    "Enabled": true
  },
  "...": "..."
}
```


## [Required] Including Az Module in requirements.psd1

Since `requirements.psd1` is used for determining what modules need to be installed, we have to specify our Az module and its desired version.

We can specify an exact version or only major version, in the latter case minor versions will be updated automatically.

```powershell
@{
    'Az' = '5.*'
}
```


## Configuring Managed Identity

[Managed Identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview){:target="_blank"} is a convenient and secure way to access Azure resources, it is managed by the platform which significantly simplifies developer's life.

In this example we will use **system-assigned** managed identity for Azure Function but the process is almost identical for *user-assigned*.

Creating a system-assigned managed identity for a Function App is extremely simple:

1. Go to "Identity" section
2. Select "System assigned"
3. Set status to "On"

[![System-assigned managed identity](/assets/img/az-powershell-module-in-azure-functions/managed-identity.png "System-assigned managed identity") _System-assigned managed identity_](/assets/img/az-powershell-module-in-azure-functions/managed-identity.png){:target="_blank"}


## Granting Permissions

This step depends on what you want to access from the Azure Function, it could be an entire subscription, a resource group or a particular resource, read more about [role-based access control](https://docs.microsoft.com/en-us/azure/role-based-access-control/overview){:target="_blank"}.

Just as an example we will assign Reader role for a subscription to our system-assigned managed identity.

Here are the steps: Subscription > Access control (IAM) > Add > Add role assignment > Select Reader Role > Find managed identity by the Function App's name > Save.


## Connecting to a Subscription

It depends on the logic you want to run in an Azure Function but it is still quite likely that you'll need to specify a subscription to work with, this is where your resources are. It can be done with [Set-AzContext command](https://docs.microsoft.com/en-us/powershell/module/az.accounts/set-azcontext?view=azps-5.7.0){:target="_blank"}.

An example is shown below, first we set the correct subscription and then retrieve information about a storage account.

```powershell
Set-AzContext -Subscription "256e8e6c-8f2e-4153-8507-d9cd404b3728"
$StorageAccount = Get-AzStorageAccount -ResourceGroupName rg-contoso -Name stcontoso
```


## Editing Files in Azure Portal

If you are just experimenting or doing some proof of concept, you might not want to set up and edit files locally, then publish to Azure. Luckily, in this case we can do it fully in Azure Portal.

Files `host.json` and `requirements.psd1` can be edited under "App files" section on the Function App page. Just select the file you want to edit in the dropdown.

[![Editing app files in portal](/assets/img/az-powershell-module-in-azure-functions/portal-app-files.png "Editing app files in portal") _Editing app files in portal_](/assets/img/az-powershell-module-in-azure-functions/portal-app-files.png){:target="_blank"}

Similarly for function code in `run.ps1` and `function.json` - edit them under "Code + Test" section of the function's page.

[![Editing function files in portal](/assets/img/az-powershell-module-in-azure-functions/portal-code-test.png "Editing function files in portal") _Editing function files in portal_](/assets/img/az-powershell-module-in-azure-functions/portal-code-test.png){:target="_blank"}


## Useful Links
- [Azure Functions PowerShell developer guide](https://docs.microsoft.com/en-us/azure/azure-functions/functions-reference-powershell?tabs=portal){:target="_blank"}
- [Azure Az PowerShell module](https://docs.microsoft.com/en-us/powershell/azure/new-azureps-module-az?view=azps-5.7.0){:target="_blank"}
- [PowerShell Core GitHub](https://github.com/PowerShell/PowerShell){:target="_blank"}
- [Managed identities in Azure](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview){:target="_blank"}
