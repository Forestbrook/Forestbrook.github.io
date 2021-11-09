---
title: Azure WebApi and WebApp
parent: Azure
has_children: false
nav_order: 1
---

## Configure and create an ASP.NET 5 WebApi or WebApp in an Azure App Service
{: .no_toc }

_Last update: November 9, 2021_<br/>
Source code in Git: [Azure WebApi and WebApp with KeyVault](https://github.com/Forestbrook/WebApiWithKeyVault){:target="_blank"}

A professional Azure ASP.NET 5 WebApi or WebApp requires several Azure resources.
This article is intended to keep track of the setup and configuration of these resources and how they are integrated in the application.
Involved resources:

* Azure App Service Plan and App Service
* Azure Key Vault
* Azure SQL Server and SQL database accessed thru the Entity Framework
* Blob Storage with an Azure Storage Account
* Authentication and Authorization (separate article, coming soon...)

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Create and configure resources in the Azure Portal

### Create or use an Azure App Service Plan
- An **App Service Plan** can host multiple apps. You will be charged only for the App Service Plan.
- Create an App Service Plan in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **App Service Plans**)
- Make sure to select the correct Subscription, Resource Group, Operating System and Region
- Carefully look at the **Pricing Tier**/costs.

### Create a Key Vault
- Use a single Key Vault for every app group (a group may consist of, for example: Api, WebApp, Mobile and tools).
- I recommend to use different Key Vaults for Staging and Production.
- Create a Key Vault in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Key vaults**)
- Make sure to select the correct **Subscription**, **Resource group**, **Region**, **Pricing tier** (Standard).

### Create and/or setup SQL Server
- An Azure **SQL Server** can host multiple databases. You will be charged for the databases, not for the server.
- Create a SQL Server in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **SQL servers**)
- Make sure to select the correct **Subscription**, **Resource group**, **Location**.
- Set **Server admin login** and **Password** and store them in the KeyVault as **Secrets**. Secret names (used in this article): **DbCredentials--UserId** and **DbCredentials--Password** (in the Key Vault **--** is used as section separator). You can access them in your app later like `Configuration.GetValue<string>("DbCredentials:UserId")`.
- In **Networking** > **Firewalls and virtual networks** turn on **Allow Azure services and resources to access this server**
- **After creation**: If you need access to the databases with e.g. SQL Server Management Studio, select the **Firewalls and virtual networks** tab and add your own ip address(es) as a Rule (your own ip address is shown at **Client IP address**).

### Create the SQL database
- Create a SQL Database in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **SQL databases**)
- Make sure to select the correct **Subscription**, **Resource group** and **Server** and enter your **Database name**.
- At **Compute + storage** > **Configure** select the right pricing model. _The default is quite expensive._ If your database is not too big, start with **Basic**, **Standard** or **Serverless**.
- In Additional settings you can select the Database Collation (sort/compare rules).
- Make sure to turn off any free trials you do not want in Additional settings.

### Create a (Blob) Storage Account
- Create a Storage Account in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Storage accounts**)
- Make sure to select the correct **Subscription**, **Resource group** and **Location**. Set a **Storage account name** (lowercase only), **Account kind** (StorageV2), **Performance** (Standard), **Replication** _(Locally-redundant storage (LRS))_.
- Select **Review + create**, verify the selected settings and select **Create**.
- Wait until **Your deployment is complete** is shown, then select **Go to resource**.
- In the Storage account left menu select **Access keys**. Copy the (key 1) **Connection string** and store it in the KeyVault as **Secret**. Secret name (used in this article): **StorageCredentials--ConnectionString**. You can access it in your app later like `Configuration.GetValue<string>("StorageCredentials:ConnectionString")`.

### Create the App Service (Web App) for the WebApi or WebApp
- Create an **App Service** in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **App services**)
- Make sure to select the correct **Subscription**, **Resource Group**, **Runtime stack** (.NET 5), **Operating System** (Windows), **Region** and **App Service Plan**.
- Monitoring: When needed, enable Application Insights and select the correct one (or create a new one). This will generate all **configuration settings** for Application Insights, but make sure you add `services.AddApplicationInsightsTelemetry();` in Startup.cs!

After creation:
- TLS/SSL settings: **HTTPS Only** and Minimum TLS Version: **1.2**
- In the Overview section select **Get publish profile** and store in temp folder.
- In the **Configuration** section under **Connection strings** add the connection string for your database (replace _--your-SQL-Server-name--_ and _--your-database-name--_ with the correct name):
  - Name (used in this article): **AppDb**
  - Value: `Data Source=--your-SQL-Server-name--.database.windows.net;Initial Catalog=--your-database-name--; Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False`
  - Type: **SQLServer**
- In the **Configuration** section under **Application settings** add:
  - Name: **ASPNETCORE_ENVIRONMENT**, Value: **Production** (you can replace that with Development for debugging)
  - Optional Name: **WEBSITE_TIME_ZONE**, Value: your time zone (e.g. W. Europe Standard Time)
  - Optional you might need for Azure AD-B2C (add only if you are sure you need this) Name: WEBSITE_LOAD_USER_PROFILE, Value: 1

  ASPNETCORE_ENVIRONMENT

### Enable the WebApi/WebApp to use the KeyVault with its Managed Identity
- In your App Service, in the **Identity** section, turn on **Status** in the **System assigned** tab and select Save. Wait until Save is ready, then copy the **Object ID**.
- Select your Key Vault in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Key vaults**)
- Select the **Access policies** section and click **+ Add Access Policy**
- **Secret permissions**: Get, List.
- At **Select principal** click "None selected" and paste the Object ID you copied in the search area. Your App Service will appear. Select it and click the **Select** button at the bottom. You can also search with the name of your App Service instead of the Object ID. The name of a [Deployment slot](pipelines#add-a-deployment-slot){:target="_blank"} of your App Service is `--your-App-Service-name--/slots/--your-Slot-name--`.
- Save with **Add** and _don't forget to_ click **Save** (upper left) to commit the changes.

## Where to store configuration settings for the App Service

See also: [Configuration in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/){:target="_blank"}

Configuration settings are stored in several places, depending on the setting type.
- Secrets (production, staging) in the **KeyVault**. Use -- instead of : as section separator.
- Secrets (development): With the Azure.Identity package it is possible to get the configuration values from the KeyVault when debugging from Visual Studio, using your Microsoft account connected to Visual Studio. This account must have at least Get and List access to the KeyVault. Alternativeliy, you can use the **Secret Manager** to store your secrets in a private **secrets.json** file. In Visual Studio: Right-click your project and select **Manage User Secrets**
- In the **Configuration** section of the App Service ([Azure Portal](https://portal.azure.com){:target="_blank"}): Connection strings (**without** secrets) etc. (for local debugging you can use `appsettings.Development.json` in the project)
- General settings in the project files `appsettings.json`. Override with `appsettings.Development.json`, `appsettings.Staging.json`, `appsettings.Production.json`

## Environments (Development, Staging, Production)

See also: [Use multiple environments in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/environments){:target="_blank"}

- Set the environment with the **ASPNETCORE_ENVIRONMENT** variable in the App Service Configuration (for debugging in `Properties/launchSettings.json`). Values: Development, Staging, Production.
- In `Startup.cs` you can use `env.IsDevelopment(), env.IsProduction(), env.IsStaging()`
- In `Program.cs` you can use `context.HostingEnvironment.IsProduction()` etc. (context is `WebHostBuilderContext)`

## Create the project solution

- Create an ASP.NET Core Web Application project.
- Select the API template or the Web Application (Model-View-Controller) template.
- At the right options make sure **Configure for HTTPS** is selected.
- Authentication: See Azure AD B2C article.

### Add the KeyVault as a configuration provider

1. Add Nuget packages:
- Azure.Extensions.AspNetCore.Configuration.Secrets
- Azure.Identity

2. In **appsettings.json** add:
   ```json
   "KeyVaultName": "--your-key-vault-name--",
   ```

3. Add extensions class **AspNetCoreHelper.cs**:
   ```cs
   using Azure.Identity;
   using Microsoft.AspNetCore.Hosting;
   using Microsoft.Extensions.Configuration;
   using System;
   
   namespace Forestbrook.WebApiWithKeyVault
   {
       public static class AspNetCoreHelper
       {
           private const string DefaultKeyVaultNameConfigurationKey = "KeyVaultName";
   
           /// <summary>
           /// Make sure to add to your appsettings.json: "KeyVaultName": "your-key-vault-name"
           /// </summary>
           public static IWebHostBuilder ConfigureAppConfigurationWithKeyVault(this IWebHostBuilder builder, string keyVaultNameConfigurationKey = DefaultKeyVaultNameConfigurationKey)
           {
               return builder.ConfigureAppConfiguration(AddKeyVaultConfigurationProvider);
   
               void AddKeyVaultConfigurationProvider(WebHostBuilderContext context, IConfigurationBuilder config)
               {
                   var builtConfig = config.Build();
                   var keyVaultUri = $"https://{builtConfig[keyVaultNameConfigurationKey]}.vault.azure.net/";
                   config.AddAzureKeyVault(new Uri(keyVaultUri), new DefaultAzureCredential());
               }
           }
       }
   }
   ```

4. In **Program.cs** change `webBuilder.UseStartup<Startup>();` to:
   ```cs
   webBuilder
       .ConfigureAppConfigurationWithKeyVault()
       .UseStartup<Startup>();
   ```

### Add a Storage Repository

If you created a (Blob) Storage Account above, you can implement a Storage Repository.

1. Example repository:
   ```cs
   public class BlobStorageRepository
   {
       public BlobStorageRepository(string storageConnectionString)
       {
           if (string.IsNullOrEmpty(storageConnectionString)) throw new ArgumentNullException(nameof(storageConnectionString));
           // TODO: Implement your repository using the Nuget Package Azure.Storage.Blobs
           //_blobServiceClient = new BlobServiceClient(storageConnectionString);
       }
   }
   ```

2. In **Startup.cs** in the `ConfigureServices` method add this line:
   ```cs
   services.AddScoped(s => new BlobStorageRepository(Configuration.GetValue<string>("StorageCredentials:ConnectionString")));
   ```

### Add a SQL Server Database Repository

If you created a SQL Database above, you can implement a SQL Server Database Repository.

1. Example repository: -- Skip this step if you want to use the Entity Framework (see below).
   ```cs
   public class SqlServerDatabaseRepository
   {
       public SqlServerDatabaseRepository(string connectionString)
       {
           if (string.IsNullOrEmpty(connectionString)) throw new ArgumentNullException(nameof(connectionString));
           // TODO: Implement your repository
       }
   }
   ```

2. To enable local testing and debugging, add the database connection string which you added to your App Service Configuration above to **appsettings.Development.json**:
   ```json
  "ConnectionStrings": {
      "AppDb": "Data Source=--your-SQL-Server-name--.database.windows.net;Initial Catalog=--your-database-name--;Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False"
  },
   ```

3. In **Startup.cs** (or if you prefer in AspNetCoreHelper) add this helper method:
   ```cs
   private static string CreateSqlConnectionString(IConfiguration configuration)
   {
       var dbUserID = configuration.GetValue<string>("DbCredentials:UserId");
       var dbPassword = configuration.GetValue<string>("DbCredentials:Password");
       var builder = new SqlConnectionStringBuilder(configuration.GetConnectionString("AppDb")) { UserID = dbUserID, Password = dbPassword };
       return builder.ConnectionString;
   }
   ```

4. In **Startup.cs** in the `ConfigureServices` method add this line:
   ```cs
   services.AddScoped(s => new SqlServerDatabaseRepository(CreateSqlConnectionString(Configuration)));
   ```

## Add Entity Framework

### DataContext and Models

- Create a .NET Core class library --your-app-name--.**Models**.
- Add the Models for the database tables.
- Create a .NET Core class library --your-app-name--.**DataAccess**.
- In the Data folder Create the DbContext with the DbSets for the tables.

### Register DbContext in Startup.cs

- In `ConfigureServices()` add the code for the database connection and add the DbContext:
  ```cs
      // Setting for EnableSqlParameterLogging: prod: turn on/off in environment; dev: see appsettings.Development.json
      services.AddDbContext<YourContext>(options => options.UseSqlServer(CreateSqlConnectionString(Configuration))
          .EnableSensitiveDataLogging(Configuration.GetValue<bool>("Logging:EnableSqlParameterLogging")));
  ```

### Create a migration

When necesary, install EF Core tools. See: [Entity Framework Core tools reference](https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dotnet){:target="_blank"}

- Open **PowerShell** and CD to the **WebApp or WebApi project folder**
- `dotnet build`
- `dotnet ef migrations add InitialCreate --project ../DataAccess/--your-app-name--.DataAccess.csproj --context YourContext`
- `dotnet ef database update --project ../DataAccess/--your-app-name--.DataAccess.csproj --context YourContext`

## Add Application Insights

See: [Application Insights for ASP.NET Core applications](https://docs.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core){:target="_blank"}

- In your WebApp/Api projects add the Nuget Package `Microsoft.ApplicationInsights.AspNetCore`
- In Startup.cs ConfigureServices() add:
  `services.AddApplicationInsightsTelemetry();`
- If you connected your WebService to Application Insights, the required configuration settings are already present.

## Local testing/debugging in Visual Studio

When you run your App from Visual Studio on your local computer in IIS Express for debugging, the App can magically connect to the KeyVault. This magic happens in the `DefaultAzureCredential()` which was connected to the KeyVault Secrets configuration provider added in Startup.

See [Authenticating via Visual Studio](https://docs.microsoft.com/en-us/dotnet/api/overview/azure/identity-readme#authenticating-via-visual-studio){:target="_blank"}

To make this work, your Microsoft Account must have at least _Get_ and _List_ access to the KeyVault.

### Issues

I have multiple accounts connected to Visual Studio and I had some issues getting this to work. I found the solution here: [DefaultAzureCredential fails when multiple accounts are available and defaulting to SharedTokenCacheCredential](https://github.com/Azure/azure-sdk-for-net/issues/8658#issuecomment-656223272){:target="_blank"}.
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
