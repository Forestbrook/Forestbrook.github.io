---
title: ASP.NET App Service
parent: Azure
has_children: false
nav_order: 1
---

## Configure and create an ASP.NET Core WebApi or WebApp in an App Service
{: .no_toc }

A professional Azure ASP.NET Core WebApi or WebApp requires several Azure resources.
This article is intented to keep track of the setup and configuration of these resources and how they are integrated in the application.
Involved resources:

* Azure App Service Plan and App Service
* Azure Key Vault
* Azure SQL Server and SQL database
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
  - Value: `Data Source=sqlServerName.database.windows.net;Initial Catalog=DatabaseName;Connect Timeout=30;Encrypt=True;TrustServerCertificate=False;ApplicationIntent=ReadWrite;MultiSubnetFailover=False`
  - Type: **SQLServer**

### Enable the WebApi/WebApp to use the KeyVault with its Managed Identity
In the Key Vault, select the **Access policies** section and click **+ Add Access Policy**
- Secret permissions: Get, List
- Select principal: Enter the name (or Identity object ID) of the app.
- Save with **Add** and click **Save** (upper left) to commit the changes.
