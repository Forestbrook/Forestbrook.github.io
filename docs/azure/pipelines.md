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

* A solution with an ASP.NET Core WebApp and/or Api and one or more .NET Core or .NET Standard libraries.
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

Remark: You might want to add **two** deployment slots: One to deploy your automated build to, and one for staging/testing. Then you can safely swap your staging slot to production when testing is ready. The old production version will be in the staging slot and there is no risk for accidentally overwriting it with an automated build just when you want to swap back the old production version because of an unexpected issue.

- Select your App Service in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **App services**)
- In the left pane, select **Deployment slots** > **Add Slot**.
- Set the name of the slot (e.g. Staging). At **Clone settings from:** select your production/parent App Service to copy its settings. Click **Add**.
- After creation click **Close**.
- Click the newly created deployment slot name. Your slot is now shown as an **App Service**. The settings in the **Configuration** tab are copied from your parent App Service, but other settings not. Check for correct values.
- If you use the **Key Vault**, make sure to turn on **Managed identities** for Azure resources: in the **Identity** section turn on **Status** in the **System assigned** tab and select Save. You also must enable the slot to use the KeyVault with its Managed Identity. **Principal name** is `yourAppServiceName/slots/yourSlotName`. See: [Enable the WebApi/WebApp to use the KeyVault with its Managed Identity](appservice#enable-the-webapiwebapp-to-use-the-keyvault-with-its-managed-identity){:target="_blank"}
- If you use AD B2C authentication, make sure to add the URL of the deployment slot to the callback urls of your AD B2C directory. See: TODO.

## TODO

### Load the source code

* If you use **private NuGet packages** or when your NuGet packages are specified in **package.json**, see: [Restore dependencies](https://docs.microsoft.com/en-us/azure/devops/pipelines/ecosystems/dotnet-core?view=azure-devops#restore-dependencies){:target="_blank"}

