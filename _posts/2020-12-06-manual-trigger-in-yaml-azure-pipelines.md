---
layout: post
title:  "Manual Trigger In Multi-Stage YAML Pipeline - Azure DevOps"
tags: azure-devops CI/CD
---

YAML pipelines in Azure Pipelines is a great functionality, however, at the time of this writing, it lacks some features. One of those is a manual trigger for a stage. Consider this sample use case:

1. PR is merged into the main branch.
2. Main pipeline is automatically invoked, YAML definition contains DEV, TEST, PROD stages to deploy to corresponding environments.
3. We want to deploy to DEV after each commit to master, however, not to TEST, there could be different reasons why we do not want to deploy to TEST right away after each commit.

## Solution

How to prevent the pipeline from executing TEST stage but be able to trigger it manually?

**As a workaround for not running stage automatically, a condition can be added to execute the stage only if the whole pipeline was triggered manually: `condition: eq(variables['Build.Reason'], 'Manual')`**

```yaml
- stage: TEST
  condition: eq(variables['Build.Reason'], 'Manual')
  jobs: Deploy
    steps:
    - script: echo Hello, World!
      displayName: 'Run a one-line script'
```

This condition should be added to the TEST stage. Of course, this check can be combined with other checks to form a larger condition. More about `Build.Reason` and other predefined variables [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml){:target="_blank"}.

## Result

When new commit is added to the main branch, pipeline is kicked off automatically but stops after DEV stage:

[![Pipeline stopped before TEST stage](/assets/img/manual-trigger-in-yaml-azure-pipelines/stopped-pipeline.png "Pipeline stopped before TEST stage")
_Pipeline stopped before TEST stage_](/assets/img/manual-trigger-in-yaml-azure-pipelines/stopped-pipeline.png){:target="_blank"}

By triggering run manually TEST stage will be executed if selected in  "Run pipeline" → "Stages to run":

[![Fully run pipeline](/assets/img/manual-trigger-in-yaml-azure-pipelines/full-pipeline.png "Fully run pipeline")
_Fully run pipeline_](/assets/img/manual-trigger-in-yaml-azure-pipelines/full-pipeline.png){:target="_blank"}


**Note:** To deploy to the TEST stage we need to create a new run manually. We cannot trigger TEST stage from an existing run.


## Alternative Solutions

- **Use environment approvals** to require manual input to run a particular stage. Information about environment approvals is [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops&tabs=check-pass){:target="_blank"}.
  **_Downside:_** stage is shown as pending until someone approves and fails after timeout. I do not prefer to a run to be marked as pending or failed, although everything executed correctly.

- **Keep two YAML pipelines**: one is triggered on every commit to the main branch and includes only Build and TEST, another one is triggered manually and includes all stages, here you can specify stages to run.
  **_Downside:_** maintenance of two pipelines.

- **Keep two pipelines: YAML + Classic**. Set up Classic Web UI Release Pipeline to deploy to higher environments (in our case TEST and PROD). In this approach we keep Build and DEV stages in yaml definition and TEST and PROD in classic release. Classic pipelines [documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/release/?view=azure-devops){:target="_blank"}.
  **_Downside:_** definition is spread across different places, classic release definition is not in source control.

- **Always run the pipeline manually**, this will allow to select stages to run.  To disable trigger set the following setting in your YAML file: **`trigger: none`**.


## Interesting Links
- [Suggestion/request/discussion](https://developercommunity.visualstudio.com/idea/629260/specify-manual-stages-in-multi-stage-yaml-pipeline.html){:target="_blank"} of a feature to specify manual trigger for a stage created in 2019.
- Stackoverflow question about [Manual Trigger on Azure Pipelines Stages (YAML)](https://stackoverflow.com/questions/58667596/manual-trigger-on-azure-pipelines-stages-yaml){:target="_blank"}.
- Completed work item [Approvals in YAML pipelines](https://dev.azure.com/mseng/AzureDevOpsRoadmap/_workitems/edit/1510336/){:target="_blank"}. However, I think it is different from a manual trigger.
