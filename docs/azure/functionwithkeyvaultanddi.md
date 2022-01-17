---
title: Function with DI and KeyVault
parent: Azure
has_children: false
nav_order: 3
---

## Create .NET 6 Azure Function App with Dependency Injection and the Key Vault as Configuration Provider
{: .no_toc }

_Last update: January 18, 2022_<br/>
Source code in Git: [Azure Function App Example](https://github.com/Forestbrook/FunctionWithKeyVaultAndDI){:target="_blank"}

Creating a basic Azure Function App is simple, but when you have to build a professional Function App it is not always easy to find the right instructions and documentation.

In this article I will show all tasks involved to create a professional .NET Core Azure Function App, compact and step-by-step.

This article is also intended as a reference source.

I will use the new Azure.Identity library, which makes it very easy to use the Azure KeyVault as a Configuration Provider.

REMARKS: 

If your Function App needs an Azure Storage Account, you can store the connection string for the storage credentials in the KeyVault as a secret named **AzureWebJobsStorage** as described below. Unfortunately, when you test local, the Function App does NOT read AzureWebJobsStorage from the configuration/KeyVault, but requires it to be stored in `local.settings.json`. To prevent storing keys on your local computer, you can set AzureWebJobsStorage to `"UseDevelopmentStorage=true"` in local.settings.json.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Create and configure resources in the Azure Portal

### Create a Key Vault
- Use a single Key Vault for every app group (a group may consist of, for example: Function Apps, Api, WebApp, Mobile and tools).
- I recommend to use different Key Vaults for Development/Staging and Production.
- Create a Key Vault in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Key vaults**)
- Make sure to select the correct **Subscription**, **Resource group**, **Region**, **Pricing tier** (Standard).

### Create a (Blob) Storage Account
- Create a Storage Account in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Storage accounts**)
- Make sure to select the correct **Subscription**, **Resource group** and **Location**. Set a **Storage account name** (lowercase only), **Performance** (Standard), **Redundancy** _(LRS or GRS)_.
- In the **Advanced** tab Check the **Security** settings.
- In the **Data protection** tab check the **Recovery** and **Tracking** options.
- Select **Review + create**, verify the selected settings and select **Create**.
- Wait until **Your deployment is complete** is shown, then select **Go to resource**.
- In the Storage account left menu select **Access keys**. Copy the (key 1) **Connection string** (the connection string includes the key).

### Add secrets (configuration values) to the Key Vault
- Go back to the Key Vault you created above.
- In the **Secrets** left menu select **+ Generate/Import**.
- Set **Name** to **AzureWebJobsStorage** and in **Value** paste the Storage Connection string.
- For the [Function Example App](https://github.com/Forestbrook/FunctionWithKeyVaultAndDI){:target="_blank"} add two more secrets:
  - Name: **DbCredentials:UserId**, Value: **UserId read from the KeyVault**
  - Name: **TestSecret**, Value: **Test Secret stored in the KeyVault**

### Create a Function App
- Create a Function App in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Function App**)
- Make sure to select the correct values for **Subscription**, **Resource group**, **Publish** (Code), **Runtime stack** (.NET), Version (6) and Region.
- In the **Hosting** tab select the **Storage account** you created above, the **Operating System**, and the **Plan Type** (Consumption (Serverless)).
- In the **Monitoring** tab: When needed, enable Application Insights and select the correct one (or create a new one).
- Select **Review + create**, verify the selected settings and select **Create**.
- Wait until **Your deployment is complete** is shown, then select **Go to resource**.
- Select **Configuration** in the left menu. Azure added the setting **AzureWebJobsStorage**. You can remove this setting, because your app will read the setting from the KeyVault. Don't forget to click the **Save** button after removing the setting.

### Enable access of your Function App to the KeyVault
- Select **Identity** in the left menu of your Function App, turn on **Status** on the **System assigned** tab and press **Save** > **Yes**.
- Wait until Save is ready, then copy the **Object ID**.
- Go back to the Key Vault you created above.
- Select **Access policies** in the left menu and choose **+ Add Access Policy**.
- Select **Secret permissions** _Get_ and _List_.
- At **Select principal** click "None selected" and paste the Object ID you copied in the search area. Your Function App will appear. Select it and click the **Select** button at the bottom.
- Click the **Add** button and _don't forget to click the **Save** button_. Your Function App can now read configuration values from the KeyVault.

## Create the Function App Solution

### Create a new Function App Solution in Visual Studio
- In Visual Studio select **Create a new project** or **File** > **New** > **Project...**.
- Select the **AzureFunctions** template and click **Next**.
- Set your **Project name**, **Location** and **Solution name** and click **Create**
- Select the Azure Functions version (V3) and the Function Type (Timer trigger). At **Storage account (AzureWebJobsStorage)** keep the value **Storage emulator**. Do NOT select the Storage Account you created above, because this will write the Storage key to the `local.settings.json` file. When the App is in production (running on Azure) it will read the Storage account connection string from the KeyVault.

### Add FunctionsStartup class with the KeyVault as a configuration provider

1. Add Nuget packages:
- Azure.Extensions.AspNetCore.Configuration.Secrets
- Azure.Identity
- Microsoft.Azure.Functions.Extensions

2. Add an **appsettings.json** file:
```json
{
    "KeyVaultName": "--your-key-vault-name--"
}
```
- Select the appsettings.json file in Solution Explorer and in the Properties at **Copy to Output Directory** select **Copy if newer**.

3. Add a **Startup.cs** class

_Make sure to add `[assembly: FunctionsStartup(typeof(YourNamespace.Startup))]` at top of the file!_

```cs
using Azure.Identity;
using Microsoft.Azure.Functions.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using System;
using System.IO;

[assembly: FunctionsStartup(typeof(Forestbrook.FunctionWithKeyVaultAndDI.Startup))]
namespace Forestbrook.FunctionWithKeyVaultAndDI
{
    public class Startup : FunctionsStartup
    {
        private IConfiguration _configuration;

        public override void Configure(IFunctionsHostBuilder builder)
        {
            // Configure your services here.
        }

        public override void ConfigureAppConfiguration(IFunctionsConfigurationBuilder builder)
        {
            var context = builder.GetContext();
            var configurationBuilder = builder.ConfigurationBuilder;
            configurationBuilder.AddJsonFile(Path.Combine(context.ApplicationRootPath, "appsettings.json"), true, false)
                .AddEnvironmentVariables();

            // Add the Key Vault:
            var configuration = configurationBuilder.Build();
            var keyVaultUri = $"https://{configuration["KeyVaultName"]}.vault.azure.net/";
            configurationBuilder.AddAzureKeyVault(new Uri(keyVaultUri), new DefaultAzureCredential());

            _configuration = configurationBuilder.Build();
        }
    }
}
```

### Add a service: DemoService.cs

```cs
public class DemoService
{
    public DemoService(string userId, string testSecret)
    {
        UserId = userId;
        TestSecret = testSecret;
    }

    public string TestSecret { get; }

    public string UserId { get; }
}
```

Configure DemoService in Startup: In the **Configure** method add this line:

```cs
builder.Services.AddSingleton(s => new DemoService(_configuration["DbCredentials:UserId"], _configuration["TestSecret"]));
```

See [Service lifetimes](https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection#service-lifetimes){:target="_blank"} to figure out when to use Singleton, Transient or Scoped.

### Change the Function1 class: get rid of Static and add a Service
- Remove Static
- Add the **DemoService** which will be loaded by Dependancy Injection.
- Mind that the **Run** function can be `async Task` if you need to call Async methods.

```cs
public class Function1
{
    private readonly DemoService _demoService;

    public Function1(DemoService demoService)
    {
        _demoService = demoService ?? throw new ArgumentNullException(nameof(demoService));
    }

    [FunctionName("Function1")]
    public void Run([TimerTrigger("0 */5 * * * *")]TimerInfo myTimer, ILogger log)
    {
        log.LogInformation(
            @$"-----------------------------------------------
C# Timer trigger function executed at: {DateTime.Now}
UserId: {_demoService.UserId}
TestSecret: {_demoService.TestSecret}
-----------------------------------------------");
    }
}
```

## Local testing/debugging in Visual Studio

When you run the Function App from Visual Studio on your local computer for debugging, the Function App can magically connect to the KeyVault. This magic happens in the `DefaultAzureCredential()` which was connected to the KeyVault Secrets configuration provider added in Startup.

See [Authenticating via Visual Studio](https://docs.microsoft.com/en-us/dotnet/api/overview/azure/identity-readme#authenticating-via-visual-studio){:target="_blank"}

To make this work, your Microsoft Account must have at least _Get_ and _List_ access to the KeyVault.

### Issues

1. Unfortunately, when you test local, the Function App does NOT read AzureWebJobsStorage from the configuration/KeyVault, but requires it to be stored in `local.settings.json`. To prevent storing keys on your local computer, you can set AzureWebJobsStorage to `"UseDevelopmentStorage=true"` in local.settings.json.

2. I have multiple accounts connected to Visual Studio and I had some issues getting this to work. I found the solution here: [DefaultAzureCredential fails when multiple accounts are available and defaulting to SharedTokenCacheCredential](https://github.com/Azure/azure-sdk-for-net/issues/8658#issuecomment-656223272){:target="_blank"}.
- I had to set the environment variables **AZURE_USERNAME** and **AZURE_TENANT_ID**
- I did _**not**_ have to set the DefaultAzureCredentialOptions.

Here is were you can find your Azure **tennant ID**:
* Go to the [Azure Portal](https://portal.azure.com){:target="_blank"}
* When necessary, switch to the Active Directory with the KeyVault (mostly the default directory).
* Search for and select **Tenant properties**
* Copy the **Tenant ID**.

To set the **environment variables** on your PC:
* In **File Explorer** right click **This PC**
* Select **Properties**
* Click **Change Settings**
* Click the **Advanced** tab
* Click **Environment variables...**.

Remember to **restart Visual Studio after setting the environment variables**.

If your account is not configured correctly, you will get an `Azure.Identity.AuthenticationFailedException`.

Because you have to restart Visual Studio every time you change the environment variables, you can set them temporary in the Debug Properties of your project, until you found the correct values.

### Run the Function App

   ![Result.png](/assets/images/azure-function-result.png)

## Publish to Azure

- Go to your Function App in the [Azure Portal](https://portal.azure.com){:target="_blank"}
- In the **Overview** area select **Get publish profile**. This will download the publish profile to your PC.
- In Visual Studio right click your Function App Project and choose **Publish...**
- Select **Import Profile**, browse to the downloaded profile and select **Finish**.
- Click **Publish**. This will build the Release version of you App and publish it to your Azure Function App.
- In the Publish view in Visual Studio you can click **View streaming logs** to see the logging of your running Azure Function App!


## References

* [Use dependency injection in .NET Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection){:target="_blank"}
* [GitHub issue: Roadmap for .NET 5 and Azure Functions V3](https://github.com/Azure/azure-functions-host/issues/6674){:target="_blank"}
