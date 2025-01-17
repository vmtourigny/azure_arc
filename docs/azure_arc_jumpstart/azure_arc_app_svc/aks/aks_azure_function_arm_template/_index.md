---
type: docs
title: "Azure Function ARM Template"
linkTitle: "Azure Function ARM Template"
weight: 2
description: >
---

## Deploy Azure Function application on AKS using an ARM Template

The following Jumpstart scenario will guide you on how to deploy a "Ready to Go" environment so you can start using [Azure Arc-enabled app services](https://docs.microsoft.com/azure/app-service/overview-arc-integration) deployed on [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/azure/aks/intro-kubernetes) cluster using [Azure ARM Template](https://docs.microsoft.com/azure/azure-resource-manager/templates/overview).

By the end of this guide, you will have an AKS cluster deployed with an App Service plan, a sample Azure Function application that sends messages to an Azure storage account queue and a Microsoft Windows Server 2022 (Datacenter) Azure VM, installed & pre-configured with all the required tools needed to work with Azure Arc-enabled app services.

> **NOTE: Currently, Azure Arc-enabled app services is in preview.**

## Prerequisites

- Clone the Azure Arc Jumpstart repository

    ```shell
    git clone https://github.com/microsoft/azure_arc.git
    ```

- [Install or update Azure CLI to version 2.25.0 and above](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest). Use the below command to check your current installed version.

  ```shell
  az --version
  ```

- [Generate SSH Key](https://docs.microsoft.com/azure/virtual-machines/linux/create-ssh-keys-detailed) (or use existing ssh key).

- Create Azure service principal (SP). To deploy this scenario, an Azure service principal assigned with multiple RBAC roles is required:

  - "Contributor" - Required for provisioning Azure resources
  - "Security admin" - Required for installing Cloud Defender Azure-Arc enabled Kubernetes extension and dismiss alerts
  - "Security reader" - Required for being able to view Azure-Arc enabled Kubernetes Cloud Defender extension findings

    To create it login to your Azure account run the below command (this can also be done in [Azure Cloud Shell](https://shell.azure.com/).

    ```shell
    az login
    subscriptionId=$(az account show --query id --output tsv)
    az ad sp create-for-rbac -n "<Unique SP Name>" --role "Contributor" --scopes /subscriptions/$subscriptionId
    az ad sp create-for-rbac -n "<Unique SP Name>" --role "Security admin" --scopes /subscriptions/$subscriptionId
    az ad sp create-for-rbac -n "<Unique SP Name>" --role "Security reader" --scopes /subscriptions/$subscriptionId
    ```

    For example:

    ```shell
    az login
    subscriptionId=$(az account show --query id --output tsv)
    az ad sp create-for-rbac -n "JumpstartArcAppSvc" --role "Contributor" --scopes /subscriptions/$subscriptionId
    az ad sp create-for-rbac -n "JumpstartArcAppSvc" --role "Security admin" --scopes /subscriptions/$subscriptionId
    az ad sp create-for-rbac -n "JumpstartArcAppSvc" --role "Security reader" --scopes /subscriptions/$subscriptionId
    ```

    Output should look like this:

    ```json
    {
    "appId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "displayName": "JumpstartArcAppSvc",
    "password": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "tenant": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    }
    ```

    > **NOTE: If you create multiple subsequent role assignments on the same service principal, your client secret (password) will be destroyed and recreated each time. Therefore, make sure you grab the correct password**.

    > **NOTE: The Jumpstart scenarios are designed with as much ease of use in-mind and adhering to security-related best practices whenever possible. It is optional but highly recommended to scope the service principal to a specific [Azure subscription and resource group](https://docs.microsoft.com/cli/azure/ad/sp?view=azure-cli-latest) as well considering using a [less privileged service principal account](https://docs.microsoft.com/azure/role-based-access-control/best-practices)**

## Automation Flow

For you to get familiar with the automation and deployment flow, below is an explanation.

- User is editing the ARM template parameters file (1-time edit). These parameters values are being used throughout the deployment.

- Main [_azuredeploy_ ARM template](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/azuredeploy.json) will initiate the deployment of the linked ARM templates:

  - [_VNET_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/VNET.json) - Deploys a Virtual Network with a single subnet to be used by the Client virtual machine.
  - [_aks_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/aks.json) - Deploys the AKS cluster where all the Azure Arc app services will be deployed.
  - [_clientVm_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/clientVm.json) - Deploys the client Windows VM. This is where all user interactions with the environment are made from.
  - [_logAnalytics_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/logAnalytics.json) - Deploys Azure Log Analytics workspace to support Azure Arc-enabled app services logs uploads.

- User remotes into client Windows VM, which automatically kicks off the [_AppServicesLogonScript_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/artifacts/AppServicesLogonScript.ps1) PowerShell script that deploy the AKS cluster and will configure Azure Arc-enabled app services Kubernetes environment on the AKS cluster.

    > **NOTE: Notice the AKS cluster will be deployed via the PowerShell script automation.**

## Deployment

As mentioned, this deployment will leverage ARM templates. You will deploy a single template that will initiate the entire automation for this scenario.

- The deployment is using the ARM template parameters file. Before initiating the deployment, edit the [_azuredeploy.parameters.json_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/azuredeploy.parameters.json) file located in your local cloned repository folder. An example parameters file is located [here](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/artifacts/azuredeploy.parameters.example.json).

  - _`sshRSAPublicKey`_ - Your SSH public key
  - _`spnClientId`_ - Your Azure service principal id
  - _`spnClientSecret`_ - Your Azure service principal secret
  - _`spnTenantId`_ - Your Azure tenant id
  - _`windowsAdminUsername`_ - Client Windows VM Administrator name
  - _`windowsAdminPassword`_ - Client Windows VM Password. Password must have 3 of the following: 1 lower case character, 1 upper case character, 1 number, and 1 special character. The value must be between 12 and 123 characters long.
  - _`myIpAddress`_ - Your local public IP address. This is used to allow remote RDP and SSH connections to the client Windows VM and AKS cluster.
  - _`logAnalyticsWorkspaceName`_ - Unique name for the deployment log analytics workspace.
  - _`deployAppService`_ - Boolean that sets whether or not to deploy App Service plan and a Web App. For this scenario, we leave it set to _**false**_.
  - _`deployFunction`_ - Boolean that sets whether or not to deploy App Service plan and an Azure Function application. For this scenario, we leave it set to _**true**_.
  - _`deployAPIMgmt`_ - Boolean that sets whether or not to deploy a self-hosted Azure API Management gateway.  For this scenario, we leave it set to _**false**_.
  - _`deployLogicApp`_ - Boolean that sets whether or not to deploy App Service plan and an Azure Logic App. For this scenario, we leave it set to _**false**_.
  - _`templateBaseUrl`_ - GitHub URL to the deployment template - filled in by default to point to [Microsoft/Azure Arc](https://github.com/microsoft/azure_arc) repository, but you can point this to your forked repo as well.
  - _`adminEmail`_ - an email address that will be used on the Azure API Management deployment to receive all system notifications.

- To deploy the ARM template, navigate to the local cloned [deployment folder](https://github.com/microsoft/azure_arc/tree/main/azure_arc_app_services_jumpstart/aks/arm_template) and run the below command:

    ```shell
    az group create --name <Name of the Azure resource group> --location <Azure Region>
    az deployment group create \
    --resource-group <Name of the Azure resource group> \
    --name <The name of this deployment> \
    --template-uri https://raw.githubusercontent.com/microsoft/azure_arc/main/azure_arc_app_services_jumpstart/aks/ARM/azuredeploy.json \
    --parameters <The *azuredeploy.parameters.json* parameters file location>
    ```

    > **NOTE: Make sure that you are using the same Azure resource group name as the one you've just used in the `azuredeploy.parameters.json` file**

    For example:

    ```shell
    az group create --name Arc-AppSvc-Demo --location "East US"
    az deployment group create \
    --resource-group Arc-AppSvc-Demo \
    --name arcappsvc \
    --template-uri https://raw.githubusercontent.com/microsoft/azure_arc/main/azure_arc_app_services_jumpstart/aks/ARM/azuredeploy.json \
    --parameters azuredeploy.parameters.json
    ```

    > **NOTE: The deployment time for this scenario can take ~10-20min**

    > **NOTE: Since Azure Arc-enabled app services is [currently in preview](https://docs.microsoft.com/azure/app-service/overview-arc-integration#public-preview-limitations), deployment regions availability is limited to East US and West Europe.**

- Once Azure resources has been provisioned, you will be able to see it in Azure portal.

    ![Screenshot showing the ARM template deployment completed](./01.png)

    ![Screenshot showing the new Azure resource group with all resources](./02.png)

## Windows Login & Post Deployment

- Now that first phase of the automation is completed, it is time to RDP to the client VM using it's public IP.

    ![Client VM public IP](./03.png)

- At first login, as mentioned in the "Automation Flow" section above, the [_AppServicesLogonScript_](https://github.com/microsoft/azure_arc/blob/main/azure_arc_app_services_jumpstart/aks/ARM/artifacts/AppServicesLogonScript.ps1) PowerShell logon script will start it's run.

- Let the script to run its course and **do not close** the PowerShell session, this will be done for you once completed. Once the script will finish it's run, the logon script PowerShell session will be closed, the Windows wallpaper will change and the Azure Function application will be deployed on the cluster and be ready to use.

    > **NOTE: As you will notices from the screenshots below, during the Azure Arc-enabled app services environment, the _log-processor_ service pods will be restarted and will go through multiple Kubernetes pod lifecycle stages. This is normal and can safely be ignored. To learn more about the various Azure Arc-enabled app services Kubernetes components, visit the official [Azure Docs page](https://docs.microsoft.com/azure/app-service/overview-arc-integration#pods-created-by-the-app-service-extension).**

    ![Screenshot showing PowerShell logon script run](./04.png)

    ![Screenshot showing PowerShell logon script run](./05.png)

    ![Screenshot showing PowerShell logon script run](./06.png)

    ![Screenshot showing PowerShell logon script run](./07.png)

    ![Screenshot showing PowerShell logon script run](./08.png)

    ![Screenshot showing PowerShell logon script run](./09.png)

    ![Screenshot showing PowerShell logon script run](./10.png)

    ![Screenshot showing PowerShell logon script run](./11.png)

    ![Screenshot showing PowerShell logon script run](./12.png)

    ![Screenshot showing PowerShell logon script run](./13.png)

    ![Screenshot showing PowerShell logon script run](./14.png)

    ![Screenshot showing PowerShell logon script run](./15.png)

    ![Screenshot showing PowerShell logon script run](./16.png)

    ![Screenshot showing PowerShell logon script run](./17.png)

    ![Screenshot showing PowerShell logon script run](./18.png)

    ![Screenshot showing PowerShell logon script run](./19.png)

  Once the script finishes it's run, the logon script PowerShell session will be closed, the Windows wallpaper will change, and both the app service plan and the Azure Function application deployed on the cluster will be ready.

    ![Screenshot showing desktop wallpaper change](./20.png)

- Since this scenario is deploying both the app service plan and a sample Azure Function application, you will also notice additional, newly deployed Azure resources in the resources group. The important ones to notice are:

  - **Azure Arc-enabled Kubernetes cluster** - Azure Arc-enabled app services are using this resource to deploy the app services [cluster extension](https://docs.microsoft.com/azure/azure-arc/kubernetes/conceptual-extensions), as well as using Azure Arc [Custom locations](https://docs.microsoft.com/azure/azure-arc/kubernetes/conceptual-custom-locations).

  - **Custom location** - Provides a way for tenant administrators to use their Azure Arc-enabled Kubernetes clusters as a target location for deploying Azure services.

  - [**App Service Kubernetes Environment**](https://docs.microsoft.com/azure/app-service/overview-arc-integration#app-service-kubernetes-environment) - The App Service Kubernetes environment resource is required before apps may be created. It enables configuration common to apps in the custom location, such as the default DNS suffix.

  - [**App Service plan**](https://docs.microsoft.com/azure/app-service/overview-hosting-plans) - In App Service (Web Apps, API Apps, or Mobile Apps), an app always runs in an App Service plan. In addition, Azure Functions also has the option of running in an App Service plan. An App Service plan defines a set of compute resources for an Azure Function to run.

  - [**Azure Function**](https://docs.microsoft.com/azure/azure-functions/functions-overview) - Azure Functions is a serverless solution that allows you to write less code, maintain less infrastructure, and save on costs.

  - [Application Insights](https://docs.microsoft.com/azure/azure-monitor/app/app-insights-overview) - Application Insights, a feature of Azure Monitor, is an extensible Application Performance Management (APM) service for developers and DevOps professionals. Use it to monitor your live applications.

  - Azure Storage Account - The storage account deployed in this scenario is used for hosting the [queue storage](https://docs.microsoft.com/azure/storage/queues/storage-queues-introduction) where the Azure Function will be sending messages to that can be leveraged later in an application event-driven architecture.

  ![Screenshot showing additional Azure resources in the resource group](./21.png)

- In this scenario, **a sample Jumpstart Azure Function application** was deployed. To open the deployed Function application in your web browser, simply click the Azure Function resource and the created URL or the Browse button.

  ![Screenshot showing the Azure Function URL](./22.png)

  ![Screenshot showing the Azure Function open in a web browser](./23.png)

- To demonstrate the messaging queuing element and to show how messages are stored in the queue storage, the Azure Function deployment script also generates 10 sample messages. To view it, click on the newly created storage account and go to the "Queues" section where you will see the new queue and the stored messages.

  ![Screenshot showing the Azure storage account](./24.png)

  ![Screenshot showing the Azure storage queue](./25.png)

  ![Screenshot showing Azure Function messages in storage queue](./26.png)

- Alternatively, you can view the same queue storage using the Azure Storage Explorer client application installed automatically in the Client VM or using the Azure Storage Browser portal-based view.

  ![Screenshot showing Azure Storage Explorer client application storage queue](./27.png)

  ![Screenshot showing Azure Storage Explorer portal-based view](./28.png)

  ![Screenshot showing Azure Storage Explorer portal-based view storage queue](./29.png)

- To generate your own messages using the Function application, use the Function invoke URL. As part of the deployment script, a _`funcUrl.txt`_ text file located in the Client VM under _C:\Temp_ folder that includes invoke URL was created for you. Copy the URL and open it in your web browser while adding the message text to it using the _`?name=<Something>`_ syntax, for example, _`?name=Bilbo`_.

  ![Screenshot showing the funcUrl.txt file](./30.png)

  ![Screenshot showing invoke URL](./31.png)

  ![Screenshot showing invoke URL in web browser](./32.png)

- Go back to the storage queue and see the new added message.

  ![Screenshot showing the new message in the storage queue](./33.png)

## Cluster extensions

In this scenario, the Azure Arc-enabled app services cluster extension was deployed and used throughout this scenario in order to deploy the app services infrastructure.

- In order to view cluster extensions, click on the Azure Arc-enabled Kubernetes resource Extensions settings.

  ![Screenshot showing the Azure Arc-enabled Kubernetes resource](./34.png)

  ![Screenshot showing Azure Arc-enabled Kubernetes cluster extensions settings](./35.png)

## Cleanup

- If you want to delete the entire environment, simply delete the deployed resource group from the Azure portal.

  ![Screenshot showing the Delete Azure resource group button](./36.png)
