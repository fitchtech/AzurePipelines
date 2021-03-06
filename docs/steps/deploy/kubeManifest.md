# Kubernetes Manifest Steps Template

- [Kubernetes Manifest Steps Template](#kubernetes-manifest-steps-template)
  - [Steps Template Usage](#steps-template-usage)
  - [Insert Steps Template into Stages Template](#insert-steps-template-into-stages-template)

## Steps Template Usage

- Deploy Kubernetes Manifests to the Kubernetes Service Connection or ADO Environment Resource
- Deploy standard or canary pods for deployment manifests
- Create/update image pull secret using docker and Kubernetes service connection
- Bake Kustomize, Docker Compose, or Helm2 charts that are then deployed
- Scale an existing replica set
- Delete a Kubernetes object
- Create Kubernetes secrets from Azure KeyVault Secrets

## Insert Steps Template into Stages Template

The following example shows how to insert the kubeManifest steps template into the [stages](../../stages.md) template with the minimum required params.

```yml
name: $(Build.Repository.Name)_$(Build.SourceVersion)_$(Build.SourceBranchName) # name is the format for $(Build.BuildNumber)

parameters:
# params to pass into stages.yaml template:

- name: deployPool # Nested into pool param of deploy jobs
  type: object
  default:
    vmImage: 'ubuntu-18.04'
- name: dockerRegistryEndpoint # Container registry and Kubernetes service connection params used to create image pull secret in Kubernetes for the registry
  type: string
  default: '' # ADO Service Connection name
- name: kubernetesServiceConnection # Kubernetes Service Connection Name
  type: string
  default: ''
- name: namespace
  type: string
  default: default
- name: manifests # Deployment manifest for canary deploy, promote, and reject jobs
  type: object
  default: '$(Pipeline.Workspace)/**deployment.yaml'
- name: strategy
  type: string
  default: canary
  values:
  - canary
  - runOnce

# parameter defaults in the above section can be set on the manual run of a pipeline to override

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
# template: file path at repo resource id to extend from
  template: stages.yaml@templates
# parameters: within stages.yaml@templates
  parameters:
  # code: jobList inserted into code stage in stages
  # build: jobList inserted into build stage in stages
  # deploy: deploymentList inserted into deploy stage in stages param
    deploy:
      - deployment: kubeDeploy # job name unique to stage
        displayName: 'Canary Deployment'
        pool: ${{ parameters.deployPool }} # param passed to pool of deployment jobs
        condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
      # variables:
        # key: 'value' # pairs of variables scoped to this job
        dependsOn: []
        strategy:
          ${{ parameters.strategy }}:
            deploy:
              steps:
              - template: steps/deploy/kubeManifest.yaml
                parameters:
                # preSteps: 
                  # - task: add preSteps into job
                  namespace: ${{ parameters.namespace }} # pass in namespace param
                  imagePullSecret: 'registry-cred'
                  dockerRegistryEndpoint: ${{ parameters.dockerRegistryEndpoint }}
                  kubernetesServiceConnection: ${{ parameters.kubernetesServiceConnection }} # pass param for kube manifest deployment service connection
                  action: deploy
                  strategy: ${{ parameters.strategy }}
                  manifests: ${{ parameters.manifests }}
                # postSteps:
                  # - task: add postSteps into job

  # promote: deploymentList inserted into promote stage in stages param
    promote:
      - ${{ if eq(parameters.strategy, 'canary') }}:
        - deployment: kubePromote # job name unique to stage
          displayName: 'Promote Canary Deployment'
          pool: ${{ parameters.deployPool }} # param passed to pool of deployment jobs
          condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
        # variables:
          # key: 'value' # pairs of variables scoped to this job
          dependsOn: []
          strategy:
            runOnce:
              deploy:
                steps:
                - template: steps/deploy/kubeManifest.yaml
                  parameters:
                  # preSteps: 
                    # - task: add preSteps into job
                    namespace: ${{ parameters.namespace }} # pass in namespace param
                    kubernetesServiceConnection: ${{ parameters.kubernetesServiceConnection }} # pass param for kube manifest deployment service connection
                    action: promote
                    strategy: canary
                    manifests: ${{ parameters.manifests }}
                  # postSteps:
                    # - task: add postSteps into job

  # reject: deploymentList inserted into reject stage in stages param
    reject:
      - ${{ if eq(parameters.strategy, 'canary') }}:
        - deployment: kubeReject # job name unique to stage
          displayName: 'Reject Canary Deployment'
          pool: ${{ parameters.deployPool }} # param passed to pool of deployment jobs
          condition: and(failed(), ne(variables['Build.Reason'], 'PullRequest'))
        # variables:
          # key: 'value' # pairs of variables scoped to this job
          dependsOn: []
          strategy:
            runOnce:
              deploy:
                steps:
                - template: steps/deploy/kubeManifest.yaml
                  parameters:
                  # preSteps: 
                    # - task: add preSteps into job
                    namespace: ${{ parameters.namespace }} # pass in namespace param
                    kubernetesServiceConnection: ${{ parameters.kubernetesServiceConnection }} # pass param for kube manifest deployment service connection
                    action: reject
                    strategy: canary
                    manifests: ${{ parameters.manifests }}
                  # postSteps:
                    # - task: add postSteps into job

  # test: jobList inserted into test stage in stages param

```
