---
layout: post
title:  "How To Get App Configuration Connection String In ARM Template"
tags: arm-template
---


[App Configuration](https://docs.microsoft.com/en-us/azure/azure-app-configuration/overview){:target="_blank"} is a service in Azure that helps manage application settings and feature flags, this could be very useful in a distributed environment.

If you are using ARM templates to author your resources, you might want to retrieve a connection string from a newly created or already existing resource to pass it as a parameter to an application or just use it for some scripts inside of your deployment pipeline.

In this short post we will see how to get a connection string for `Microsoft.AppConfiguration/configurationStores` resource in the ARM template.

**Contents:**
* TOC
{:toc}

## listKeys function

We will use ARM template [listKeys](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource?tabs=json#listkeys){:target="_blank"} function to get keys data about the App Configuration resource. The syntax is the following:

`listKeys(resourceName or resourceIdentifier, apiVersion)`

App Configuration's implementation of `listKeys` function invokes [this API method](https://docs.microsoft.com/en-us/rest/api/appconfiguration/configurationstores/listkeys){:target="_blank"} under the hood, this link contains the contract and the response structure of this endpoint.

In general, `listKeys` return object varies depending on the resource and in case of the App Configuration it has the following structure (apiVersion 2020-06-01):

```json
{
  "value": [
    {
      "id": "bmk/-l1-s0:wreIfHE5AVdfbuUQdiB/",
      "name": "Primary",
      "value": "rvS7jvukwlyep0mI8qYgR6b3y6QN71UkSSKBmxcsjjU=",
      "connectionString": "Endpoint=https://cfg-contoso.azconfig.io;Id=bmk/-l1-s0:wreIfHE5AVdfbuUQdiB/;Secret=rvS7jvukwlyep0mI8qYgR6b3y6QN71UkSSKBmxcsjjU=",
      "lastModified": "2021-02-14T04:46:26+00:00",
      "readOnly": false
    },
    {
      "id": "bSMo-l1-s0:E2Ji37cEdkaQr3CadNR9",
      "name": "Secondary",
      "value": "+qgLfIdpBBqUrS7SVlPdhN8W9e+zk+F15WAUtddsJQI=",
      "connectionString": "Endpoint=https://cfg-contoso.azconfig.io;Id=bSMo-l1-s0:E2Ji37cEdkaQr3CadNR9;Secret=+qgLfIdpBBqUrS7SVlPdhN8W9e+zk+F15WAUtddsJQI=",
      "lastModified": "2021-02-14T04:46:26+00:00",
      "readOnly": false
    },
    {
      "id": "WAYb-l1-s0:VQXvOdg0XNWrPvMaE+o3",
      "name": "Primary Read Only",
      "value": "e4Nt5LzW93InEtZZMGLM28bYTNIDW59m8ZMrG4Nb8yU4=",
      "connectionString": "Endpoint=https://cfg-contoso.azconfig.io;Id=WAYb-l1-s0:VQXvOdg0XNWrPvMaE+o3;Secret=e4Nt5LzW93InEtZZMGLM28bYTNIDW59m8ZMrG4Nb8yU4=",
      "lastModified": "2021-02-14T04:46:26+00:00",
      "readOnly": true
    },
    {
      "id": "iREd-l1-s0:7aYj/Q2+mx3oTCI7x2sp",
      "name": "Secondary Read Only",
      "value": "/EKUA1/7e7RL93tFpe9SLXQzhU3BJ0/i8r47jX+pGjg=",
      "connectionString": "Endpoint=https://cfg-contoso.azconfig.io;Id=iREd-l1-s0:7aYj/Q2+mx3oTCI7x2sp;Secret=/EKUA1/7e7RL93tFpe9SLXQzhU3BJ0/i8r47jX+pGjg=",
      "lastModified": "2021-02-14T04:46:26+00:00",
      "readOnly": true
    }
  ]
}
```


## Getting connectionString

Now, given the output from the `listKeys` function above, it is very simple how to get a connection string for the configuration store.

The return object contains an array of 4 keys: **Primary**, **Secondary**, **Primary Read Only**, **Secondary Read Only**. We just need to select the key we are interested in, below is an example for connectionString of Primary key.

**NOTE:** Specify an array index for the `value` property to access the desired key. In the following example it is set to `0` to get Primary key.

```json
"[listKeys(resourceId('Microsoft.AppConfiguration/configurationStores', parameters('resourceName')), '2020-06-01').value[0].connectionString]"
```

Also, here is an example of ARM template that outputs **connectionString** of the App Configuration resource specified in the parameters (in this case `cfg-contoso`). You can use it in your template in a similar way.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "resourceType": {
            "defaultValue": "Microsoft.AppConfiguration/configurationStores",
            "type": "String"
        },
        "resourceName": {
            "defaultValue": "cfg-contoso",
            "type": "String"
        },
        "apiVersion": {
            "defaultValue": "2020-06-01",
            "type": "String"
        }
    },
    "resources": [],
    "outputs": {
        "connectionString": {
            "type": "String",
            "value": "[listKeys(resourceId(parameters('resourceType'), parameters('resourceName')), parameters('apiVersion')).value[0].connectionString]"
        }
    }
}
```
