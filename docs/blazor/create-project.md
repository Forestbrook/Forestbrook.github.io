---
title: Create Project
parent: Blazor
has_children: false
nav_order: 1
---

_Last update: January 23, 2020_

## Create a new Blazor WebAssembly project with an ASP.NET Core Api

You can host a Blazor WebAssembly App as a static website, e.g. in [Azure Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website){:target="_blank"}, but because we use an ASP.NET Core Api, I prefer to use the Api project for hosting. Mind that this is **NOT** the same as a Blazor Server App!

## Preparations

* See for more details: [Get started with ASP.NET Core Blazor (in Microsoft Docs)](https://docs.microsoft.com/en-us/aspnet/core/blazor/get-started){:target="_blank"}.

* When you have the latest version of Visual Studio installed and up-to-date, the required .NET Core 3.1 (or later) SDK should be already present. But because Blazor WebAssembly is in preview, you must install the template for the Blazor WebAssembly App (_you might even need to reinstall it after a Visual Studio update_). Run in a command prompt:

   ```
   dotnet new -i Microsoft.AspNetCore.Blazor.Templates::3.1.0-preview4.19579.2
   ```

## Create the solution

1. In Visual Studio choose **Create a new project**
1. Choose **Blazor App**
1. Choose your project/solution name like **BlazorExample** and select **Create**.
1. Select the **Blazor WebAssembly App** template (if not present, see [Preparations](#Preparations) above).
1. Make sure to check **ASP.NET Core hosted** at the right and select **Create**.

## Reorganize the solution

Three projects are created. In my examples I use a different naming convention, so I do some renaming of the projects. Don't forget to rename the namespace!

1. **BlazorExample.Server**. This will be our Api, so I rename it to **BlazorExample.Api**.
1. **BlazorExample.Client**. I like to rename this to **BlazorExample.WebApp**.
1. **BlazorExample.Shared**. I like to rename this to **BlazorExample.Common**.
1. Set the **RootNamespace** of the BlazorExample.Common project to **BlazorExample**.
1. In BlazorExample.Common add a folder **Models** and move the generated **WeatherForecast** class to that folder. Change the namespace to **BlazorExample.Models**.

