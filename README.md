# arm64v8challenge
[![GitHub license](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](https://raw.githubusercontent.com/fegonfe/arm64v8challenge/master/LICENSE)

Build arm64v8 docker containers and release to IoT Edge using Azure DevOps

## Prerequisites
* A valid Azure subscription. If you don’t have one, you can create it for [free](https://azure.microsoft.com/en-us/free/).
* An Azure DevOps org. If you don’t have one, you can create it following the instructions [here](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization?view=azure-devops).
* A DevOps project. Create one following the instructions [here](https://docs.microsoft.com/en-us/azure/devops/organizations/projects/create-project?view=azure-devops).

## Lab 1: First things first
It all begins with an OS image. You probably are not running ARM64 on your dev environment, but you need to build a container to run on this platform. There are some arm64v8 container images available on http://hub.docker.com, but how can you build one? If you have Windows 10 or macOS, it is very straightforward because docker already does the job of running a container of a different platform.

With Linux, the story is different. You need a machine emulator such as [QEMU](http://wiki.qemu.org/) to do the job. The problem now is how to do this on a build agent. This [article](https://docs.microsoft.com/en-us/azure/devops/pipelines/languages/docker?view=azure-devops#build-arm-containers) explains the requirements: 
1. Author your Dockerfile so that a binary of QEMU (for the target platform) exists in the base Docker image.
2. Run a privileged container that register QEMU on the build agent.
Actually, you only need ton run a privileged container if you have platform dependent instructions, like RUN or ENV, on your Dockerfile.

**Step 1.** Define the variables that you are going to use through out the labs:
```bash
userId=$(az ad signed-in-user show --query objectId)
unique=${userId:1:8}
resourceGroup=arm64v8challenge
location=eastus
containerRegistry=arm64v8cr$unique
iotHub=iothub-arm64v8challenge$unique
svcPrincipalDevice=iotdevice-pull$unique
svcPrincipalPipeline=sp-arm64v8challenge$unique
keyVault=arm64v8KeyVault$unique
```
**Step 2.** Create a resource group and a private container registry:
```bash
az group create --name $resourceGroup --location $location
az acr create --resource-group $resourceGroup --name $containerRegistry --sku Basic
```
**Step 3.** Create the service connection that you are going to reference in the pipeline. In the Azure DevOps project, open **Project Settings** (last option on the left menu). Under **Pipelines**, click on **Service Connections** and create a new service connection. Select **Docker Registry** and configure using the following values:
* Registry type: Azure Container Registry
* Connection name: acr-serviceconnection
* Azure subscription: {your subscription}
* Azure container registry: {select the registry created in step 2}

**Step 4.** Create a new repo on Azure Repo (or github).

**Step 5.** Create a new folder and name it ubuntu16.04/base and set Dockerfile as the file name. Copy and Paste the following to the Dockerfile:

    ARG UBUNTU_VERSION=xenial
    FROM arm64v8/ubuntu:${UBUNTU_VERSION} as base
    # this will satisfy the requirement to have the binary inside the image
    COPY qemu-aarch64-static /usr/bin 

When you build your container image, it will pull a base image from the Docker Hub and add the static file as required.

**Step 6.** Create a new folder and name it pipelines and set ubuntu16.04-base.yml as the file name. Copy and Paste the following to the yml file:
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
  enabled: false

- task: Docker@2
  displayName: 'build and push'
  inputs:
    # uses service connection created in step 3
    containerRegistry: 'acr-serviceconnection'
    repository: arm64v8/base
    Dockerfile: ubuntu16.04/base/Dockerfile
    tags: |
     1.0.$(Build.BuildId)-xenial
     latest
```
Here is what this pipeline does:
1. First, a bash script that installs QEMU on the Host Agent and copy the static file for ARM64 to the same folder where the Dockerfile is. With this, when docker builds the image, it will find the right file to copy. 
2. Note that the second task is not enabled. It is the task that runs the privileged container as explained above. Since the Dockerfile does not have any platform dependent instructions, you don't need to enable it.
3. The third task builds the image and push it to the container registry. 

**Step 7.** Open **Pipelines**, select **Builds** and **New build pipeline**. Select your source and your repo. In the **Configure your pipeline** step, select **Existing Azure Pipelines YAML file**. Set Path to /pipelines/ubuntu16.04-base.yml and click **Continue**. Review the yml file and click **Run**.

**Step 8.** If the build succeeded, you can check your image:
```bash
az acr repository list --name $containerRegistry
az acr repository show-tags --name $containerRegistry --repository arm64v8/base
```
### Create and configure the IoT Edge device
You have built an ARM64 image and pushed it to a repository. In the next steps, you are going to create and configure the resources that will allow the IoT Edge to pull that image. For the most part, you are going to follow the same steps as in this [article](https://docs.microsoft.com/en-us/azure/iot-edge/quickstart-linux):

![Image](https://docs.microsoft.com/en-us/azure/iot-edge/media/quickstart-linux/install-edge-full.png)

**Step 1.** Create a VM that will work as your IoT Edge device. This is especially helpful during development, because you don't need the device to build the app. 
```bash
az vm image accept-terms --urn microsoft_iot_edge:iot_edge_vm_ubuntu:ubuntu_1604_edgeruntimeonly:latest
az vm create --resource-group $resourceGroup --name EdgeVM --image microsoft_iot_edge:iot_edge_vm_ubuntu:ubuntu_1604_edgeruntimeonly:latest --admin-username edgeadmin --generate-ssh-keys
```
**Step 2.** Install the Azure IoT extension
```bash
az extension add --name azure-cli-iot-ext
```
**Step 3.** Create an IoT Hub:
```bash  
az iot hub create --resource-group $resourceGroup --name $iotHub --sku F1
```  
**Step 4.** Register an IoT Edge device and get its connection string
```bash
az iot hub device-identity create -g $resourceGroup --hub-name $iotHub --device-id edgeDeviceVM --edge-enabled
connString=$(az iot hub device-identity show-connection-string -g $resourceGroup --device-id edgeDeviceVM --hub-name $iotHub --query connectionString --output tsv)
```
**Step 5.** Set the connection string on the IoT Edge device: 
```bash  
az vm run-command invoke -g $resourceGroup -n EdgeVM --command-id RunShellScript --script "/etc/iotedge/configedge.sh '$connString'"
```
**Step 6.** Connect to the VM:
```bash    
ipAddress=$(az vm show -d -g $resourceGroup -n EdgeVM --query publicIps --output tsv)
ssh edgeadmin@$ipAddress
```    
**Step 7.** Check if your IoT Edge Device is configured:
```bash
sudo systemctl status iotedge
sudo iotedge list
```
**Step 8.** Configure it to run the docker image. Run `uname -p` to check the platform. If you pull your image now, is it going to run?
You are back to the original problem. As this device is not an ARM64, you need to install QEMU on it.
```bash
sudo apt-get update
sudo apt-get install -y qemu qemu-user-static qemu-user binfmt-support
```
### Update the Build pipeline 
You have the image and the device running. In the next steps, you are going to complete the build pipeline to create a deployment manifest. A deployment manifest tells an IoT Edge device (or a group of devices) which modules to install and how to configure them.

**Step 1.** Create a service principal with AcrPull permission to be used by the device.
```bash
scope=$(az acr show --name $containerRegistry --query id --output tsv)
iotpwd=$(az ad sp create-for-rbac --name $svcPrincipalDevice --scopes $scope --role acrpull --query password --output tsv)
iotappId=$(az ad sp show --id http://$svcPrincipalDevice --query appId --output tsv)
```
**Step 2.** Create a key vault to save the credential. You don't want to save the password in plain text in the yaml file.
```bash
az keyvault create --name $keyVault --resource-group $resourceGroup --location $location
az keyvault secret set --vault-name $keyVault --name iot-device-user --value $iotappId
az keyvault secret set --vault-name $keyVault --name iot-device-pwd --value $iotpwd
```
**Step 3.** Create a new service principal with permissions to get the secrets from the key vault.
```bash
devopspwd=$(az ad sp create-for-rbac --name $svcPrincipalPipeline --skip-assignment --query password --output tsv)
devopsappId=$(az ad sp show --id http://$svcPrincipalPipeline --query appId --output tsv)
az keyvault set-policy --name $keyVault --spn http://$svcPrincipalPipeline --secret-permissions get list
```
**Step 4.** Create a new service connection. Select **Azure Resource Manager** and click on the link **use the full version of the service connection dialog**. With this, you can use the service principal that you have created in the last step. Configure it using the following values:
* Connection name: arm-serviceconnection
* Scope level: Subscription
* Service principal client ID: {fill it with the value of the $devopsappId variable}
* Service principal key: {fill it with the value of the $devopspwd variable}

**Step 5.** On your repo, create a new folder, name it deploy and set deployment.template.json as the file name. Copy and Paste the content from [this file](https://raw.githubusercontent.com/fegonfe/arm64v8challenge/master/deploy/deployment.template.json).

**Step 6.** Edit the build pipeline (ubuntu16.04-base.yml). Add the following tasks and replace the value of the **KeyVaultName** and **CONTAINER_REGISTRY** keys: 
```yml
- task: AzureKeyVault@1
  displayName: 'Azure Key Vault: Get Credentials'
  inputs:
    # uses service connection created in step 4
    azureSubscription: 'arm-serviceconnection'
    KeyVaultName: {fill it with the value of the $keyVault variable}

- task: AzureIoTEdge@2
  displayName: 'Azure IoT Edge - Generate deployment manifest'
  inputs:
    action: 'Generate deployment manifest'
    templateFilePath: deploy/deployment.template.json
    defaultPlatform: arm64v8
  env:
    CONTAINER_REGISTRY_USERNAME: $(iot-device-user)
    CONTAINER_REGISTRY_PASSWORD: $(iot-device-pwd)
    CONTAINER_REGISTRY: {fill it with the value of the $containerRegistry variable}
    MODULE_NAME: arm64v8base
    IMAGE: arm64v8/base:latest

- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact: drop'
  inputs:
    PathtoPublish: '$(System.DefaultWorkingDirectory)/config/deployment.json'
```
1. The Key Vault task gets the credentials (secrets) that you added to the key vault in step 2.
2. The Azure IoT Edge task generates the deployment manifest. You will use it when you create the Release pipeline. It takes the deployment template that was saved in step 5, replaces the variables and save it.
3. Last task just publishes the manifest file as a build artifact.

**Step 7.** On **Pipelines**, select **Builds**. Click **Queue** to start a new pipeline and then **Run**.

**Step 8.** If the build succeeded, click on **Artifacts** and select **drop**. In the **Artifacts explorer**, you can check that the deployment.json file was created.

### Deploy to the IoT Edge Device
After the deployment manifest is created, you need to create a Release pipeline to publish it to the IoT Hub.

**Step 1.** Allow the service principal to deploy a module.
```bash
scope=$(az iot hub show -g $resourceGroup --name $iotHub --query id --output tsv)
az role assignment create --role Contributor --assignee http://$svcPrincipalPipeline --scope $scope
```
**Step 2.** Create a release pipeline. In the Azure DevOps project, select **Pipelines** and open **Releases**. Click on **New** and then **New Release Pipeline**. In the **Select a template** panel, click **Empty job**. In the **Stage** panel, set **Stage name** as Test and close the panel.

**Step 3.** Change the Pipeline name to "Deploy to IoT Edge".

**Step 4.** Add an Artifact. Next to **Artifacts**, click on **Add.**.

![Image](https://docs.microsoft.com/en-us/azure/devops/pipelines/release/_img/artifacts-02.png?view=azure-devops)

Select the **Source (build pipeline)** and click on **Add**. Select the artifact trigger and make sure the **Continuous deployment trigger** is enabled.

![Image](https://docs.microsoft.com/en-us/azure/devops/pipelines/archive/apps/_shared/_img/build-artifact-trigger-in-release-definition.png?view=azure-devops)

**Step 5.** Open the **Tasks** tab and, with **Agent Job** selected, set **Agent Specification** to ubuntu-16.04. Select + and search for the **Azure IoT Edge task**. Select **Add**. Configure the task as shown below:
* Action: Deploy to IoT Edge devices
* Deployment file: Click on the ellipsis and in the **Select a file or folder**, select deployment.json
* Azure subscription: arm-serviceconnection
* IoT Hub name: {fill it with the value of the the $iotHub variable}
* Choose single/multiple device: Single Device
* IoT Edge device ID: edgeDeviceVM

Click on **Save**. Click **OK** to save and click on **Create release**. In the **Create a new release** panel, check that the artifact version you want to use is selected and choose **Create**. Choose the release link in the information bar message. For example: "Release **Release-1** has been created". In the pipeline view, choose the status link in the stages of the pipeline to see the logs and agent output.

**Step 6.** Connect to the VM:
```bash
ssh edgeadmin@$ipAddress
```

**Step 7.** Check if the module was deployed.
```bash
sudo docker images
```

**Step 8.** Start the container.
```bash
image=$(sudo docker images | grep azurecr.io/arm64v8/base | awk '{print $3}')
sudo docker run -it $image /bin/bash
```

**Step 9.** Check the platform.
```bash
uname -p
```
Congrats! You have completed the CI/CD Pipelines and you have a place to test your app on the target platform.

## Lab 2: Python
Python is one of the programming languages that is supported on ARM64, but with some limitation regarding 
packages. With the base image already created, adding Python to it becomes very simple.

**Step 1.** Create a new folder and name it ubuntu16.04/python3.5 and set Dockerfile as the file name. Copy and Paste the following to the Dockerfile:

    ARG DOCKER_REGISTRY
    FROM ${DOCKER_REGISTRY}/arm64v8/base

    RUN apt-get update \
        && apt-get install -y --no-install-recommends wget ca-certificates python3 \
        && wget -nv https://bootstrap.pypa.io/get-pip.py \
        ## installs pip using recommended method
        && python3 get-pip.py --disable-pip-version-check --no-cache-dir \
        && apt-get purge -y wget \
        && apt-get autoremove -y \
        && apt-get clean -y \
        && rm -rf /var/lib/apt/lists/* \
        && rm -f get-pip.py

**Step 2.** Create a new file under the pipelines folder and set ubuntu16.04-python3.5.yml as the file name. Copy and Paste the following to the yml file:
```yml
pool:
  name: Hosted Ubuntu 1604
steps:
- task: Docker@2
  displayName: 'Run priviledged container'
  inputs:
    command: run
    arguments: '--rm --privileged multiarch/qemu-user-static:register --reset'

- task: Docker@2
  displayName: 'build'
  inputs:
    command: build
    # uses service connection created in Lab 1
    containerRegistry: 'acr-serviceconnection'
    repository: arm64v8/python3.5
    Dockerfile: ubuntu16.04/python3.5/Dockerfile
    arguments: --build-arg DOCKER_REGISTRY=$(DOCKER_REGISTRY)
    tags: |
     1.0.$(Build.BuildId)-xenial
     latest

- task: Docker@2
  displayName: 'push'
  inputs:
    command: push
    # uses service connection created in Lab 1
    containerRegistry: 'acr-serviceconnection'
    repository: arm64v8/python3.5
    tags: |
     1.0.$(Build.BuildId)-xenial
     latest
```
Here is what this pipeline does:
1. First task runs the privileged container as explained in Lab 1. It is need now because there is a RUN command in the Dockerfile.
2. The second task builds the image using an argument to pass the container registry from where the base image will be pulled.
3. The third task pushes the image.

**Step 3.** The pipeline needs permission to pull the base image from the container registry:
```bash
scope=$(az acr show --name $containerRegistry --query id --output tsv)
spdevops=$(az role assignment list --scope $scope --role AcrPush --query "[?principalName!=''].principalName" --output tsv)
az role assignment create --role AcrPull --assignee $spdevops --scope $scope
```

**Step 4.** Open **Pipelines**, select **Builds** and **New build pipeline**. Select your source and your repo. In the **Configure your pipeline** step, select **Existing Azure Pipelines YAML file**. Set Path to /pipelines/ubuntu16.04-python3.5.yml and click **Continue**. 
After reviewing the yml file, click on **Variables** and then **New variable**. Set the name to **DOCKER_REGISTRY** and value with full name of the container registry, i.e. the value of $containerRegistry variable with ".azurecr.io" suffix. Click **OK**, **Save** and then **Run**.

**Step 5.** If the build succeeded, you can check your image:
```bash
az acr repository show-tags --name $containerRegistry --repository arm64v8/python3.5
```

### Big wheel keep on turning
The challenge for python with ARM64 is that some packages are not available in wheel format. When you run `pip install package-name`, it actually downloads the source and compile it on your device. The [Numpy](https://www.numpy.org/) library, a well-known library used for scientific computing, is one example. It would not be a problem to allow the library to be compiled during the application build, but some libs take a lot of time to build. For example, numpy takes 20 minutes to build on a hosted agent ([Standard_DS2_v2](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes-general#dsv2-series)). This is where Artifacts feeds become handy. You can publish your packages to a feed and reuse them on as many applications as you need. But, there is a caveat for ARM64: you cannot use the python feed. If you try, you will receive this error:

    The input file name 'package-name-version-none-none-linux_aarch64.whl' contains an invalid platform part: 'linux_aarch64'. See the platform section of PEP 425 (and PEP 513 for Linux distributions) for more information.

For ARM64, you will have to use a Universal feed. Since there are standard tasks to publish and download packages to/from a Universal feed, you will need to use them in your pipelines accordinly. Let's build a numpy package and publish to a feed.

**Step 1.** On Azure DevOps, select **Artifacts** and click on **New feed**. Name it **arm64** and click **Create**.

**Step 2.** Create a new folder and name it ubuntu16.04/python3.5/numpy and set Dockerfile as the file name. Copy and Paste the following to the Dockerfile:

    ARG DOCKER_REGISTRY
    FROM ${DOCKER_REGISTRY}/arm64v8/python3.5
    
    RUN apt-get update \
      && apt-get install -y --no-install-recommends build-essential libatlas-base-dev \
      && pip3 wheel --no-deps --wheel-dir=./dist numpy==1.16.4

**Step 3.** Create a new file under the pipelines folder and set ubuntu16.04-python3.5-numpy.yml as the file name. Copy and Paste the following to the yml file:
```yml
jobs:
- job: Build
  pool:
    vmImage: 'ubuntu-16.04'
  steps:
  - task: Docker@2
    displayName: 'Run priviledged container'
    inputs:
      command: run
      arguments: '--rm --privileged multiarch/qemu-user-static:register --reset'

  - task: Docker@2
    displayName: build
    inputs:
      containerRegistry: 'acr-serviceconnection'
      command: build
      Dockerfile: ubuntu16.04/python3.5/numpy/Dockerfile
      arguments: '--build-arg DOCKER_REGISTRY=$(DOCKER_REGISTRY)'
      tags: arm64v8/python3.5-numpy
      addPipelineData: false

  - task: Bash@3
    inputs:
      targetType: 'inline'
      script: |
        docker create -ti --name numpy arm64v8/python3.5-numpy bash
        docker cp numpy:/dist $(Build.ArtifactStagingDirectory)/dist
        docker rm -fv numpy

  - task: UniversalPackages@0
    inputs:
      command: 'publish'
      publishDirectory: '$(Build.ArtifactStagingDirectory)/dist'
      feedsToUsePublish: 'internal'
      vstsFeedPublish: 'arm64'
      versionOption: 'custom'
      versionPublish: '1.16.4'
      vstsFeedPackagePublish: 'numpy'
      packagePublishDescription: 'numpy built with arm64v8/python:3.5'
```
Here is what this pipeline does:
1. First task runs the privileged container as explained in Lab 1. It is need now because there is a RUN command in the Dockerfile.
2. The second task builds the image using an argument to pass the container registry from where the base image will be pulled.
3. The third task creates a container just to copy the wheel file. Note that you don't to save this image to a repository.
4. Last task publishes the package to the feed.