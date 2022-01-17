---
title: Azure WebApi and WebApp
parent: Azure
has_children: false
nav_order: 1
---

## Configure and create an ASP.NET 6 WebApi or WebApp in an Azure App Service
{: .no_toc }

_Last update: January 17, 2022_<br/>
Source code in Git: [Azure WebApi and WebApp with KeyVault](https://github.com/Forestbrook/WebApiWithKeyVault){:target="_blank"}

A professional Azure ASP.NET 6 WebApi or WebApp requires several Azure resources.
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
- In **Networking** > **Firewall rules** turn on **Allow Azure services and resources to access this server**
- **After creation**: If you need access to the databases with e.g. SQL Server Management Studio, select the **Firewalls and virtual networks** tab and add your own ip address(es) as a Rule (your own ip address is shown at **Client IP address**).

### Create the SQL database
- Create a SQL Database in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **SQL databases**)
- Make sure to select the correct **Subscription**, **Resource group** and **Server** and enter your **Database name**.
- At **Compute + storage** > **Configure** select the right pricing model. _The default is quite expensive._ If your database is not too big, start with Service Tier **Basic** or **Standard**, or choose **Serverless**.
- In Additional settings you can select the Database Collation (sort/compare rules).
- Make sure to turn off any free trials you do not want in Additional settings.

### Create a (Blob) Storage Account
- Create a Storage Account in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **Storage accounts**)
- Make sure to select the correct **Subscription**, **Resource group** and **Location**. Set a **Storage account name** (lowercase only), **Performance** (Standard), **Redundancy** _(LRS or GRS)_.
- In the **Advanced** tab Check the **Security** settings.
- In the **Data protection** tab check the **Recovery** and **Tracking** options. Enabeling Soft Delete and Versioning is a safe way to prevent accidental data loss, but if you need to delete blobs and imediately after that create a new blob with the same name, this might fail (according to the documentation there needs to be a short delay).
- Select **Review + create**, verify the selected settings and select **Create**.
- Wait until **Your deployment is complete** is shown, then select **Go to resource**.
- In the Storage account left menu select **Access keys**. Copy the (key 1) **Connection string** (the connection string includes the key) and store it in the KeyVault as **Secret**. Secret name (used in this article): **StorageCredentials--ConnectionString**. You can access it in your app later like `Configuration.GetValue<string>("StorageCredentials:ConnectionString")`.

### Create the App Service (Web App) for the WebApi or WebApp
- Create an **App Service** in the [Azure Portal](https://portal.azure.com){:target="_blank"} (search for **App services**)
- Make sure to select the correct **Subscription**, **Resource Group**, **Runtime stack** (.NET 6), **Operating System**, **Region** and **App Service Plan** (click _Create new_ to specify a name for a new App Service Plan).
- At **Sku and size** click _Change size_ to see options and pricing. I did have some trouble to upgrade a S1 tier to a P1V2 tier, so you might want to start with a more powerfull V2 tier.
- In the **Monitoring** tab: When needed, enable Application Insights and select the correct one (or create a new one). This will generate all **configuration settings** for Application Insights, but make sure you add `builder.Services.AddApplicationInsightsTelemetry();` in Program.cs!

After creation:
- TLS/SSL settings section: **HTTPS Only** and Minimum TLS Version: **1.2**
- In the **Configuration** section under **Application settings** add:
  - Name: **ASPNETCORE_ENVIRONMENT**, Value: **Production** (you can replace that with Development for debugging or with Staging)
  - Optional Name: **WEBSITE_TIME_ZONE**, Value: your time zone (e.g. W. Europe Standard Time)
  - Optional you might need for Azure AD-B2C (add only if you are sure you need this) Name: **WEBSITE_LOAD_USER_PROFILE**, Value: 1
- In the **Configuration** section under **Connection strings** add the connection string for your database (replace _--your-SQL-Server-name--_ and _--your-database-name--_ with the correct name):
  - Name (used in this article): **AppDb**
  - Value: `Data Source=--your-SQL-Server-name--.database.windows.net;Initial Catalog=--your-database-name--; Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False`
  - Type: **SQLServer**
  - _Warning:_ Do NOT include the SQL Server login and password in the connection string! These are stored in the Key Vault.
Mind that the optional configuration settings and the connection strings can also be specified in the **appsettings.json** file of your WebApp or WebApi. The values you specify here in the Configuration section will override the values in appsettings.json.

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
- In `Program.cs` you can use `builder.Environment.IsDevelopment(), .IsProduction(), .IsStaging()`.

## Create the project solution

- In Visual Studio create an **ASP.NET Core Web App**, **ASP.NET Core Web Api** or **ASP.NET Core Web App (Model-View-Controller)**.
- In the Additional Information step select the Framework (.NET 6) and make sure **Configure for HTTPS** is selected.
- Authentication: Will be added later. See Azure AD B2C article.

_Remark:_ In the source code examples I assume `<ImplicitUsings>enable</ImplicitUsings>` is set in the .csproj of the project.

Add the class **ConfigurationKeys.cs**:
   ```cs
   namespace Forestbrook.WebApiWithKeyVault;

   public static class ConfigurationKeys
   {
       public const string DatabaseConnectionString = "AppDb";
       public const string DatabasePassword = "DbCredentials:Password";
       public const string DatabaseUserId = "DbCredentials:UserId";
       public const string EnableSqlParameterLogging = "Logging:EnableSqlParameterLogging";
       public const string KeyVaultName = "KeyVaultName";
       public const string KeyVaultTenantId = "KeyVaultTenantId";
       public const string StorageConnectionString = "StorageCredentials:ConnectionString";
   }
   ```

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
   using Azure.Core;
   using Azure.Identity;

   namespace Forestbrook.WebApiWithKeyVault;

   public static class AspNetCoreHelper
   {
       /// <summary>
       /// Make sure to add to your appsettings.json: "KeyVaultName": "your-key-vault-name"
       /// </summary>
       public static void AddAzureKeyVault(this WebApplicationBuilder builder)
       {
           var keyVaultUri = builder.Configuration.CreateKeyVaultUri();
           var keyVaultCredential = builder.Configuration.CreateKeyVaultCredential();
           builder.Configuration.AddAzureKeyVault(keyVaultUri, keyVaultCredential);
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

4. In **Program.cs** directly after `var builder = WebApplication.CreateBuilder(args);` add:
   ```cs
   builder.AddAzureKeyVault();
   ```

### Add a Storage Repository

If you created a (Blob) Storage Account above, you can implement a Storage Repository.

1. Example repository:
   ```cs
   namespace Forestbrook.WebApiWithKeyVault;

   public class BlobStorageRepository
   {
       public BlobStorageRepository(IConfiguration configuration)
       {
           if (configuration == null) throw new ArgumentNullException(nameof(configuration));
           var storageConnectionString = configuration.GetValue<string>(ConfigurationKeys.StorageConnectionString);
           // TODO: Implement your repository using the Nuget Package Azure.Storage.Blobs
           //_blobServiceClient = new BlobServiceClient(storageConnectionString);
       }
   }
   ```

2. In **Program.cs** at 'Add services to the container:' add this line:
   ```cs
   builder.Services.AddSingleton<BlobStorageRepository>();
   ```

### Add a SQL Server Database Repository

If you created a SQL Database above, you can implement a SQL Server Database Repository.

1. Add Nuget package **System.Data.SqlClient**

2. In the extensions class **AspNetCoreHelper.cs** add:
   ```cs
   using System.Data.SqlClient;

   public static string CreateSqlConnectionString(this IConfiguration configuration)
   {
       var dbUserId = configuration.GetValue<string>(ConfigurationKeys.DatabaseUserId);
       var dbPassword = configuration.GetValue<string>(ConfigurationKeys.DatabasePassword);
       var builder = new SqlConnectionStringBuilder(configuration.GetConnectionString(ConfigurationKeys.DatabaseConnectionString)) { UserID = dbUserId, Password = dbPassword };
       return builder.ConnectionString;
   }
   ```

3. Example repository: -- Skip this step if you want to use the Entity Framework (see below).
   ```cs
   namespace Forestbrook.WebApiWithKeyVault;

   public class DatabaseRepository
   {
       public DatabaseRepository(IConfiguration configuration)
       {
           if (configuration == null) throw new ArgumentNullException(nameof(configuration));
           var connectionString = configuration.CreateSqlConnectionString();
           // TODO: Implement an SQL Server or Entity Framework repository.
       }
   }
   ```

3. To enable local testing and debugging, add the database connection string which you added to your App Service Configuration above to **appsettings.Development.json**:
   ```json
  "ConnectionStrings": {
      "AppDb": "Data Source=--your-SQL-Server-name--.database.windows.net;Initial Catalog=--your-database-name--;Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False"
  },
   ```

4. In **Program.cs** at 'Add services to the container:' add this line:
   ```cs
   builder.Services.AddScoped<DatabaseRepository>();
   ```

## Add Entity Framework

### DataContext and Models

- Create a .NET Core class library --your-app-name--.**Models**.
- Add the Models for the database tables.
- Create a .NET Core class library --your-app-name--.**DataAccess**.
- In the Data folder Create the DbContext with the DbSets for the tables.

### Register DbContext in Program.cs

1. Add Nuget package **Microsoft.EntityFrameworkCore.SqlServer**

2. In the extensions class **AspNetCoreHelper.cs** add:
   ```cs
   using Microsoft.EntityFrameworkCore;
   using Microsoft.EntityFrameworkCore.Diagnostics;

   public static void AddDbContext<TContext>(this WebApplicationBuilder builder) where TContext : DbContext
   {
       // Registers TContext as a scoped service (created once per request):
       // TODO: See for EnableRetryOnFailure(): Connection Resiliency https://docs.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency
       builder.Services.AddDbContext<TContext>(BuildContextOptions);

       void BuildContextOptions(DbContextOptionsBuilder contextOptionsBuilder)
       {
           contextOptionsBuilder.ConfigureWarnings(w =>
           {
    #if DEBUG
               w.Ignore(CoreEventId.SensitiveDataLoggingEnabledWarning);
    #endif
               // Ignore warnings caused by QuerySplittingBehavior.SplitQuery:
               w.Ignore(CoreEventId.RowLimitingOperationWithoutOrderByWarning);
           });

           var connectionString = builder.Configuration.CreateSqlConnectionString();
           contextOptionsBuilder.UseSqlServer(connectionString, providerOptions =>
           {
               providerOptions.EnableRetryOnFailure();
               providerOptions.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
           });

           if (builder.Configuration.GetValue<bool>(ConfigurationKeys.EnableSqlParameterLogging))
               contextOptionsBuilder.EnableSensitiveDataLogging();
       }
   }
   ```

3. In **Program.cs** at 'Add services to the container:' add this line:
   ```cs
   builder.AddDbContext<--your-dbcontext-->();
   ```

### Create a migration

When necesary, install EF Core tools. See: [Entity Framework Core tools reference](https://docs.microsoft.com/en-us/ef/core/miscellaneous/cli/dotnet){:target="_blank"}

- Open **Command Prompt** and CD to the **WebApp or WebApi project folder**
- `dotnet build`
- `dotnet ef migrations add InitialCreate --project ../DataAccess/--your-app-name--.DataAccess.csproj --context YourContext`
- `dotnet ef database update --project ../DataAccess/--your-app-name--.DataAccess.csproj --context YourContext`

## Add Application Insights

See: [Application Insights for ASP.NET Core applications](https://docs.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core){:target="_blank"}

- In your WebApp/Api projects add the Nuget Package `Microsoft.ApplicationInsights.AspNetCore`
- In **Program.cs** at 'Add services to the container:' add this line:
  `builder.Services.AddApplicationInsightsTelemetry();`
- If you connected your WebService to Application Insights, the required configuration settings are already present.

## Local testing/debugging in Visual Studio

When you run your App from Visual Studio on your local computer in IIS Express for debugging, the App can magically connect to the KeyVault. This magic happens in the `DefaultAzureCredential()` which was connected to the KeyVault Secrets configuration provider added in AspNetCoreHelper.

See [Authenticating via Visual Studio](https://docs.microsoft.com/en-us/dotnet/api/overview/azure/identity-readme#authenticating-via-visual-studio){:target="_blank"}

To make this work, your Microsoft Account must have at least _Get_ and _List_ access to the KeyVault!

### Issues

1. You might have to tell Visual Studio your Azure tennant ID by setting the environment variable **AZURE_TENANT_ID**
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

2. If you have solutions for more than one Azure tennants, it is quite annoying to change the AZURE_TENANT_ID environment variable. That is why I added the ConfigurationKey **KeyVaultTenantId**. You can store the KeyVaultTenantId=--your-tennant--id-- in UserSecrets (in Visual Studio right-click your WebApi-project and choose **Manage User Secrets**) or in appsettings.Development.json.

3. If you have multiple Microsoft accounts connected to Visual Studio, you might have to tell Visual Studio which account to use:
    * Select Tools > Options...
    * Go to option **Azure Service Authentication** > **Account Selection**
    * Choose an account.
    * Alternatively, you can set the environment variable **AZURE_USERNAME**

4. If your account is not configured correctly, you will get an `Azure.Identity.AuthenticationFailedException`.

See also: [DefaultAzureCredential fails when multiple accounts are available and defaulting to SharedTokenCacheCredential](https://github.com/Azure/azure-sdk-for-net/issues/8658){:target="_blank"}.
