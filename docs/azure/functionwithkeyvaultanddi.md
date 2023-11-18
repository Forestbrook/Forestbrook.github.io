---
title: Function with DI and KeyVault
parent: Azure
has_children: false
nav_order: 3
---

## Create .NET 6 Azure Function App with Dependency Injection and the Key Vault as Configuration Provider
{: .no_toc }

_Last update: January 22, 2022_<br/>
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
- Make sure to select the correct values for **Subscription**, **Resource group**, **Publish** (Code), **Runtime stack** (.NET), **Version** (6) and **Region**.
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
- Select the **Azure Functions** template and click **Next**.
- Set your **Project name**, **Location** and **Solution name** and click **Create**
- Select the version (.NET 6) and the **Timer trigger** template. At **Storage account (AzureWebJobsStorage)** keep the value **Storage emulator**. Do NOT select the Storage Account you created above, because this will write the Storage key to the `local.settings.json` file. When the App is in production (running on Azure) it will read the Storage account connection string from the KeyVault.

### Add FunctionsStartup class with the KeyVault as a configuration provider

1. Add Nuget packages:
- Azure.Extensions.AspNetCore.Configuration.Secrets
- Azure.Identity
- Microsoft.Azure.Functions.Extensions
- Microsoft.Extensions.Configuration.UserSecrets
- Microsoft.Extensions.Hosting.Abstractions
- Microsoft.NET.Sdk.Functions

2. Add an **appsettings.json** file:
```json
{
    "KeyVaultName": "--your-key-vault-name--"
}
```
- Select the appsettings.json file in Solution Explorer and in the Properties at **Copy to Output Directory** select **Copy if newer**. (Keep Build Action: None).

3. Add a static class **ConfigurationKeys.cs**:
   ```cs
   namespace Forestbrook.FunctionWithKeyVaultAndDI;
   
   public static class ConfigurationKeys
   {
       /// <summary>
       /// StorageConnectionString for Azure Function
       /// </summary>
       public const string AzureWebJobsStorage = "AzureWebJobsStorage";
   
       public const string DatabaseUserId = "DbCredentials:UserId";
       public const string KeyVaultName = "KeyVaultName";
       public const string KeyVaultTenantId = "KeyVaultTenantId";
       public const string StorageConnectionString = "StorageCredentials:ConnectionString";
   }
   ```

4. Add the static helper class **FunctionHelper.cs**:
   ```cs
   using Azure.Core;
   using Azure.Identity;
   using Microsoft.Azure.Functions.Extensions.DependencyInjection;
   using Microsoft.Extensions.Configuration;
   using System;
   using System.IO;
   
   namespace Forestbrook.FunctionWithKeyVaultAndDI;
   
   public static class FunctionHelper
   {
       public static IConfigurationBuilder AddAppSettingsJson(this IConfigurationBuilder builder, FunctionsHostBuilderContext context)
       {
           builder.AddJsonFile(Path.Combine(context.ApplicationRootPath, "appsettings.json"), optional: true, reloadOnChange: false);
           builder.AddJsonFile(Path.Combine(context.ApplicationRootPath, $"appsettings.{context.EnvironmentName}.json"), optional: true, reloadOnChange: false);
           return builder;
       }
   
       public static IConfigurationBuilder AddAzureKeyVault(this IConfigurationBuilder builder)
           => builder.AddAzureKeyVault(builder.Build());
   
       /// <summary>
       /// Make sure to add to your appsettings.json: "KeyVaultName": "your-key-vault-name"
       /// </summary>
       public static IConfigurationBuilder AddAzureKeyVault(this IConfigurationBuilder builder, IConfiguration configuration)
       {
           var keyVaultUri = configuration.CreateKeyVaultUri();
           var keyVaultCredential = configuration.CreateKeyVaultCredential();
           builder.AddAzureKeyVault(keyVaultUri, keyVaultCredential);
           return builder;
       }
   
       private static TokenCredential CreateKeyVaultCredential(this IConfiguration configuration)
       {
           // WARNING: Make sure to give the App in the Azure Portal access to the KeyVault.
           //          In the Identity tab: System Assigned part: turn Status On and copy the Object ID.
           //          In the KeyVault: Access Policies > Add Access Policy > Secret Permissions Get, List and Select Principal: Object ID copied above.
           // When running on Azure, you do NOT need to set the KeyVaultTenantId.
           var keyVaultTenantId = configuration[ConfigurationKeys.KeyVaultTenantId];
           if (string.IsNullOrEmpty(keyVaultTenantId))
               return new DefaultAzureCredential();
   
           // When debugging local from VisualStudio AND the TenantId differs from default AZURE_TENANT_ID (in Windows settings/environment variables),
           // you can store KeyVaultTenantId= in appsettings or in UserSecrets and read it here from the configuration (as done above)
           var options = new DefaultAzureCredentialOptions { VisualStudioTenantId = keyVaultTenantId };
           return new DefaultAzureCredential(options);
       }
   
       private static Uri CreateKeyVaultUri(this IConfiguration configuration)
       {
           if (configuration == null) throw new ArgumentNullException(nameof(configuration));
           var keyVaultName = configuration[ConfigurationKeys.KeyVaultName];
           if (string.IsNullOrEmpty(keyVaultName))
               throw new InvalidOperationException($"Missing configuration setting {ConfigurationKeys.KeyVaultName}");
   
           return new Uri($"https://{keyVaultName}.vault.azure.net/");
       }
   }
   ```

5. Add the **Startup.cs** class
   _Make sure to add `[assembly: FunctionsStartup(typeof(YourNamespace.Startup))]` at top of the file!_
   ```cs
   using Microsoft.Azure.Functions.Extensions.DependencyInjection;
   using Microsoft.Extensions.Configuration;
   using Microsoft.Extensions.DependencyInjection;

   [assembly: FunctionsStartup(typeof(Forestbrook.FunctionWithKeyVaultAndDI.Startup))]
   
   namespace Forestbrook.FunctionWithKeyVaultAndDI;
   
   public class Startup : FunctionsStartup
   {
       public override void Configure(IFunctionsHostBuilder builder)
       {
           // Configure your services here.
       }
   
       public override void ConfigureAppConfiguration(IFunctionsConfigurationBuilder builder)
       {
           // local.settings.json are automatically loaded when debugging.
           // When running on Azure, values are loaded defined in app settings. See: https://docs.microsoft.com/en-us/azure/azure-functions/functions-how-to-use-azure-function-app-settings
           builder.ConfigurationBuilder
               .AddAppSettingsJson(builder.GetContext())
               .AddEnvironmentVariables()
               .AddUserSecrets<Startup>(true)
               .AddAzureKeyVault()
               .Build();
       }
   }
   ```

### Add a service: DemoService.cs

```cs
using Microsoft.Extensions.Configuration;
using System;

namespace Forestbrook.FunctionWithKeyVaultAndDI;

public class DemoService
{
    public DemoService(IConfiguration configuration)
    {
        if (configuration == null) throw new ArgumentNullException(nameof(configuration));

        // Get test secrets from the KeyVault:
        UserId = configuration[ConfigurationKeys.DatabaseUserId];
        TestSecret = configuration["TestSecret"];
    }

    public string TestSecret { get; }

    public string UserId { get; }
}
```

Configure DemoService in Startup: In the **Configure** method add this line:

```cs
builder.Services.AddSingleton<DemoService>();
```

See [Service lifetimes](https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection#service-lifetimes){:target="_blank"} to figure out when to use Singleton, Transient or Scoped.

### Add a HttpTrigger Function with dependancy injection: IsAliveFunction.cs

- Remove the Function1 class
- Mind that the **Run** function can be `async Task` if you need to call Async methods.

```cs
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.Extensions.Hosting;
using System;
using System.Text;
using System.Threading.Tasks;

namespace Forestbrook.FunctionWithKeyVaultAndDI;

public class IsAliveFunction
{
    // Test local: http://localhost:7071/api/IsAliveFunction
    // Test on Azure: https://TODO-your-function-name-.azurewebsites.net/api/IsAliveFunction
    private const string IsRunningMessage = "Forestbrook Function is running. Version:";
    private readonly IHostEnvironment _hostEnvironment;

    public IsAliveFunction(IHostEnvironment hostEnvironment)
    {
        _hostEnvironment = hostEnvironment ?? throw new ArgumentNullException(nameof(hostEnvironment));
    }

    [FunctionName(nameof(IsAliveFunction))]
    public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", "post")] HttpRequest req)
    {
        // Execute async tasks:
        await Task.CompletedTask;

        var version = GetType().Assembly.GetName().Version?.ToString(3);
        var sb = new StringBuilder($"{DateTime.Now:d-M-yyyy H:mm:ss} - {IsRunningMessage} {version}<br/>");
        sb.Append("<br/>");
        sb.Append($"EnvironmentName = {_hostEnvironment.EnvironmentName}<br/>");
        sb.Append($"IsDevelopment: {_hostEnvironment.IsDevelopment()}<br/>");
        sb.Append($"IsProduction: {_hostEnvironment.IsProduction()}<br/>");
        sb.Append($"ApplicationName = {_hostEnvironment.ApplicationName}<br/>");
        sb.Append($"ContentRootPath = {_hostEnvironment.ContentRootPath}<br/>");
        sb.Append("<br/>");
        if (req.Query.Count > 0)
        {
            sb.Append("QUERY VALUES:<br/>");
            foreach (var (key, value) in req.Query)
                sb.Append($"{key} = {value}<br/>");
        }

        // Show we're alive:
        return new ContentResult()
        {
            Content = sb.ToString(),
            ContentType = "text/html",
        };
    }
}
```

### Add a TimerTrigger Function with dependancy injection: TimerTriggerFunction.cs

```cs
using Microsoft.Azure.WebJobs;
using Microsoft.Extensions.Logging;
using System;
using System.Text;

namespace Forestbrook.FunctionWithKeyVaultAndDI;

public class TimerTriggerFunction
{
    private readonly DemoService _demoService;

    public TimerTriggerFunction(DemoService demoService)
    {
        _demoService = demoService ?? throw new ArgumentNullException(nameof(demoService));
    }

    [FunctionName(nameof(TimerTriggerFunction))]
    public void Run([TimerTrigger("0 */5 * * * *")]TimerInfo myTimer, ILogger log)
    {
        var sb = new StringBuilder();
        sb.AppendLine("-----------------------------------------------");
        sb.AppendLine($"C# Timer trigger function executed at: {DateTime.Now}");
        sb.AppendLine($"UserId: {_demoService.UserId}");
        sb.AppendLine($"TestSecret: {_demoService.TestSecret}");
        sb.AppendLine("-----------------------------------------------");
        log.LogInformation(sb.ToString());
    }
}
```

## Local testing/debugging in Visual Studio

When you run the Function App from Visual Studio on your local computer for debugging, the Function App can magically connect to the KeyVault. This magic happens in the `DefaultAzureCredential()` which was connected to the KeyVault Secrets configuration provider added in Startup.

See [Authenticating via Visual Studio](https://docs.microsoft.com/en-us/dotnet/api/overview/azure/identity-readme#authenticating-via-visual-studio){:target="_blank"}

To make this work, your Microsoft Account must have at least _Get_ and _List_ access to the KeyVault.

### Issues

1. Unfortunately, when you test local, the Function App does NOT read AzureWebJobsStorage from the configuration/KeyVault, but requires it to be stored in `local.settings.json`. To prevent storing keys on your local computer, you can set AzureWebJobsStorage to `"UseDevelopmentStorage=true"` in local.settings.json.

2. You might have to tell Visual Studio your Azure tennant ID by setting the environment variable **AZURE_TENANT_ID**.
   
   Here is were you can find your Azure **tennant ID**:
   * Go to the [Azure Portal](https://portal.azure.com){:target="_blank"}
   * When necessary, switch to the Active Directory with the KeyVault.
   * Search for and select **Tenant properties**
   * Copy the **Tenant ID**.

   To set the **environment variables** on your PC:
   * In **File Explorer** right click **This PC**
   * Select **Properties**
   * Click **Change Settings**
   * Click the **Advanced** tab
   * Click **Environment variables...**.

   Remember to **restart Visual Studio after setting the environment variables**.

3. If you have solutions for more than one Azure tennant, it is quite annoying to change the AZURE_TENANT_ID environment variable. That is why I added the ConfigurationKey **KeyVaultTenantId**. You can store the KeyVaultTenantId=--your-tennant--id-- in UserSecrets (in Visual Studio right-click your WebApi-project and choose **Manage User Secrets**) or in appsettings.Development.json.

4. If you have multiple Microsoft accounts connected to Visual Studio, you might have to tell Visual Studio which account to use:
    * Select Tools > Options...
    * Go to option **Azure Service Authentication** > **Account Selection**
    * Choose an account.
    * Alternatively, you can set the environment variable **AZURE_USERNAME**

5. If your account is not configured correctly, you will get an `Azure.Identity.AuthenticationFailedException`.

### Run the Function App

   ![Result.png](/assets/images/azure-function-result.png)

## Publish to Azure

- In Visual Studio right click your Function App Project and choose **Publish...**
- Select target **Azure** and click **Next**.
- Select Specific target **Azure Function App**
- In the Publish window, make sure to select the right **Microsoft account** and **Subscription name**, select the Function App you created and select **Run from package file**. Click **Finish**.
- Click **Publish**. This will build the Release version of you App and publish it to your Azure Function App.

## References

* [Use dependency injection in .NET Azure Functions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection){:target="_blank"}
