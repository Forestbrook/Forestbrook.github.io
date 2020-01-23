---
title: Localize WebApp and Api
parent: ASP.NET Core
has_children: false
nav_order: 1
---

<i>Last update: January 23, 2020</i><div style="text-align: right">Source code in Git:</div>

|_Last update: January 23, 2020_|Source code in Git: [ASP.NET Localization Example](todo){:target="_blank"}|

## Localization of a MVC WebApplication and an Api

See also: [Globalization and localization in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization){:target="_blank"}

**The example is based on an ASP.NET Core 3 WebApplication (MVC) project.**

## Create a resource library project

I like to put my resources in a separate resource library project to be able to use them in multiple projects. I use a **netstandard** project type. This way, the resources can also be used in Xamarin, Wpf and maybe in the future Blazor projects.

1. Select the solution in Visual Studio, right click and choose **Add** > **New Project...**
1. Choose the **Class Library (.NET Standard)** template and choose **Next**.
1. Enter your project name, e.g. **Example.ResourceLibrary** and choose **Create**.
1. You can set the **TargetFramework** of the project to **netstandard2.1**.
1. Remove **Class1.cs**.
1. Create a **Resources** folder (not required, you can also put your resources in the root).
1. I use only one set of shared resources, but if you like you can also add view and controller resources as described in [Resource file naming](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/localization#resource-file-naming){:target="_blank"}.
1. Add resource files for the languages you support like **SharedResource.en.resx** and **SharedResource.nl-NL.resx**: Right-click on the Resources folder, choose **Add** > **New item...** and select the **Resources File** template.
1. In the project root add an empty marker class **SharedResource** like this:
   ```cs
    namespace Example.ResourceLibrary
    {
        /// <summary>
        /// REMARK: Class cannot be static, because it is used as a marker class to load the resources in a IStringLocalizer
        /// </summary>
        public class SharedResource
        {
        }
    }
   ```
1. Add your translated texts in the resource files (e.g. Name=**Welcome**, Value=**Welkom** in nl-NL).
1. Add a reference to the **Example.ResourceLibrary** in your Web and/or Api projects.

## Startup.cs configuration in your Web and/or Api projects

```cs
using System.Globalization;
using Microsoft.AspNetCore.Localization;
using Example.ResourceLibrary;
```

Add in ConfigureServices():
```cs
// REMARK: You can leave out the options if you put the resources in the root instead of the Resources folder.
services.AddLocalization(options => options.ResourcesPath = "Resources");

// This line is only needed to be able to lookup your localization options in a controller, view or service:
services.AddSingleton(CreateRequestLocalizationOptions());

// Change existing line:
services.AddControllersWithViews() // or AddControllers(), AddMvc(), etc.
    // Add this line if you want to use your shared resources in the data annotations:
    .AddDataAnnotationsLocalization(options => { options.DataAnnotationLocalizerProvider = (type, factory) => factory.Create(typeof(SharedResource)); })
    // Add this line to be able to use the IHtmlLocalizer in your views:
    .AddViewLocalization(LanguageViewLocationExpanderFormat.SubFolder)
    ...
    ;
```

Add in Configure() **_Before app.UseRouting (or UseMvc)_**:
```cs
app.UseRequestLocalization(app.ApplicationServices.GetService<RequestLocalizationOptions>());
// Or if you like this better:
app.UseRequestLocalization(CreateRequestLocalizationOptions());
```

Add a private static method **CreateRequestLocalizationOptions**:

```cs
private static RequestLocalizationOptions CreateRequestLocalizationOptions()
{
    var supportedLanguages = new[] { new CultureInfo("nl-NL"), new CultureInfo("en") };
    var supportedFormattingCultures = new[] { new CultureInfo("nl-NL"), new CultureInfo("en-US") };
    var result = new RequestLocalizationOptions
    {
        DefaultRequestCulture = new RequestCulture("nl-NL", "nl-NL"),
        SupportedCultures = supportedFormattingCultures,
        SupportedUICultures = supportedLanguages
    };

    // Now if your default browser language is English, your WebApplication will startup in English,
    // even though you set the DefaultRequestCulture to Dutch (Netherlands).
    // In most situations this is correct, but if you DO want to start in the language specified in DefaultRequestCulture
    // you can add these lines:
    var acceptLanguageProvider = result.RequestCultureProviders.FirstOrDefault(p => p is AcceptLanguageHeaderRequestCultureProvider);
    if (acceptLanguageProvider != null)
        result.RequestCultureProviders.Remove(acceptLanguageProvider);

    return result;
}
```

## Use dependency injection to get the resource values in your view, controller or service

In the **HomeController**:
```cs
using Microsoft.Extensions.Localization;
using Example.ResourceLibrary;
...
public HomeController(IStringLocalizer<SharedResource> localizer)
```

Or in **Index.cshtml**:
```html
@using Example.ResourceLibrary
@using Microsoft.AspNetCore.Mvc.Localization
@inject IHtmlLocalizer<SharedResource> SharedLocalizer
...
<h1 class="display-4">SharedLocalizer[Welcome]</h1>
```

## Let the user change the language

In the **Views/Shared** folder of the WebApplication add a partial view **_SelectLanguagePartial.cshtml**:
```cs
@using Example.ResourceLibrary
@using Microsoft.AspNetCore.Builder
@using Microsoft.AspNetCore.Localization
@using Microsoft.Extensions.Localization
@inject RequestLocalizationOptions RequestLocalizationOptions
@inject IStringLocalizer<SharedResource> SharedLocalizer

@{
    var uiCulture = Context.Features.Get<IRequestCultureFeature>().RequestCulture.UICulture;
    var language = uiCulture.Name;
    var languageNativeName = uiCulture.NativeName;
    var cultureItems = RequestLocalizationOptions.SupportedUICultures
        .Select(c => new SelectListItem { Value = c.Name, Text = c.NativeName })
        .ToList();

    // TODO: Add Context.Request.QueryString if necessary
    var returnUrl = string.IsNullOrEmpty(Context.Request.Path) ? "~/" : $"~{Context.Request.Path.Value}";

<div title="@SharedLocalizer["Select Language"] (@languageNativeName)">
    <form asp-controller="Home" asp-action="SetLanguage" asp-route-returnUrl="@returnUrl" method="post" role="form">
        <select class="form-control" name="language" onchange="this.form.submit();" asp-for="@language" asp-items="cultureItems">
        </select>
    </form>
</div>
}
```

In Views/Shared/**_Layout.cshtml** add this code:
```html
<!-- Existing code: -->
<div class="navbar-collapse collapse d-sm-inline-flex flex-sm-row-reverse">
    <!-- Code to add: -->
    <ul class="navbar-nav">
        <li class="nav-item">
            <partial name="_SelectLanguagePartial" />
        </li>
    </ul>
```

Now for Dutch the dropdown list shows **Nederlands (Nederland)**. If you want it to show only **Nederlands** there are two possible solutions:

1. Change the name of the resource file to **SharedResource.nl.resx** and use `new CultureInfo("nl")` for the supported language.

1. Or use a helper method like this:
```cs
public static string GetNativeName(this CultureInfo cultureInfo)
    => cultureInfo.IsNeutralCulture ? cultureInfo.NativeName : cultureInfo.Parent.NativeName;
```

## Store the selected language in a Culture Cookie

In the **HomeController** inject the **RequestLocalizationOptions** which you added as a singleton in Startup:
```cs
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Localization;

private readonly RequestLocalizationOptions _localizationOptions;
public HomeController(RequestLocalizationOptions localizationOptions)
{
    _localizationOptions = localizationOptions;
}
```

Add the **SetLanguage** action to which _SelectLanguagePartial posts. This will create a culture cookie which will store the selected language:
```cs
[HttpPost]
public IActionResult SetLanguage(string language, string returnUrl)
{
    var defaultFormattingCulture = _localizationOptions.DefaultRequestCulture.Culture.Name;
    var formattingCulture = defaultFormattingCulture;
    if (HttpContext.Request.Cookies.TryGetValue(CookieRequestCultureProvider.DefaultCookieName, out var value))
    {
        formattingCulture = CookieRequestCultureProvider.ParseCookieValue(value).Cultures.FirstOrDefault().Value;
        if (string.IsNullOrEmpty(formattingCulture))
            formattingCulture = defaultFormattingCulture;
    }

    value = CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(formattingCulture, language));
    var expiration = new CookieOptions { Expires = DateTime.UtcNow.AddYears(4) };
    HttpContext.Response.Cookies.Delete(CookieRequestCultureProvider.DefaultCookieName);
    HttpContext.Response.Cookies.Append(CookieRequestCultureProvider.DefaultCookieName, value, expiration);
    return LocalRedirect(returnUrl);
}
```
