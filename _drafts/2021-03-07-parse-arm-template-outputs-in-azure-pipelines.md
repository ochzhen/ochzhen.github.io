---
layout: post
title:  "Parse ARM Template JSON Outputs In Azure Pipelines"
tags: azure-devops
---

It is common to have an ARM template deployment step as part of a pipeline/release in Azure DevOps. During this step resources are created or modified and we often want to export their names, URIs, IP addresses, keys for further use in the pipeline.

In this short article, we will take a look at how to parse [outputs from an ARM template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-outputs?tabs=azure-powershell){:target="_blank"} using PowerShell and Bash for both YAML-based and Classic release pipelines.

**Contents:**
* TOC
{:toc}


## Defining Sample Outputs Section

To make our discussion easier to follow, let's define a very simple ARM template that returns `someResourceUri` as part of outputs section.

Our goal is to retrieve `someResourceUri` value in the pipeline and create corresponding pipeline variable so that we can use it in the next steps.

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "resources": [],
    "outputs": {
        "someResourceUri": {
            "type": "string",
            "value": "https://example.com"
        }
    }
}
```


## How To Set Pipeline Variable In Scripts

In order to use output values from ARM template deployment somewhere further in a pipeline, we need to be able to create pipeline variables from parsed values.

It can be achieved using `task.setvariable` logging command. Logging command is a way for tasks and scripts to communicate with the agent, read more about syntax [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/scripts/logging-commands?view=azure-devops&tabs=powershell){:target="_blank"}.

There is a section in docs about [setting variables in scripts](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#set-variables-in-scripts){:target="_blank"}, but to keep it short, refer to the example below where `myVar` is the name of a pipeline variable, and `someValue` is value =).

```bash
echo "##vso[task.setvariable variable=myVar]someValue"
```

We will use this functionality to set variables for both YAML pipelines and Classic releases.


## Should I Use Bash Or PowerShell?

This decision is completely up to you, in this post we will look how to do it with either of them. But here are just a few notes:

- In examples we use pwsh - [PowerShell Core](https://github.com/PowerShell/PowerShell){:target="_blank"} which is the cross-platform version
- Microsoft-hosted agents usually have both pwsh and bash installed on both windows and linux agents, more information [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops&tabs=yaml#software){:target="_blank"}
- If your agents are self-hosted, then choice is simple, just use what's available


## YAML Pipeline

Below is shown a sample ARM template deployment step, and its outputs we would like to parse.

**NOTE:** Outputs are assigned to a pipeline variable `armOutputs` (deploymentOutputs setting).

```yaml
- stage: DEV
  jobs:
  - job: Deploy
    steps:

    - task: AzureResourceManagerTemplateDeployment@3
      inputs:
        deploymentScope: 'Resource Group'
        azureResourceManagerConnection: '...'
        subscriptionId: '8a918a0a-b436-484b-8317-fa5bb7fb9f67'
        action: 'Create Or Update Resource Group'
        resourceGroupName: 'rg-contoso'
        location: 'West US'
        templateLocation: 'Linked artifact'
        csmFile: 'template.json'
        deploymentMode: 'Incremental'
        deploymentOutputs: 'armOutputs'
```

### PowerShell

So, this is the actual part where we parse output values. Note that the second step just logs the parsed value, so won't probably need it.

- [ConvertFrom-Json](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json?view=powershell-7.1){:target="_blank"} is used to parse JSON
- `armOutputs` object is passed as `ARM_OUTPUTS` environment variable
- Using [pwsh task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/powershell?view=azure-devops){:target="_blank"}

```yaml
- pwsh: |
    $outputsObject = $env:ARM_OUTPUTS | ConvertFrom-Json
    Write-Host "##vso[task.setvariable variable=someResourceUriPwsh]$($outputsObject.someResourceUri.value)"
  displayName: 'Parse ARM deploymentOutputs | pwsh'
  env:
    ARM_OUTPUTS: $(armOutputs)
- script: echo $(someResourceUriPwsh)
  displayName: 'Log someResourceUriPwsh'
```

### Bash

Here is another way to parse outputs:

- [jq](https://stedolan.github.io/jq/){:target="_blank"} is used for parsing JSON, '-r' flag is passed to get value without quotes
- `armOutputs` object is passed as `ARM_OUTPUTS` environment variable
- Using [bash task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/bash?view=azure-devops){:target="_blank"}

```yaml
- bash: |
    echo "##vso[task.setvariable variable=someResourceUriBash]$(echo $ARM_OUTPUTS | jq -r '.someResourceUri.value')"
  displayName: 'Parse ARM deploymentOutputs | bash'
  env:
    ARM_OUTPUTS: $(armOutputs)
- script: echo $(someResourceUriBash)
  displayName: 'Log someResourceUriBash'
```


## Classic Release

The idea here is literally the same, the only difference is that it is now done through UI. We can use the same approach to set pipeline variables in scripts as illustrated above.

I will only briefly describe the steps since it's quite similar to what we've already discussed before.

### PowerShell

1. Add PowerShell task
2. Use inline script
3. Check "Use PowerShell Core" option in "Advanced" section
4. Set `ARM_OUTPUTS` variable to `$(armOutputs)` in "Environment Variables" section

```powershell
$outputsObject = $env:ARM_OUTPUTS | ConvertFrom-Json
Write-Host "##vso[task.setvariable variable=someResourceUriPwsh]$($outputsObject.someResourceUri.value)"
```

### Bash

1. Add Bash task
2. Use inline script
3. Set `ARM_OUTPUTS` variable to `$(armOutputs)` in "Environment Variables" section

```bash
echo "##vso[task.setvariable variable=someResourceUriBash]$(echo $ARM_OUTPUTS | jq -r '.someResourceUri.value')"
```
