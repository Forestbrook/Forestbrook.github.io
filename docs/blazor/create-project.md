---
title: Create WebAssembly Solution
parent: Blazor
has_children: false
nav_order: 1
---

_Last update: Septemer 6, 2025_

# Create a Blazor WebAssembly ASP.NET Core hosted solution for .NET 9

You can host a Blazor WebAssembly App as a static website, e.g. in [Azure Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website){:target="_blank"}, but because we use an ASP.NET Core Api, I prefer to use an Api project for hosting. But I do not want a Blazor Server or Blazor Hybrid app. I prefer the WebAssembly app: the code in the browser is running completely independent and only uses Api calls to the server.

Previously, Visual Studio had a template to create an ASP.NET Core hosted Blazor WebAssembly solution.
Unfortunately, this project template is not available anymore, except for an old teplate which sticks to .NET 7 (Blazor WebAssembly App Empty)

## This way you can easily create a Blazor WebAssembly ASP.NET Core hosted solution for .NET 9:

See also the [example app in github](https://github.com/Forestbrook/BlazorWebAssemblyAspNetHosted){:target="_blank"}.

1. Use the Visual Studio template **Blazor WebAssembly Standalone App**, select **.NET 9** and name the project (here: **BlazorExample.Client**).
2. Add a new project to the solution using the template **ASP.NET Core Web API**, select **.NET 9** and **Use controllers** and name the project (here: **BlazorExample.Server**)
3. Add Nuget package to the server project: **Microsoft.AspNetCore.Components.WebAssembly.Server**
4. Add a **reference** in the server project to the client project

5. In the server project edit **Program.cs** and add the lines between the lines:
```cs
public static void Main(string[] args)
{
	var builder = WebApplication.CreateBuilder(args);
	builder.Services.AddControllers();
	// TODO: Add your own services.
	var app = builder.Build();
	// ADD ------------------------------------
	if (app.Environment.IsDevelopment())
		app.UseWebAssemblyDebugging();
	// ------------------------------------
	app.UseHttpsRedirection();
	// ADD ------------------------------------
	app.UseBlazorFrameworkFiles();
	app.UseStaticFiles();
	app.UseRouting();
	// ------------------------------------
	app.UseAuthorization();
	app.MapControllers();
	// ADD ------------------------------------
	app.MapFallbackToFile("index.html");
	// ------------------------------------
	app.Run();
}
```

6. In the server project change Properties/**launchSettings.json** (you can use other port numbers if you like):

```json
{
  "profiles": {
	"http": {
	  "commandName": "Project",
	  "launchBrowser": true,
	  "environmentVariables": {
		"ASPNETCORE_ENVIRONMENT": "Development"
	  },
	  "dotnetRunMessages": true,
	  "inspectUri": "{wsProtocol}://{url.hostname}:{url.port}/_framework/debug/ws-proxy?browser={browserInspectUri}",
	  "applicationUrl": "http://localhost:5225"
	},
	"https": {
	  "commandName": "Project",
	  "launchBrowser": true,
	  "environmentVariables": {
		"ASPNETCORE_ENVIRONMENT": "Development"
	  },
	  "dotnetRunMessages": true,
	  "inspectUri": "{wsProtocol}://{url.hostname}:{url.port}/_framework/debug/ws-proxy?browser={browserInspectUri}",
	  "applicationUrl": "https://localhost:7283;http://localhost:5225"
	},
	"IIS Express": {
	  "commandName": "IISExpress",
	  "launchBrowser": true,
	  "environmentVariables": {
		"ASPNETCORE_ENVIRONMENT": "Development"
	  },
	  "inspectUri": "{wsProtocol}://{url.hostname}:{url.port}/_framework/debug/ws-proxy?browser={browserInspectUri}"
	}
  },
  "iisExpress": {
	"applicationUrl": "http://localhost:26936",
	"sslPort": 44351
  },
  "iisSettings": {
	"windowsAuthentication": false,
	"anonymousAuthentication": true,
	"iisExpress": {
	  "applicationUrl": "http://localhost:65397/",
	  "sslPort": 44373
	}
  }
}
```
