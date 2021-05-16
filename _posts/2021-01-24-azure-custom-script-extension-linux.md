---
layout: post
title:  "Azure Custom Script Extension On Linux VM and VMSS"
tags: virtual-machine
---

Recently, there was a [post about Custom Script Extension for Windows](/blog/azure-custom-script-extension-windows), however, there's a similar functionality for Linux VMs. I was interested in it and decided to write a bit about this extension as well.

In general, the idea behind Custom Script Extension for Linux is the same as for Windows. However, there are some differences in usage which we will discuss.

This post has a lot of ideas and settings in common with Windows Custom Script Extension, that's why I'll try to keep this one short, mostly we'll walk through examples how we can use this extension.

Here's what we will do:

- Explore five possible ***use cases*** how to leverage this extension:
    - Using solely `commandToExecute` property when having only a few commands to run
    - Base64 `script` property for scripts under 256 KB
    - Common use case of running a shell script stored in a storage account
    - Executing a python script with preceding installation of modules
    - Writing and exporting custom logs during script execution
- Store scripts in storage account and use ***Shared Access Signature*** as well as ***Managed Identity*** to access them
- See ARM template examples for ***virtual machine scale sets*** but also learn how to apply the same examples on ***virtual machines***

**NOTE:** ARM template examples in this post are for Virtual Machine Scale Set (VMSS), but you can easily apply them to Virtual Machines (VM), please see [this section](#custom-script-extension-on-linux-vm) to understand what changes are needed.

**Contents:**
* TOC
{:toc}


## Prerequisites

In this article we will use two azure resources: *storage account* and *virtual machine scale set*. The next two subsections shortly describe these prerequisites.

### Storage Account

**NOTE:** This is only needed if we want to keep our script as a file or export logs. Storage account is not necessary when we specify commands only or base64 encoded script in ARM template

For our purposes a simple storage account is sufficient. We are going to use it to store our script(s) so that VM instances can download and run these custom scripts. And optionally we can use this storage account to export our custom logs.

Basically, we just need to create a storage account with blob containers and generate a SAS token:

- [Creating Storage Account and Containers](/blog/azure-custom-script-extension-windows#creating-storage-account-and-containers) - storage account and containers names are the same in this post
- [Generating Shared Access Signature (SAS) token](/blog/azure-custom-script-extension-windows#generating-sas-token)

### About VMSS

This should be your virtual machine scale set on which you want to apply your custom script. 

For illustration purposes I have created a VMSS resource in Azure Portal. A few notes about it:

- VMSS resource name is `vmss-contoso`
- Image - Ubuntu Server 18.04 LTS Gen1
- Load balancer is set up (simply by enabling corresponding checkbox)
- Upgrade mode - `Automatic` (this means that new changes to VMSS model will be applied automatically to all instances)

All other settings are default and it should be good enough.


## Inline Commands Without Script File

This is the simplest way to use the extension. We just put all commands we need inside `commandToExecute` setting. Consider this approach if you have only a few commands to run.

For example: `apt update && apt -y upgrade && apt install -y python3-pip`

The following JSON shows how our ARM template will look like. With the template in hand navigate to the section about [applying ARM template](#applying-arm-template).

```json
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vmssName": {
            "defaultValue": "vmss-contoso",
            "type": "string"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Compute/virtualMachineScaleSets/extensions",
            "apiVersion": "2019-03-01",
            "name": "[concat(parameters('vmssName'),'/CustomScriptExtension')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "publisher": "Microsoft.Azure.Extensions",
                "type": "CustomScript",
                "typeHandlerVersion": "2.1",
                "autoUpgradeMinorVersion": true,
                "settings": {
                    "timestamp": 202101091
                },
                "protectedSettings": {
                    "commandToExecute": "apt update && apt -y upgrade && apt install -y python3-pip"
                }
            }
        }
    ]
}
```


## Inline Base64 Encoded Script

There is an option to put our script directly inside of the ARM template. Interestingly, custom script extension for Windows doesn't have this functionality at this moment.

It could be very handy to use this option if our script is relatively small and can fit into 256 KB when compressed and encoded, in this case we don't need to store our script file in storage account or github.

**NOTES:**
- Setting name is `script`
- Must be base64 encoded
- Optionally can be compressed with gzip
- Maximum size is 256 KB
- Script is executed by `/bin/sh`
- Replaces `commandToExecute` and `fileUris` settings
- No need to store script file elsewhere and make it downloadable for VMSS

This Microsoft [documentation section](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux#property-script) describes this option well.

### Preparing Script

These steps are well illustrated in the documentation, but we'll include them here as well for completeness.

Let's write some code and save it as `script.sh` file.

```bash
#!/bin/sh
echo "Running custom script"
apt update
apt upgrade -y
```

Next, we compress (optional) and base64 encode (required). The following code snippets shows both of the options, choose whatever suites your needs better.

```bash
cat script.sh | base64 -w 0
cat script.sh | gzip -9 | base64 -w 0
```

The output will be the value of our `script` setting in ARM template.

### ARM Template

Below is an example of ARM template that applies the script on the VMSS. With this template you can jump straight to [this section](#applying-arm-template) to see how to apply it.

**NOTES:**

- We declare our extension as a separate resource but it can be declared in one of three ways: osProfile, child resource or separate resource
- `script` contains our base64 encoded script

```json
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vmssName": {
            "defaultValue": "vmss-contoso",
            "type": "string"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Compute/virtualMachineScaleSets/extensions",
            "apiVersion": "2019-03-01",
            "name": "[concat(parameters('vmssName'),'/CustomScriptExtension')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "publisher": "Microsoft.Azure.Extensions",
                "type": "CustomScript",
                "typeHandlerVersion": "2.1",
                "autoUpgradeMinorVersion": true,
                "settings": {
                    "timestamp": 202101091
                },
                "protectedSettings": {
                    "script": "IyEvYmluL3NoCmVjaG8gIlJ1bm5pbmcgY3VzdG9tIHNjcmlwdCIKYXB0IHVwZGF0ZQphcHQgdXBncmFkZSAteQo="
                }
            }
        }
    ]
}
```


## Regular Script

I think that this is the standard option how to use this extension. We just create a script file, upload it to our storage account and apply through ARM template.

### Creating Script

For our example let it be something as simple as outputting "Hello, World!" phrase. Our `script.sh` file:

```bash
#!/bin/sh
echo 'Hello, World!'
```

### Putting File Into Blob Storage

Simply upload the script above to a storage account and generate a SAS token as described in the [storage account section](#storage-account).

As a result, our file will have the following link:

- https://stcontoso.blob.core.windows.net/src/**script.sh**<your_SAS_token>

### ARM Template

To apply this template please navigate to [this section](#applying-arm-template).

```json
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vmssName": {
            "defaultValue": "vmss-contoso",
            "type": "string"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Compute/virtualMachineScaleSets/extensions",
            "apiVersion": "2019-03-01",
            "name": "[concat(parameters('vmssName'),'/CustomScriptExtension')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "publisher": "Microsoft.Azure.Extensions",
                "type": "CustomScript",
                "typeHandlerVersion": "2.1",
                "autoUpgradeMinorVersion": true,
                "settings": {
                    "timestamp": 202101101
                },
                "protectedSettings": {
                    "fileUris": [
                        "https://stcontoso.blob.core.windows.net/src/script.sh?sv=2019-12-12&.."
                    ],
                    "commandToExecute": "sh script.sh"
                }
            }
        }
    ]
}
```


## Python Script

In this use case we may want to run a python script, and maybe we need to install some python modules before that. Let's see how we can do it.

Luckily, Python is included in Ubuntu distribution by default. My sample VMSS with Ubuntu 18.04 LTS image has the following versions installed (we will use python 3 for our examples):

```bash
$ python --version
Python 2.7.17
$ python3 --version
Python 3.6.9
```

### Script Files and Python Modules Installation

We will create two files:

- `script.sh` - bash script which installs pip3 and required modules
- `hello.py` - sample script that uses installed modules

Please note that you can invoke `hello.py` from `script.sh`, but we will do this in the `commandToExecute` setting.

Our `script.sh` file, as an example we install `azure-storage-blob` module:

```bash
#!/bin/sh
apt update
apt upgrade -y
apt install -y python3-pip
# pip3 --version
pip3 install azure-storage-blob
```

Our `hello.py` file:

```python
from azure.storage.blob import BlobServiceClient

print('Hello, World!')
```

### Saving Files In Blob Storage

Now we need to put our script files into a blob storage. Please read [Storage Account section](#storage-account) for the instructions how to upload files and generate a SAS token.

As a result, we are going to have the following links:

- https://stcontoso.blob.core.windows.net/src/**script.sh**<your_SAS_token>
- https://stcontoso.blob.core.windows.net/src/**hello.py**<your_SAS_token>

### ARM Template

Here is a template which we can use to deploy our extension. To apply this extension go to [this section](#applying-arm-template).

**NOTES:**

- `fileUris` - contains links of our scripts stored in blob storage, URIs include a SAS token
- `commandToExecute` - first running `script.sh` and then `hello.py`

```json
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vmssName": {
            "defaultValue": "vmss-contoso",
            "type": "string"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Compute/virtualMachineScaleSets/extensions",
            "apiVersion": "2019-03-01",
            "name": "[concat(parameters('vmssName'),'/CustomScriptExtension')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "publisher": "Microsoft.Azure.Extensions",
                "type": "CustomScript",
                "typeHandlerVersion": "2.1",
                "autoUpgradeMinorVersion": true,
                "settings": {
                    "timestamp": 202101101
                },
                "protectedSettings": {
                    "fileUris": [
                        "https://stcontoso.blob.core.windows.net/src/script.sh?sv=2019-12-12&..",
                        "https://stcontoso.blob.core.windows.net/src/hello.py?sv=2019-12-12&.."
                    ],
                    "commandToExecute": "sh script.sh && python3 hello.py"
                }
            }
        }
    ]
}
```


## Script With Custom Logs

Lastly, we'll just discuss how we can write logs during the execution of the script and export them into our Storage Account. This way we don't need to connect to our instances to view extension logs.

The idea is simple:

- Write your custom logs to a file
- Upload this file to a Storage Account using REST API and SAS token

**NOTE:** I'm not including the code here but I'm sure that you can easily do this by yourself.

To help you a bit, here is an implementation in PowerShell, you can adapt it to Bash:

- [Writing Script With Logs](/blog/azure-custom-script-extension-windows#writing-script-with-logs) gives an idea what how to upload a file to blob storage via REST API
- [Viewing Logs](/blog/azure-custom-script-extension-windows#viewing-logs) shows how logs will be exported to a storage account

## Applying ARM Template

With an ARM template in hand you can apply it in multiple ways. For ad-hoc extension run I prefer using Azure Portal, but you can also incorporate it as part of your CI/CD pipeline.

[Here](/blog/azure-custom-script-extension-windows#custom-template-deployment-in-azure-portal) is a guidance on how to create a custom template deployment in Azure Portal from a related post. This way you can deploy your ARM template so that the extension is applied.

**IMPORTANT:** You may need a `dependsOn` property in the extension definition if you deploy script and VMSS in the same template. In this way extension will be deployed only after VMSS.

**NOTE:** Changing `timestamp` property in the ARM template and redeploying causes script to be rerun. It could be needed if our custom script extension is already installed and we want to run it again.


## Using Managed Identity To Fetch Scripts

In the previous examples we used Shared Access Signature (SAS) to make our script files downloadable from VMSS instances. However, this is not the only to achieve this.

It is possible to use managed identity (system or user assigned) or storage account key as well. In this section we will use ***user assigned managed identity***.

**NOTE:** You might want to look at [an example of system assigned managed identity usage](/blog/azure-custom-script-extension-windows#using-managed-identity-instead-of-sas) as well. Please note that the ARM template in there is for Windows version of custom script extension, so for Linux slight changes are needed.

### Creating User Assigned Managed Identity

First step is to create a user assigned managed identity, we can do it in Azure Portal in just a few clicks. 

**NOTES:**

- Our user assigned managed identity resource name is `mi-contoso`
- We will use "Client ID" and "Object ID" values in the next steps

As a result, your managed identity might look like on the following screenshot.

[![Managed identity](/assets/img/azure-custom-script-extension-linux/managed-identity.png "Managed identity")
_Managed identity_](/assets/img/azure-custom-script-extension-linux/managed-identity.png){:target="_blank"}

### Assigning Managed Identity To VMSS

Go to "VMSS → Identity tab → User assigned" and add the managed identity created in the previous step.

[![VMSS with assigned managed identity](/assets/img/azure-custom-script-extension-linux/assigned-identity-to-vmss.png "VMSS with assigned managed identity")
_VMSS with assigned managed identity_](/assets/img/azure-custom-script-extension-linux/assigned-identity-to-vmss.png){:target="_blank"}

### Giving Permissions To Access Storage Account

Go to "Storage Account → Access Control (IAM) tab → Add role assignment". Assign "***Storage Blob Data Reader***" role to the managed identity.

[![Storage account permissions](/assets/img/azure-custom-script-extension-linux/storage-account-permissions.png "Storage account permissions")
_Storage account permissions_](/assets/img/azure-custom-script-extension-linux/storage-account-permissions.png){:target="_blank"}

### Crafting ARM Template

The last step is to create an ARM template that we will deploy to apply our extension. The template itself is not much different from the previous examples except for a couple of properties.

**NOTES:**
- `fileUris` links doesn't need SAS token anymore since it is retrieved using managed identity
- `managedIdentity` property is required <br>
**IMPORTANT:** It can contain either `clientId` or `objectId` of the managed identity but ***not both***, otherwise you'll get an error

```json
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "vmssName": {
            "defaultValue": "vmss-contoso",
            "type": "string"
        }
    },
    "variables": {},
    "resources": [
        {
            "type": "Microsoft.Compute/virtualMachineScaleSets/extensions",
            "apiVersion": "2019-03-01",
            "name": "[concat(parameters('vmssName'),'/CustomScriptExtension')]",
            "location": "[resourceGroup().location]",
            "properties": {
                "publisher": "Microsoft.Azure.Extensions",
                "type": "CustomScript",
                "typeHandlerVersion": "2.1",
                "autoUpgradeMinorVersion": true,
                "settings": {
                    "timestamp": 202101101
                },
                "protectedSettings": {
                    "fileUris": [
                        "https://stcontoso.blob.core.windows.net/src/script.sh"
                    ],
                    "commandToExecute": "sh script.sh",
                    "managedIdentity": {
                        "objectId": "17d1aa9d-3310-4687-a671-b4cfcaae8fbe"
                    }
                }
            }
        }
    ]
}
```


## Custom Script Extension On Linux VM

Lastly, let's briefly discuss how easily we can adapt all VMSS examples above to a regular virtual machines.

**IMPORTANT:** The only change needed is to set extension type to `Microsoft.Compute/virtualMachines/extensions`

Simply changing the extension type should make it work for VMs as well. Just remember to provide correct name of your VM resource and specify `dependsOn` section if needed.

## Useful Links

- [https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux](https://docs.microsoft.com/en-us/azure/virtual-machines/extensions/custom-script-linux)
- [https://github.com/Azure/azure-quickstart-templates/tree/master/201-vm-custom-script-output](https://github.com/Azure/azure-quickstart-templates/tree/master/201-vm-custom-script-output)
- [https://github.com/Azure/custom-script-extension-linux](https://github.com/Azure/custom-script-extension-linux)