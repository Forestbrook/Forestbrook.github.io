---
title: Azure WebApi and WebApp
parent: Azure
has_children: false
nav_order: 1
---

## Configure and create an ASP.NET Core WebApi or WebApp in an Azure App Service
{: .no_toc }

A professional Azure ASP.NET Core WebApi or WebApp requires several Azure resources.
This article is intented to keep track of the setup and configuration of these resources and how they are integrated in the application.
Involved resources:

* Azure App Service Plan and App Service
* Azure Key Vault
* Azure SQL Server and SQL database accessed thru the Entity Framework
* Blob Storage with an Azure Storage Account
* Authentication and Authorization (separate article, comming soon...)

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
- Set **Server admin login** and **Password** and store them in the KeyVault as **Secrets**. Secret names: **DbCredentials--UserId** and **DbCredentials--Password** (in the Key Vault **--** is used as section separator).
- In **Networking** > **Firewalls and virtual networks** turn on **Allow Azure services and resources to access this server**
- **After creation**: If you need access to the databases with e.g. SQL Server Management Studio, select the **Firewalls and virtual networks** tab and add your own ip address(es) as a Rule (your own ip address is shown at **Client IP address**).

### Create the SQL database
- Create a SQL Database in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **SQL databases**)
- Make sure to select the correct **Subscription**, **Resource group** and **Server** and enter your **Database name**.
- At **Compute + storage** > **Configure** select the right pricing model. The default is quite expensive. If your database is not too big, start with **Basic**, **Standard** or **Serverless**.
- In Additional settings you can select the Database Collation (sort/compare rules).
- Make sure to turn off any free trials you do not want in Additional settings.

### Create a (Blob) Storage Account
- Create a Storage Account in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Storage accounts**)
- Make sure to select the correct **Subscription**, **Resource group** and **Location**. Set a **Storage account name** (lowercese only), **Account kind** (StorageV2), **Performance** (Standard), **Replication** _(Locally-redundant storage (LRS))_ and **Access tier (default)** (Hot).
- Select **Review + create**, verify the selected settings and select **Create**.
- Wait until **Your deployment is complete** is shown, then select **Go to resource**.
- In the Storage account left menu select **Access keys** to get the key(s) and connection string(s) and store them in a safe place (Key Vault).

### Create the App Service (Web App) for the WebApi or WebApp
- Create an **App Service** in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **App services**)
- Make sure to select the correct **Subscription**, **Resource Group**, **Runtime stack** (.Net Core 3.0), **Operating System** (Windows), **Region** and **App Service Plan**.
- Monitoring: When needed, enable Application Insights and select the correct one (or create a new one). This will generate all **configuration settings** for Application Insights, but make sure you add `services.AddApplicationInsightsTelemetry();` in Startup.cs!

After creation:
- TLS/SSL settings: **HTTPS Only** and Minimum TLS Version: **1.2**
- In the Overview section select **Get publish profile** and store in temp folder.
- To use Managed identities for Azure resources (needed for the Key Vault), in the **Identity** section turn on **Status** in the **System assigned** tab and select Save.
- In the **Configuration** section under **Connection strings** add the connection string for your database (replace _sqlServerName_ with the correct name):
  - Name: **DatabaseNameDb**
  - Value: `Data Source=yourSqlServerName.database.windows.net;Initial Catalog=DatabaseName;Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False`
  - Type: **SQLServer**

### Enable the WebApi/WebApp to use the KeyVault with its Managed Identity
- Select your Key Vault in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Key vaults**)
- Select the **Access policies** section and click **+ Add Access Policy**
- **Secret permissions**: Get, List
- Select principal: Enter the name of your App Service. You can also use the Object ID from the Identity section of your AppService. The name of a [Deployment slot](pipelines#add-a-deployment-slot){:target="_blank"} of your App Service is `yourAppServiceName/slots/yourSlotName`.
- Save with **Add** and _don't forget to_ click **Save** (upper left) to commit the changes.

## Where to store configuration settings for the App Service

See also: [Configuration in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/configuration/){:target="_blank"}

Configuration settings are stored in several places, depending on the setting type.
- Secrets (production, staging) in the **KeyVault**. Use -- instead of : as section separator.
- Secrets (development): use the **Secret Manager** to store your secrets in a private **secrets.json** file. In Visual Studio: Right-click your project and select **Manage User Secrets**
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

## Add the KeyVault as a configuration provider

1. Add Nuget packages:
- Microsoft.Azure.KeyVault
- Microsoft.Azure.Services.AppAuthentication
- Microsoft.Extensions.Configuration.AzureKeyVault

2. In **appsettings.json** add:
   ```json
   "KeyVaultName": "--your-key-vault-name--",
   ```

3. In **Program.cs**:
   ```cs
   using Microsoft.Azure.KeyVault;
   using Microsoft.Azure.Services.AppAuthentication;
   using Microsoft.Extensions.Configuration;
   using Microsoft.Extensions.Configuration.AzureKeyVault;
   ...
   public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
       WebHost.CreateDefaultBuilder(args)
           .ConfigureAppConfiguration(AddKeyVaultConfigurationProvider)
           .UseStartup<Startup>();

   private static void AddKeyVaultConfigurationProvider(WebHostBuilderContext context, IConfigurationBuilder config)
   {
       // Key Vault not available in Development environment:
       if (context.HostingEnvironment.IsDevelopment())
           return;

       var builtConfig = config.Build();
       var azureServiceTokenProvider = new AzureServiceTokenProvider();
       var keyVaultClient = new KeyVaultClient(
           new KeyVaultClient.AuthenticationCallback(
               azureServiceTokenProvider.KeyVaultTokenCallback));

       config.AddAzureKeyVault(
           $"https://{builtConfig["KeyVaultName"]}.vault.azure.net/",
           keyVaultClient,
           new DefaultKeyVaultSecretManager());
   }
   ```

## Add Entity Framework

### Connection string and database credentials:

1. Add the development database credentials to secrets.json (right-click project and choose **Manage User Secrets**). Example: 
 `{ "DbCredentials:UserId": "SqlUser@your-sqlserver", "DbCredentials:Password": "yourPassword" }`
1. Add the production database credentials to the **KeyVault**. Use -- instead of : as section separator.
1. Add the development connection string to **appsettings.Development.json**: `"ConnectionStrings": { "AppDb": "Data Source=..." },`
1. Add the production connection string in the **Configuration** section of the **App Service** ([Azure Portal](https://portal.azure.com){:target="_blank"}):
- Name: **AppDb**
- Value: `Data Source=yourSqlServerName.database.windows.net;Initial Catalog=DatabaseName;Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False`
- Type: **SQLAzure**

### DataContext and Models

- Create a .NET Core class library AppName.**Models**.
- Add the Models for the database tables.
- Create a .NET Core class library AppName.**DataAccess**.
- In the Data folder Create the DbContext with the DbSets for the tables.

### Register DbContext in Startup.cs

- In `ConfigureServices()` add the code for the database connection and add the DbContext:
  ```cs
      // Get the connection string from the environment (dev: see appsettings.Development.json)
      var builder = new SqlConnectionStringBuilder(Configuration.GetConnectionString("AppDb"));

      // Get the userID and password from the environment:
      // (prod: keyvault; dev: secrets.json -- right-click project and choose 'Manage User Secrets')
      var dbCredentials = Configuration.GetSection("DbCredentials");
      builder.UserID = dbCredentials["UserId"];
      builder.Password = dbCredentials["Password"];

      // Registers YourContext as a scoped service.
      // Scoped lifetime => YourContext created once per request:
      // Setting for EnableSqlParameterLogging: prod: turn on/off in environment; dev: see appsettings.Development.json
      services.AddDbContext<YourContext>(options => options.UseSqlServer(builder.ConnectionString)
          .EnableSensitiveDataLogging(Configuration.GetValue<bool>("Logging:EnableSqlParameterLogging")));
  ```

### Create a migration

When necesary, install EF Core 3.0 tools. See: [Announcing Entity Framework Core 3.0](https://devblogs.microsoft.com/dotnet/announcing-ef-core-3-0-and-ef-6-3-general-availability/){:target="_blank"}

- Open **PowerShell** and CD to the **WebApi project folder**
- `dotnet build`
- `dotnet ef migrations add InitialCreate --project ../DataAccess/AppName.DataAccess.csproj --context YourContext`
- `dotnet ef database update --project ../DataAccess/AppName.DataAccess.csproj --context YourContext`

## Add Application Insights

See: [Application Insights for ASP.NET Core applications](https://docs.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core){:target="_blank"}

- In your WebApp/Api projects add the Nuget Package `Microsoft.ApplicationInsights.AspNetCore`
- In Startup.cs ConfigureServices() add:
  `services.AddApplicationInsightsTelemetry();`
- If you connected your WebService to Application Insights, the required configuration settings are already present.
