# Azure Pipeline Templates

- [Azure Pipeline Templates](#azure-pipeline-templates)
  - [Preset Use Case](#preset-use-case)
  - [Template Parameter Schema](#template-parameter-schema)

## Preset Use Case

This preset uses parameters based on a job type that predefines jobs and what stages they run in. For example, each item in the 'dockerFiles' parameter inserts a container image build job into the build stage. This allows you to provide a list of 'dockerFiles' that creates a job for each 'dockerFile' in the build stage that can run in parallel or with dependencies.

This is a prescriptive and opinionated preset template. In that, it abstracts the stages, jobs, and steps templates using a predefined pattern that is common for building and deploying infrastructure and applications.

## Template Parameter Schema

To use the pipeline templates in this repository it must be listed as a resource in your pipeline YAML file. This allows you to reference paths in another repository by using the resource identifier.

```yaml
name: $(Build.Repository.Name)_$(Build.SourceVersion)_$(Build.SourceBranchName) # name is the format for $(Build.BuildNumber)

resources:
  repositories:
    - repository: templates # Resource identifier for template usage
      type: github
      name: fitchtech/AzurePipelines # This repository
      ref: refs/tags/v1 # The tagged release of the repository
      endpoint: GitHub # Azure Service Connection Name

trigger:
  branches:
    include:
      - master # CI Trigger on commit to master
  tags:
    include:
      - v*.*.*-* # CI Trigger when tag matches format

extends:
  # file path to template at repo resource id to extend from
  template: presets/stages-jobTypes.yaml@templates
  parameters:
# code: stage in stages
  # dotNetTests: list of projects for dotNet test task of each item
    dotNetTests:
      - projects: '**[Uu]nit.[Tt]est*/*[Uu]nit.[Tt]est*.csproj' # Pattern search for unit test projects
        arguments: '--collect "Code Coverage" /p:CollectCoverage=true  /p:CoverletOutputFormat=cobertura /p:CoverletOutput=$(Common.TestResultsDirectory)\Coverage\'
        displayName: 'dotNet Unit Tests'
      - projects: '**[Cc][Ll][Ii].[Tt]est*/*[Cc][Ll][Ii].[Tt]est*.csproj' # Pattern search for cli test projects
        arguments: '--collect "Code Coverage" /p:CollectCoverage=true  /p:CoverletOutputFormat=cobertura /p:CoverletOutput=$(Common.TestResultsDirectory)\Coverage\'
        displayName: 'dotNet CLI Tests'
# build: stage in stages
  # dockerFiles: list of docker build jobs. Job, dockerFile, containerRegistry, and containerRepository, required for each
    dockerFiles:
      - job: containerImage1
        dockerFile: App1.Dockerfile
        containerRegistry: 'Docker' # Optional override of dockerRegistry parameter. Container registry service connection name
        containerRepository: 'App1' # Optional path within registry that overrides dockerRepository param. registry/repository/name:tag
      - job: containerImage2
        dependsOn: containerImage1
        dockerFile: App2.Dockerfile
        containerRegistry: 'Docker' # Optional override of dockerRegistry parameter. Container registry service connection name
        containerRepository: 'App2' # Optional path within registry that overrides dockerRepository param. registry/repository/name:tag
        dockerArgs: '--build-arg repository=baseimages/dotnet/core/aspnet'
        dockerTags: '$(Build.BuildNumber)' # Optional list of image tags. Default is $(Build.BuildNumber)
# deploy: stage in stages
    armTemplates:
      - deployment: 'armTemplate1' # Required: deployment name must be unique
        template: 'deployment1.json' # Required: ARM template file name
        parameters: 'parameters1.json' # Optional: ARM parameters file name
      - deployment: 'armTemplate2'
        template: 'deployment2.json'
        parameters: 'parameters2.json'
      - deployment: 'armTemplate3' # Required: deployment name must be unique
        template: 'deployment3.json'
        parameters: 'parameters3.json'
        subscription: 'subscriptionServiceConnectionName'
        resourceGroup: 'resourceGroupName'
      # dependsOn: Optional list of dependencies. Example of armTemplate3 depending on armTemplate1 and armTemplate2 succeeding
        dependsOn:
          - armTemplate1
          - armTemplate2
      # Insert steps for deployment strategy lifecycle hooks of the deployment
        preDeploy:
          - script: echo add stepList of tasks into preDeploy steps lifecycle hook
        routeTraffic:
          - script: echo add stepList of tasks into routeTraffic steps lifecycle hook
        postRouteTraffic:
          - script: echo add stepList of tasks into postRouteTraffic steps lifecycle hook
        onFailure:
          - script: echo add stepList of tasks into on failure steps lifecycle hook
        onSuccess:
          - script: echo add stepList of tasks into on failure steps lifecycle hook
    kubeDeployments:
      - deployment: kubeDeploy
        kubeServiceConnection: 'serviceConnectionName'
        manifests: 'deployment.yaml'
        action: deploy
        strategy: canary
      # Insert steps for deployment strategy lifecycle hooks of the deployment
        preDeploy:
          - script: echo add stepList of tasks into preDeploy steps lifecycle hook
        routeTraffic:
          - script: echo add stepList of tasks into routeTraffic steps lifecycle hook
        postRouteTraffic:
          - script: echo add stepList of tasks into postRouteTraffic steps lifecycle hook
        onFailure:
          - script: echo add stepList of tasks into on failure steps lifecycle hook
        onSuccess:
          - script: echo add stepList of tasks into on failure steps lifecycle hook

```