# arm64v8challenge
[![GitHub license](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](https://raw.githubusercontent.com/fegonfe/arm64v8challenge/master/LICENSE)

Build arm64v8 docker containers and release to IoT Edge using Azure DevOps

## First things first
It all begins with an OS image. You probably are not running ARM64 on your dev environment, but you need to build a container to run on this platform. There are some arm64v8 container images on http://hub.docker.com, but how can you build them? If you have Windows 10 or macOS, it is very straightforward because docker already does the job of running a container of a different platform.

Linux has been doing this forever using [QEMU](http://wiki.qemu.org/), but how to do this on a build agent? This [article](https://docs.microsoft.com/en-us/azure/devops/pipelines/languages/docker?view=azure-devops#build-arm-containers) explains the requirements: 
1. Author your Dockerfile so that a binary of QEMU (for the target platform) exists in the base Docker image
2. Run a privileged container that register QEMU on the build agent.

**Step 1.** Create a new repo on Azure Repo (or github).

**Step 2.** Create a new folder and name it ubuntu16.04/base and set Dockerfile as the file name. Start the Dockerfile with the following content for now:

    ARG UBUNTU_VERSION=xenial
    FROM arm64v8/ubuntu:${UBUNTU_VERSION} as base

**Step 3.** Create a new folder and name it pipelines and set ubuntu16.04-base-build.yml as the file name. Copy and Paste the following to the yml file:
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
    containerRegistry: 'acr-serviceconnection'
    repository: arm64v8/base
    Dockerfile: ubuntu16.04/base/Dockerfile
    tags: |
     1.0.$(Build.BuildId)-xenial
     latest
```
First task installs QEMU on the Host Agent and copy the static file for ARM64 to the same folder where the Dockerfile is. Second task runs the privileged container as explained above. Last task builds the image and push it to a container registry. You will define the container registry in the next steps.

**Step 4.** Create a resource group and a private container registry:
```sh
az group create --name arm64v8challenge --location eastus
az acr create --resource-group arm64v8challenge --name arm64v8cr --sku Basic
```
**Step 5.** Create a service principal with AcrPush permission:
```sh
az ad sp create-for-rbac --name azuredevops-push --scopes $(az acr show --name myContainerRegistryARM --query id --output tsv) --role acrpush
```
**Step 6.** Now on Azure DevOps, open Project Settings (last option on left-menu). Under Pipelines, click on Service Connections and create a new service connection. Select Docker Registry and configure using the following:
* Registry type: Azure Container Registry
* Connection name: acr-serviceconnection  `# this links to the name used in Step 3`
* Azure subscription: your subscription
* Azure container registry: arm64v8cr

**Step 7.** Add the following line to the Dockerfile. This will satisfy the requirement to have the binary inside the image: 

    COPY qemu-aarch64-static /usr/bin

**Step 8.** Open Pipelines, select Builds and New build pipeline. Select your source and your repo. In the Configure your pipeline step, select Existing Azure Pipelines YAML file. Set Path to /pipelines/ubuntu16.04-base-build.yml and click Continue. Review yml file and click Run.

**Step 9.** If the build succeeded, you can check your image:
```sh
az acr repository list --name arm64v8cr
az acr repository show-tags --name arm64v8cr --repository arm64v8/base
```
