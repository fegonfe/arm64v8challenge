# arm64v8challenge
[![GitHub license](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](https://raw.githubusercontent.com/fegonfe/arm64v8challenge/master/LICENSE)

Build arm64v8 docker containers and release to IoT Edge using Azure DevOps

## First things first
It all begins with an OS image. You probably are not running ARM64 on your dev environment, but you need to build a container to run on this platform. There are some arm64v8 container images on http://hub.docker.com, but how can you build them? If you have Windows 10 or macOS, it is very straightforward because docker already does the job of running a container of a different platform.

Linux has been doing this forever using [QEMU](http://wiki.qemu.org/), but how to do this on a build agent? This [article](https://docs.microsoft.com/en-us/azure/devops/pipelines/languages/docker?view=azure-devops#build-arm-containers) explains the requirements: 
1. Author your Dockerfile so that a binary of QEMU (for the target platform) exists in the base Docker image.
2. Run a privileged container that register QEMU on the build agent.

**Step 1.** Create a resource group and a private container registry:
```powershell
az group create --name arm64v8challenge --location eastus
az acr create --resource-group arm64v8challenge --name arm64v8cr --sku Basic
```
**Step 2.** Create a service principal with AcrPull permission to be used by the device.
```powershell
$scope = $(az acr show --name arm64v8cr --query id --output tsv)
$iotpwd=$(az ad sp create-for-rbac --name iotdevice-pull --scopes $scope --role acrpull --query password --output tsv)
$iotappId=$(az ad sp show --id http://iotdevice-pull --query appId)
```
**Step 3.** Create a key vault to save the credential. You don't want to save the password in plain text in the yaml file.
```powershell
az keyvault create --name arm64v8KeyVault --resource-group arm64v8challenge --location eastus
az keyvault secret set --vault-name arm64v8KeyVault --name iot-device-user --value $iotappId
az keyvault secret set --vault-name arm64v8KeyVault --name iot-device-pwd --value $iotpwd
```
**Step 4.** Create a new service principal with permissions to get the secrets from the key vault.
```powershell
$devopspwd=$(az ad sp create-for-rbac --name sp-arm64v8challenge --skip-assignment --query password --output tsv)
$devopsappId=$(az ad sp show --id http://sp-arm64v8challenge --query appId)
az keyvault set-policy --name arm64v8KeyVault --spn http://sp-arm64v8challenge --secret-permissions get list
```
**Step 5.** Create the service connections that you are going to reference in the pipelines. On Azure DevOps, open Project Settings (last option on the left menu). Under Pipelines, click on Service Connections and create a new service connection. Select Azure Resource Manager and click on the link "use the full version of the service connection dialog". With this, you can use the service principal that you have created in the last step. Configure it using the following values:
* Connection name: arm-serviceconnection
* Scope level: Subscription
* Service principal client ID: {fill it with the value of $devopsappId variable}
* Service principal key: {fill it with the value of $devopspwd variable}

Create a new Service Connection. Select Docker Registry and configure using the following values:
* Registry type: Azure Container Registry
* Connection name: acr-serviceconnection
* Azure subscription: {your subscription}
* Azure container registry: arm64v8cr

**Step 6.** Create a new repo on Azure Repo (or github).

**Step 7.** Create a new folder and name it ubuntu16.04/base and set Dockerfile as the file name. Copy and Paste the following to the Dockerfile:

    ARG UBUNTU_VERSION=xenial
    FROM arm64v8/ubuntu:${UBUNTU_VERSION} as base
    # this will satisfy the requirement to have the binary inside the image
    COPY qemu-aarch64-static /usr/bin 

**Step 8.** Create a new folder and name it deploy and set deployment.template.json as the file name. Copy and Paste the content from [this file](https://raw.githubusercontent.com/fegonfe/arm64v8challenge/master/deploy/deployment.template.json).

**Step 9.** Create a new folder and name it pipelines and set ubuntu16.04-base-build.yml as the file name. Copy and Paste the following to the yml file:
```yml
pool:
  name: Hosted Ubuntu 1604
steps:
- bash: |
   # Install QEMU on host agent
   sudo apt-get update
   sudo apt-get install -y qemu qemu-user-static qemu-user binfmt-support
   
   # copy to sourcedir       
   cp /usr/bin/qemu-aarch64-static $sourcedir

  displayName: 'Install QEMU'
  env:
    sourcedir: $(Build.SourcesDirectory)/ubuntu16.04/base

- task: Docker@2
  displayName: 'Run priviledged container'
  inputs:
    command: run
    arguments: '--rm --privileged multiarch/qemu-user-static:register --reset'

- task: Docker@2
  displayName: 'build and push'
  inputs:
    # uses service connection created in step 5
    containerRegistry: 'acr-serviceconnection'
    repository: arm64v8/base
    Dockerfile: ubuntu16.04/base/Dockerfile
    tags: |
     1.0.$(Build.BuildId)-xenial
     latest

- task: AzureKeyVault@1
  displayName: 'Azure Key Vault: Get Credentials'
  inputs:
    # uses service connection created in step 5
    azureSubscription: 'arm-serviceconnection'
    KeyVaultName: arm64v8KeyVault

- task: AzureIoTEdge@2
  displayName: 'Azure IoT Edge - Generate deployment manifest'
  inputs:
    action: 'Generate deployment manifest'
    templateFilePath: deploy/deployment.template.json
    defaultPlatform: arm64v8
  env:
    CONTAINER_REGISTRY_USERNAME: $(iot-device-user)
    CONTAINER_REGISTRY_PASSWORD: $(iot-device-pwd)
    CONTAINER_REGISTRY: arm64v8cr.azurecr.io
    MODULE_NAME: arm64v8base
    IMAGE: arm64v8/base:latest

- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact: drop'
  inputs:
    PathtoPublish: '$(System.DefaultWorkingDirectory)/config/deployment.json'
```
Here is what this pipeline does:
1. First, a bash script that installs QEMU on the Host Agent and copy the static file for ARM64 to the same folder where the Dockerfile is. With this, when docker builds the image, it will find the right file to copy. 
2. The second task runs the privileged container as explained above. 
3. The third builds the image and push it to the container registry. 
4. The fourth task gets the credentials (secrets) that we added to the key vault in step 3.
5. The fifth task generates the deployment manifest. You will use it when you create the Release pipeline. It takes the deployment template that was saved in step 8, replaces the variables and save it.
6. Last task just publishes the manifest file as a build artifact.

**Step 10.** Open Pipelines, select Builds and New build pipeline. Select your source and your repo. In the Configure your pipeline step, select Existing Azure Pipelines YAML file. Set Path to /pipelines/ubuntu16.04-base-build.yml and click Continue. Review the yml file and click Run.

**Step 11.** If the build succeeded, you can check your image:
```powershell
az acr repository list --name arm64v8cr
az acr repository show-tags --name arm64v8cr --repository arm64v8/base
```
