---
title: Function with DI and KeyVault
parent: Azure
has_children: false
nav_order: 3
---

## Create Azure Function App with Dependency Injection and the Key Vault as Configuration Provider
{: .no_toc }

_Last update: October 20, 2020_<br/>
Source code in Git: [Azure Function App Example](https://github.com/Forestbrook/FunctionWithKeyVaultAndDI){:target="_blank"}

The Visual Studio project template for an Azure Function App is very basic and the Microsoft documentation is not always clear and up-to-date.

This article shows step-by-step how to create a .NET Core Azure Function App.

I will use the new Azure.Identity library, which makes is very easy to use the Azure KeyVault as a Configuration Provider.

REMARKS: 

* Azure Functions do not support .NET 5 yet. See [GitHub issue](https://github.com/Azure/azure-functions-host/issues/6674){:target="_blank"}.
* If your Function App needs an Azure Storage Account, you can store the connection string for the storage credentials in the KeyVault as a secret named **AzureWebJobsStorage**. Unfortunately when you test local, the Function App does NOT read AzureWebJobsStorage from the configuration, but requires it to be stored in `local.settings.json`. To prevent storing keys on your local computer, you can set AzureWebJobsStorage to `"UseDevelopmentStorage=true"` in local.settings.json.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Create and configure resources in the Azure Portal

### Create a Key Vault
- Use a single Key Vault for every app group (a group may consist of, for example: Function Apps, Api, WebApp, Mobile and tools).
- I recommend to use different Key Vaults for Development/Testing and Production.
- Create a Key Vault in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Key vaults**)
- Make sure to select the correct **Subscription**, **Resource group**, **Region**, **Pricing tier** (Standard).

### Create a (Blob) Storage Account
- Create a Storage Account in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Storage accounts**)
- Make sure to select the correct **Subscription**, **Resource group** and **Location**. Set a **Storage account name** (lowercase only), **Account kind** (StorageV2), **Performance** (Standard), **Replication** _(Locally-redundant storage (LRS))_.
- Select **Review + create**, verify the selected settings and select **Create**.
- Wait until **Your deployment is complete** is shown, then select **Go to resource**.
- In the Storage account left menu select **Access keys**. Copy the (key 1) **Connection string**.

### Add secrets to the Key Vault
- Go back to the Key Vault you created above.
- In the **Secrets** left menu select **+ Generate/Import**.
- Set **Name** to **AzureWebJobsStorage** and in **Value** paste the Stoage Connection string.
- For the [Function Example App](https://github.com/Forestbrook/FunctionWithKeyVaultAndDI){:target="_blank"} add two more secrets:
  - Name: **DbCredentials:UserId**, Value: **UserId read from the KeyVault**
  - Name: **TestSecret**, Value: **Test Secret stored in the KeyVault**

### Create a Function App
- Create a Function App in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Function App**)
- Make sure to select the correct values for **Subscription**, **Resource group**, **Publish** (Code), **Runtime stack** (.NET Core), Version (3.1) and Region.
- Select **Next: Hosting >** and select the **Storage account** you created above, the **Operating System** (Windows), and the **Plan Type** (Consumption (Serverless)).
- Select **Next: Monitoring >** and select or create new **Application Insight** for your App.
- Select **Review + create**, verify the selected settings and select **Create**.
- Wait until **Your deployment is complete** is shown, then select **Go to resource**.

### Enable access of your Function App to the KeyVault
- Select **Identity** in the left menu of your Function App, turn on **Status** on the **System assigned** tab and press **Save** > **Yes**.
- Wait until Save is ready, then copy the **Object ID**.
- Go back to the Key Vault you created above.
- Select **Access policies** in the left menu and choose **+ Add Access Policy**.
- Select **Secret permissions** _Get_ and _List_.
- At **Select principal** click "None selected" and paste the Object ID you copied in the search area. Your Function App will appear. Select it and click the **Select** button at the bottom.
- Click the **Add** button and _don't forget to click the **Save** button_. Your Function App can now read configuration values from the KeyVault.









## References

* [Use dependency injection in .NET Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection){:target="_blank"}
* [GitHub issue Roadmap for .NET 5 and Azure Functions V3](https://github.com/Azure/azure-functions-host/issues/6674){:target="_blank"}


