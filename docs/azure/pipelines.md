---
title: Azure DevOps Pipelines
parent: Azure
has_children: false
nav_order: 2
---

## Azure DevOps Pipelines: Automated build and deploy
{: .no_toc }

When you check-in the code (or merge a pull request) of your ASP.NET WebApp or Api, it would be great when everything is build, tested and deployed to a test or staging site.

Azure Pipelines are a great service to do this. Combined with **Deployment slots** you can even swap the running staging site with your production site to have the new version instantly on-line. And when something goes wrong, just swap back to the old production site (well... on condition that you didn't do a non-backward-compatible migration of your database or something like that).

There is an awfull lot of documentation about Azure Pipelines, because it can be used for many different techniques in many different ways, so it's easy to get lost when you start. Also, the tools to help you with the setup work fine with a simple example solution, but when your solution is a little more complicated they might fail. So I will make these presumption (and add some links to help you find your way when your solution is different):

* A solution with an ASP.NET Core WebApp and/or Api and one or more .NET Core or .NET Standard class libraries.
* NuGet packages with their PackageReferences in the .csproj files (not in package.json).
* An Azure DevOps Git repository.
* Publish to a deployment slot of an Azure App Service.
* Manual swap to production slot.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Configure your App Service in the Azure Portal

* To be able to create **deployment slots**, your **Azure App Service Plan** must be at least in the S1 pricing tier. See [Create or use an Azure App Service Plan](appservice#create-or-use-an-azure-app-service-plan){:target="_blank"}
* If you do not have an App Service yet for your WebApp or Api, see [Create the App Service (Web App) for the WebApi or WebApp](appservice#create-the-app-service-web-app-for-the-webapi-or-webapp){:target="_blank"}

### Add a deployment slot

- Select your App Service in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **App services**)
- In the left pane, select **Deployment slots** > **Add Slot**.
- Set the name of the slot (e.g. Staging). At **Clone settings from:** select your production/parent App Service to copy its settings. Click **Add**.
- After creation click **Close**.
- Click the newly created deployment slot name. Your slot is now shown as an **App Service**. The settings in the **Configuration** tab are copied from your parent App Service, but other settings not. Check for correct values.
- If you use the **Key Vault**, make sure to turn on **Managed identities** for Azure resources: in the **Identity** section turn on **Status** in the **System assigned** tab and select Save. You also must enable the slot to use the KeyVault with its Managed Identity, see: [Enable the WebApi/WebApp to use the KeyVault with its Managed Identity](appservice#enable-the-webapiwebapp-to-use-the-keyvault-with-its-managed-identity){:target="_blank"}.
- If you use AD B2C authentication, make sure to add the URL of the deployment slot to the callback urls of your AD B2C directory. See: TODO.

### Create an Azure Service Connection

To deploy your app, your Azure DevOps Pipeline must have access to your Azure App Service. This is possible with an Azure Service Connection. I'll describe here the simple approach to create one. For more advanced scenarios see: [Connect to Microsoft Azure](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure?view=azure-devops){:target="_blank"}.

- Open your Azure DevOps project.
- Select **Project Settings** at the bottom - left.
- Open the **Service connections** page.
- Choose **+ New service connection** (or Create service connection) and select **Azure Resource Manager**.
- Select **Service Principal Authentication** (default).
- Connection name: You can, for instace, use the name of the resource group of your App Service or the name of your Azure subscription.
- Scope level: Subscription (default).
- Subscription: Select your Azure subscription.
- Resource Group: Select the Resource group of your App Service or leave blank for full subscription access.
- Select **OK** to create the connection.

## Create the DevOps Pipeline

A pipeline is defined in a YAML file called azure-pipelines.yml in the root of your repository. Azure DevOps can help you to create the YAML file (see: [Create your first pipeline](https://docs.microsoft.com/en-us/azure/devops/pipelines/create-first-pipeline?view=azure-devops){:target="_blank"}), but the result did not make me happy.

You can create an **azure-pipelines.yml** file like this in the root of your repository:

```yaml
trigger:
  - master # Branch to trigger the build

pool:
  vmImage: 'windows-latest'

variables:
  solution: '**/*.sln'
  buildConfiguration: 'Release'
  azureServiceConnection: 'connectionYouCreated'
  azureAppServiceName: 'yourAppService'
  deployToSlot: 'yourSlotName'
  packageName: 'WebApp.zip' # Use the FOLDER name of the project (.csproj) to publish.

steps:
- task: DotNetCoreCLI@2
  displayName: Build
  inputs:
    command: build
    projects: '**/*.csproj'
    arguments: '--configuration $(buildConfiguration)'

- task: DotNetCoreCLI@2
  inputs:
    command: publish
    publishWebProjects: True
    arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
    zipAfterPublish: True

- task: AzureWebApp@1
  displayName: Azure Deploy to deployment slot
  inputs:
    azureSubscription: '$(azureServiceConnection)'
    appName: '$(azureAppServiceName)'
    slotName: '$(deployToSlot)'
    package: $(System.ArtifactsDirectory)/**/$(packageName)
    deploymentMethod: auto
```

Commit the created azure-pipelines.yml file and sync with your Azure DevOps Git repository.

DevOps Pipelines will recognize the file, add it as a pipeline and trigger the build the next time you sync (or merge a pull request) with your repository.

* See: [YAML predefined variables in Azure pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/variables?view=azure-devops&tabs=yaml){:target="_blank"}
* See: [Microsoft-hosted agent pools (like **windows-latest**)](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/hosted?view=azure-devops){:target="_blank"}
* See: [YAML for DotNetCoreCLI@2 task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/build/dotnet-core-cli?view=azure-devops){:target="_blank"}
* See: [YAML for AzureWebApp@1 task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-rm-web-app?view=azure-devops){:target="_blank"}
* If you use **private NuGet packages** or when your NuGet packages are specified in **package.json**, see: [Restore dependencies](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/dotnet-core?view=azure-devops#restore-dependencies){:target="_blank"}
* More info, see: [Build, test, and deploy .NET Core apps](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/dotnet-core?view=azure-devops){:target="_blank"}

## Swap Staging to Production

- Select your App Service in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **App services**)
- Select the **Deployment slots** page.
- Select **Swap** at the top of the page.
- Select the source (e.g. Staging) and target (e.g. Production) slot and verify the **Config Changes**.
- Press **Swap** to start the swapping.
- When the swap is finished, the content of the Staging slot is now on the Production site, and the old content of the Production slot is on the Staging site.
- When something is wrong, its easy to swap back the old Production version: just do the swap again.
- You might want to add **two** extra deployment slots: One to deploy your automated build to, and one for staging/testing. Then you can safely swap your staging slot to production when testing is ready. The old production version will be in the staging slot and there is no risk for accidentally overwriting it with an automated build just when you want to swap back the old production version because of an unexpected issue...

## References

* [Set up staging environments in Azure App Service](https://docs.microsoft.com/en-us/azure/app-service/deploy-staging-slots){:target="_blank"}
* [Build, test, and deploy .NET Core apps](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/dotnet-core?view=azure-devops){:target="_blank"}


