---
title: Deployment jobs
description: Deploy to resources within an environment
ms.prod: devops
ms.technology: devops-cicd
ms.topic: conceptual
ms.assetid: fc825338-7012-4687-8369-5bf8f63b9c10
ms.manager: mijacobs
ms.author: jukullam
author: juliakm
ms.date: 5/2/2019
monikerRange: azure-devops
---

# Deployment jobs

[!INCLUDE [version-team-services](../includes/version-team-services.md)]

> [!NOTE]
> To use deployment jobs, [make sure the multi-stage pipelines experience is turned on](../../project/navigation/preview-features.md).

In YAML pipelines, we recommend that you put your deployment steps in a deployment job. A deployment job is a special type of [job](phases.md) that's a collection of steps, which are run sequentially against the environment.

Deployment jobs provide the following benefits:

 - **Deployment history**: You get end-to-end deployment history across pipelines, down to a specific resource and status of the deployments for auditing.
 - **Apply deployment strategy**: You define how your application is rolled out.

   > [!NOTE] 
   > We currently only support the *runOnce*, *rolling*, and the *canary* strategies. 

## Schema

Here's the full syntax to specify a deployment job: 

```YAML
jobs:
- deployment: string   # name of the deployment job, A-Z, a-z, 0-9, and underscore
  displayName: string  # friendly name to display in the UI
  pool:                # see pool schema
    name: string
    demands: string | [ string ]
  dependsOn: string 
  condition: string 
  continueOnError: boolean                # 'true' if future jobs should run even if this job fails; defaults to 'false'
  container: containerReference # container to run this job inside
  services: { string: string | container } # container resources to run as a service container
  timeoutInMinutes: nonEmptyString        # how long to run the job before automatically cancelling
  cancelTimeoutInMinutes: nonEmptyString  # how much time to give 'run always even if cancelled tasks' before killing them
  variables: { string: string } | [ variable | variableReference ]  
  environment: string  # target environment name and optionally a resource-name to record the deployment history; format: <environment-name>.<resource-name>
  strategy: [ deployment strategy ] # see deployment strategy schema
```

### Deployment strategies

When you're deploying application updates, it's important that the technique you use to deliver the update will: 

* Enable initialization.
* Deploy the update.
* Route traffic to the updated version.
* Test the updated version after routing traffic.
* In case of failure, run steps to restore to the last known good version. 

We achieve this by using life cycle hooks that can run steps during deployment. Each of the life cycle hooks resolves into an agent job or a [server job](https://docs.microsoft.com/azure/devops/pipelines/process/phases?view=azure-devops&tabs=yaml#server-jobs) (or a container or validation job in the future), depending on the `pool` attribute. By default, the life cycle hooks will inherit the `pool` specified by the `deployment` job. 

#### Descriptions of life cycle hooks

`preDeploy`: Used to run steps that initialize resources before application deployment starts. 

`deploy`: Used to run steps that deploy your application. Download artifact task will be auto injected only in the `deploy` hook for deployment jobs. To stop downloading artifacts, use `- download: none` or choose specific artifacts to download by specifying [Download Pipeline Artifact task](https://docs.microsoft.com/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema#download).

`routeTraffic`: Used to run steps that serve the traffic to the updated version. 

`postRouteTraffic`: Used to run the steps after the traffic is routed. Typically, these tasks monitor the health of the updated version for defined interval. 

`on: failure` or `on: success`: Used to run steps that perform rollback actions or clean-up. 

### RunOnce deployment strategy

`runOnce` is the simplest deployment strategy wherein all the life cycle hooks, namely `preDeploy` `deploy`, `routeTraffic`, and `postRouteTraffic`, are executed once. Then,  either `on:` `success` or `on:` `failure` is executed.  

```YAML
strategy: 
    runOnce:
      preDeploy:        
        pool: [ server | pool ] # see pool schema        
        steps:
        - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
      deploy:          
        pool: [ server | pool ] # see pool schema        
        steps:
        ...
      routeTraffic:         
        pool: [ server | pool ]         
        steps:
        ...        
      postRouteTraffic:          
        pool: [ server | pool ]        
        steps:
        ...
      on:
        failure:         
          pool: [ server | pool ]           
          steps:
          ...
        success:          
          pool: [ server | pool ]           
          steps:
          ...
```

### Rolling deployment strategy

A rolling deployment replaces instances of the previous version of an application with instances of the new version of the application on a fixed set of virtual machines (rolling set) in each iteration. 

We currently only support the rolling strategy to VM resources.

For example, a rolling deployment typically waits for deployments on each set of virtual machines to complete before proceeding to the next set of deployments. You could do a health check after each iteration and if a significant issue occurs, the rolling deployment can be stopped.

Rolling deployments can be configured by specifying the keyword `rolling:` under `strategy:` node. 
The `strategy.name` variable is available in this strategy block which takes the name of the strategy. In this case, rolling.

```YAML
strategy:
  rolling:
    maxParallel: [ number or percentage as x% ]
    preDeploy:        
      steps:
      - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
    deploy:          
      steps:
      ...
    routeTraffic:         
      steps:
      ...        
    postRouteTraffic:          
      steps:
      ...
    on:
      failure:         
        steps:
        ...
      success:          
        steps:
        ...
```
All the lifecycle hooks are supported and lifecycle hook jobs are created to run on each VM.

`preDeploy`, `deploy`, `routeTraffic`, and `postRouteTraffic` are executed once per batch size defined by `maxParallel`. 
Then, either `on: success` or `on: failure` is executed.

With `maxParallel: <# or % of VMs>`, you can control the number/percentage of virtual machine targets to deploy to in parallel. This ensures that the app is running on these machines and is capable of handling requests while the deployment is taking place on the rest of the machines which reduces overall downtime.

 > [!NOTE]
 > There are a few known gaps in this feature. For example, when you retry a stage, it will re-run the deployment on all VMs not just failed targets. 

### Canary deployment strategy

Canary deployment strategy is an advanced deployment strategy that helps mitigate the risk involved in rolling out new versions of applications. By using this strategy, you can roll out the changes to a small subset of servers first. As you gain more confidence in the new version, you can release it to more servers in your infrastructure and route more traffic to it. 

Currently, this is applicable to only Kubernetes resources.


```YAML
strategy: 
    canary:
      increments: [ number ]
      preDeploy:        
        pool: [ server | pool ] # see pool schema        
        steps:
        - script: [ script | bash | pwsh | powershell | checkout | task | templateReference ]
      deploy:          
        pool: [ server | pool ] # see pool schema        
        steps:
        ...
      routeTraffic:         
        pool: [ server | pool ]         
        steps:
        ...        
      postRouteTraffic:          
        pool: [ server | pool ]        
        steps:
        ...
      on:
        failure:         
          pool: [ server | pool ]           
          steps:
          ...
        success:          
          pool: [ server | pool ]           
          steps:
          ...
```
Canary deployment strategy supports the `preDeploy` life cycle hook (executed once) and iterates with the `deploy`, `routeTraffic`, and `postRouteTraffic` life cycle hooks. It then exits with either the `success` or `failure` hook.

 
The following variables are available in this strategy:

`strategy.name`: Name of the strategy. For example, canary.
<br>`strategy.action`: The action to be performed on the Kubernetes cluster. For example, deploy, promote, or reject.
<br>`strategy.increment`: The increment value used in the current interaction. This variable is available only in `deploy`, `routeTraffic`, and `postRouteTraffic` life cycle hooks.



## Examples

### RunOnce deployment strategy

The following example YAML snippet showcases a simple use of a deploy job by using the `runOnce` deployment strategy. 

```YAML
jobs:
  # track deployments on the environment
- deployment: DeployWeb
  displayName: deploy Web App
  pool:
    vmImage: 'Ubuntu-16.04'
  # creates an environment if it doesn't exist
  environment: 'smarthotel-dev'
  strategy:
    # default deployment strategy, more coming...
    runOnce:
      deploy:
        steps:
        - script: echo my first deployment
```

With each run of this job, deployment history is recorded against the `smarthotel-dev` environment.

> [!NOTE]
> - It's also possible to create an environment with empty resources and use that as an abstract shell to record deployment history, as shown in the previous example.

The next example demonstrates how a pipeline can refer both an environment and a resource to be used as the target for a deployment job.

```YAML
jobs:
- deployment: DeployWeb
  displayName: deploy Web App
  pool:
    vmImage: 'Ubuntu-16.04'
  # records deployment against bookings resource - Kubernetes namespace
  environment: 'smarthotel-dev.bookings'
  strategy: 
    runOnce:
      deploy:
        steps:
          # No need to explicitly pass the connection details
        - task: KubernetesManifest@0
          displayName: Deploy to Kubernetes cluster
          inputs:
            action: deploy
            namespace: $(k8sNamespace)
            manifests: |
              $(System.ArtifactsDirectory)/manifests/*
            imagePullSecrets: |
              $(imagePullSecret)
            containers: |
              $(containerRegistry)/$(imageRepository):$(tag)
```

This approach has the following benefits:
- Records deployment history on a specific resource within the environment, as opposed to recording the history on all resources within the environment.
- Steps in the deployment job **automatically inherit** the connection details of the resource (in this case, a Kubernetes namespace, `smarthotel-dev.bookings`), because the deployment job is linked to the environment. 
This is particularly useful in the cases where the same connection detail is set for multiple steps of the job.


### Rolling deployment strategy

The rolling strategy for VMs updates up to 5 targets in each iteration. `maxParallel` will determine the number of targets that can be deployed to, in parallel. The selection accounts for absolute number or percentage of targets that must remain available at any time excluding the targets that are being deployed to. It is also used to determine the success and failure conditions during deployment.

```YAML
jobs: 
- deployment: VMDeploy
  displayName: web
  environment:
    name: smarthotel-dev
    resourceType: VirtualMachine
  strategy:
    rolling:
      maxParallel: 5  #for percentages, mention as x%
      preDeploy:
        steps:
        - download: current
          artifact: drop
        - script: echo initialize, cleanup, backup, install certs
      deploy:
        steps:
        - task: IISWebAppDeploymentOnMachineGroup@0
          displayName: 'Deploy application to Website'
          inputs:
            WebSiteName: 'Default Web Site'
            Package: '$(Pipeline.Workspace)/drop/**/*.zip'
      routeTraffic:
        steps:
        - script: echo routing traffic
      postRouteTraffic:
        steps:
        - script: echo health check post-route traffic
      on:
        failure:
          steps:
          - script: echo Restore from backup! This is on failure
        success:
          steps:
          - script: echo Notify! This is on success
```

### Canary deployment strategy

In the next example, the canary strategy for AKS will first deploy the changes with 10-percent pods, followed by 20 percent, while monitoring the health during `postRouteTraffic`. If all goes well, it will promote to 100 percent.  

```YAML
jobs: 
- deployment: 
  environment: smarthotel-dev.bookings
  pool: 
    name: smarthotel-devPool
  strategy:                  
    canary:      
      increments: [10,20]  
      preDeploy:                                     
        steps:           
        - script: initialize, cleanup....   
      deploy:             
        steps: 
        - script: echo deploy updates... 
        - task: KubernetesManifest@0 
          inputs: 
            action: $(strategy.action)       
            namespace: 'default' 
            strategy: $(strategy.name) 
            percentage: $(strategy.increment) 
            manifests: 'manifest.yml' 
      postRouteTaffic: 
        pool: server 
        steps:           
        - script: echo monitor application health...   
      on: 
        failure: 
          steps: 
          - script: echo clean-up, rollback...   
        success: 
          steps: 
          - script: echo checks passed, notify... 
```