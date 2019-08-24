# arm64v8challenge
[![GitHub license](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](https://raw.githubusercontent.com/fegonfe/arm64v8challenge/master/LICENSE)

Build arm64v8 docker containers and release to IoT Edge using Azure DevOps

## Prerequisites
* A valid Azure subscription. If you don’t have one, you can create it for [free](https://azure.microsoft.com/en-us/free/).
* An Azure DevOps org. If you don’t have one, you can create it following the instructions [here](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/create-organization?view=azure-devops).
* A DevOps project. Create one following the instructions [here](https://docs.microsoft.com/en-us/azure/devops/organizations/projects/create-project?view=azure-devops).

## First things first
It all begins with an OS image. You probably are not running ARM64 on your dev environment, but you need to build a container to run on this platform. There are some arm64v8 container images available on http://hub.docker.com, but how can you build one? If you have Windows 10 or macOS, it is very straightforward because docker already does the job of running a container of a different platform.

With Linux, the story is different. You need a machine emulator such as [QEMU](http://wiki.qemu.org/) to do the job. The problem now is how to do this on a build agent. This [article](https://docs.microsoft.com/en-us/azure/devops/pipelines/languages/docker?view=azure-devops#build-arm-containers) explains the requirements: 
1. Author your Dockerfile so that a binary of QEMU (for the target platform) exists in the base Docker image.
2. Run a privileged container that register QEMU on the build agent.
Actually, you only need ton run a privileged container if you have platform dependent instructions, like RUN or ENV, on your Dockerfile.

**Step 1.** Create a resource group and a private container registry:
```powershell
az group create --name arm64v8challenge --location eastus
az acr create --resource-group arm64v8challenge --name arm64v8cr --sku Basic
```
**Step 2.** Create the service connection that you are going to reference in the pipeline. In the Azure DevOps project, open **Project Settings** (last option on the left menu). Under **Pipelines**, click on **Service Connections** and create a new service connection. Select **Docker Registry** and configure using the following values:
* Registry type: Azure Container Registry
* Connection name: acr-serviceconnection
* Azure subscription: {your subscription}
* Azure container registry: arm64v8cr

**Step 3.** Create a new repo on Azure Repo (or github).

**Step 4.** Create a new folder and name it ubuntu16.04/base and set Dockerfile as the file name. Copy and Paste the following to the Dockerfile:

    ARG UBUNTU_VERSION=xenial
    FROM arm64v8/ubuntu:${UBUNTU_VERSION} as base
    # this will satisfy the requirement to have the binary inside the image
    COPY qemu-aarch64-static /usr/bin 

**Step 5.** Create a new folder and name it pipelines and set ubuntu16.04-base-build.yml as the file name. Copy and Paste the following to the yml file:
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
    # uses service connection created in step 2
    containerRegistry: 'acr-serviceconnection'
    repository: arm64v8/base
    Dockerfile: ubuntu16.04/base/Dockerfile
    tags: |
     1.0.$(Build.BuildId)-xenial
     latest
```
Here is what this pipeline does:
1. First, a bash script that installs QEMU on the Host Agent and copy the static file for ARM64 to the same folder where the Dockerfile is. With this, when docker builds the image, it will find the right file to copy. 
2. Note that the second task is not enabled. It is the task that runs the privileged container as explained above. Since the Dockerfile does not have any platform dependent instructions, you don't it to enabled it.
3. The third task builds the image and push it to the container registry. 

**Step 6.** Open **Pipelines**, select **Builds** and **New build pipeline**. Select your source and your repo. In the **Configure your pipeline** step, select **Existing Azure Pipelines YAML file**. Set Path to /pipelines/ubuntu16.04-base-build.yml and click **Continue**. Review the yml file and click **Run**.

**Step 7.** If the build succeeded, you can check your image:
```powershell
az acr repository list --name arm64v8cr
az acr repository show-tags --name arm64v8cr --repository arm64v8/base
```
## Create and configure the IoT Edge device
You have built an ARM64 image and pushed it to a repository. In the next steps, you are going to create and configure the resources that will allow the IoT Edge to pull that image. For the most part, you are going to follow the same steps as in this [article](https://docs.microsoft.com/en-us/azure/iot-edge/quickstart-linux):

![Image](https://docs.microsoft.com/en-us/azure/iot-edge/media/quickstart-linux/install-edge-full.png)

**Step 1.** Create a VM that will work as your IoT Edge device. This is specially helpful during development, because you don't need the device to build the app. 
```powershell
az vm image accept-terms --urn microsoft_iot_edge:iot_edge_vm_ubuntu:ubuntu_1604_edgeruntimeonly:latest
az vm create --resource-group arm64v8challenge --name EdgeVM --image microsoft_iot_edge:iot_edge_vm_ubuntu:ubuntu_1604_edgeruntimeonly:latest --admin-username edgeadmin --generate-ssh-keys
```
**Step 2.** Create an IoT Hub:
```powershell  
az iot hub create --resource-group arm64v8challenge --name iothub-arm64v8challenge --sku F1
```  
**Step 3.** Register an IoT Edge device and get its connection string
```powershell
az iot hub device-identity create --hub-name iothub-arm64v8challenge --device-id edgeDeviceVM --edge-enabled
$connString = $(az iot hub device-identity show-connection-string -g arm64v8challenge --device-id edgeDeviceVM --hub-name iothub-arm64v8challenge --query connectionString -o tsv)
```
**Step 4.** Set the connection string on the IoT Edge device: 
```powershell  
az vm run-command invoke -g arm64v8challenge -n EdgeVM --command-id RunShellScript --script "/etc/iotedge/configedge.sh '$connString'"
```  

**Step 5.** Connect to the VM:
```powershell    
$ipAddress  = $(az vm show -d -g arm64v8challenge -n EdgeVM --query publicIps)
ssh edgeadmin@$ipAddress
```    
**Step 6.** Check if your IoT Edge Device is configured:
```bash
sudo systemctl status iotedge
sudo iotedge list
```
**Step 7.** Configure it to run the docker image. Run `uname -p` to check the platform. If you pull your image now, is it going to run?
You are back to the original problem. As this device is not an ARM64, you need to install QEMU on it.
```bash
sudo apt-get update
sudo apt-get install -y qemu qemu-user-static qemu-user binfmt-support
```
## Update the Build pipeline 
You have the image and the device running. In the next steps, you are going to complete the build pipeline to create a deployment manifest. A deployment manifest tells an IoT Edge device (or a group of devices) which modules to install and how to configure them.

**Step 1.** Create a service principal with AcrPull permission to be used by the device.
```powershell
$scope = $(az acr show --name arm64v8cr --query id --output tsv)
$iotpwd=$(az ad sp create-for-rbac --name iotdevice-pull --scopes $scope --role acrpull --query password --output tsv)
$iotappId=$(az ad sp show --id http://iotdevice-pull --query appId)
```
**Step 2.** Create a key vault to save the credential. You don't want to save the password in plain text in the yaml file.
```powershell
az keyvault create --name arm64v8KeyVault --resource-group arm64v8challenge --location eastus
az keyvault secret set --vault-name arm64v8KeyVault --name iot-device-user --value $iotappId
az keyvault secret set --vault-name arm64v8KeyVault --name iot-device-pwd --value $iotpwd
```
**Step 3.** Create a new service principal with permissions to get the secrets from the key vault.
```powershell
$devopspwd=$(az ad sp create-for-rbac --name sp-arm64v8challenge --skip-assignment --query password --output tsv)
$devopsappId=$(az ad sp show --id http://sp-arm64v8challenge --query appId)
az keyvault set-policy --name arm64v8KeyVault --spn http://sp-arm64v8challenge --secret-permissions get list
```
**Step 4.** Create a new service connection. Select **Azure Resource Manager** and click on the link **use the full version of the service connection dialog**. With this, you can use the service principal that you have created in the last step. Configure it using the following values:
* Connection name: arm-serviceconnection
* Scope level: Subscription
* Service principal client ID: {fill it with the value of $devopsappId variable}
* Service principal key: {fill it with the value of $devopspwd variable}

**Step 5.** On your repo, create a new folder, name it deploy and set deployment.template.json as the file name. Copy and Paste the content from [this file](https://raw.githubusercontent.com/fegonfe/arm64v8challenge/master/deploy/deployment.template.json).

**Step 6.** Edit the build pipeline (ubuntu16.04-base-build.yml) and add the following tasks:
```yml
- task: AzureKeyVault@1
  displayName: 'Azure Key Vault: Get Credentials'
  inputs:
    # uses service connection created in step 4
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
    CONTAINER_REGISTRY: arm64v8cr
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

## Deploy to the IoT Edge Device
After the deployment manifest is created, you need to create a Release pipeline to publish it to the IoT Hub.

**Step 1.** Allow the service principal to deploy a module.
```powershell
$scope = $(az iot hub show --name iothub-arm64v8challenge --query id --output tsv)
az role assignment create --role Contributor --assignee http://sp-arm64v8challenge --scope $scope
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
* IoT Hub name: iothub-arm64v8challenge
* Choose single/multiple device: Single Device
* IoT Edge device ID: edgeDeviceVM

Click on **Save**. Select Folder **\pipelines**, save and click on **Create release**. In the **Create a new release** panel, check that the artifact version you want to use is selected and choose **Create**. Choose the release link in the information bar message. For example: "Release **Release-1** has been created". In the pipeline view, choose the status link in the stages of the pipeline to see the logs and agent output.

**Step 6.** Connect to the VM:
```powershell
ssh edgeadmin@$ipAddress
```

**Step 7.** Check if the module was deployed.
```bash
sudo docker images
```

**Step 8.** Start the container.
```bash
sudo docker run -it arm64v8cr.azurecr.io/arm64v8/base /bin/bash
```

**Step 9.** Check the platform.
```bash
uname -p
```
Congrats! You have completed the CI/CD Pipelines and you have a place to test your app on the target platform.
