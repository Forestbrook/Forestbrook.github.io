---
title: Localization
parent: Blazor
has_children: false
nav_order: 2
---

_Last update: Februari 3, 2020_<br/>
Source code in Git: [Blazor Localization Example](https://github.com/Forestbrook/BlazorLocalizationExample){:target="_blank"}

# Blazor Localization
{: .no_toc }

This solution:
* Is intended for [Blazor WebAssembly](https://docs.microsoft.com/en-us/aspnet/core/blazor/hosting-models#blazor-webassembly){:target="_blank"}. 
* Reads the translations from your Api when the Blazor app is started and when the language is changed.
* Can be used with translations in **.resx files**, in a **translations database** or with **any other source of translations**.
* Using translations on a page is just as easy as inserting **`@Translation["Welcome"]`**.
* When the language is changed, pages with translations are automatically updated.
* Implements a `<SelectLanguage />` Razor component with a drop-down list to select the language.
* The selected language code is stored in an ASP.NET Culture Cookie, but if you prefer you can also store the selected language code in the browsers local storage and use a `ui-culture` query parameter or an Accept-Language header in the Api calls.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Introduction

When I started to move our ASP.NET Core MVC website to Blazor, one of the missing pieces was Localization.

The current preview version of Blazor WebAssembly cannot handle localized resources (.resx) with satellite assemblies.

I found several solutions on the web with workarounds for this (see [References](#references) section). I expect the Blazor team to implement the usage of .resx resources in satellite assemblies sooner or later, so created this solution, which keeps using my existing shared resource library project.

## Create the project

1. Create the initial BlazorExample solution as described in the knowledgebase article [Create Blazor WebAssembly project with an ASP.NET Core Api](create-project).

1. Add a resource library project **BlazorExample.ResourceLibrary**. See description [Create a resource library project](/docs/asp-net-core/localize-app-and-api#create-a-resource-library-project). _Skip this step if you want to use a database or other source of translations instead of .resx files._

1. Edit Startup.cs in **BlazorExample.Api** to use the resource library as described in [Startup.cs configuration in your Web and/or Api projects](/docs/asp-net-core/localize-app-and-api#startupcs-configuration-in-your-web-andor-api-projects).
You can skip `AddDataAnnotationsLocalization` and `AddViewLocalization`.
_If you want to use a database or other source of translations instead of .resx files, make sure to add a service to load the translations in your Startup.cs._

## Create the GetLanguage and SetLanguage Api actions

1. In the BlazorExample.Common project, add a **LanguageResources** class in the **Models** folder:
   ```cs
    using System.Collections.Generic;

    namespace BlazorExample.Models
    {
        public class LanguageResources
        {
            /// <summary>
            /// Language culture code like "en-US" or "nl-NL"
            /// </summary>
            public string Language { get; set; }

            public IReadOnlyCollection<string> AvailableLanguages { get; set; }

            public IReadOnlyDictionary<string, string> Translations { get; set; }
        }
    }
   ```

1. In the BlazorExample.Common project, add a **LanguageModel** class in the **Models** folder:
   ```cs
    namespace BlazorExample.Models
    {
        public class LanguageModel
        {
            /// <summary>
            /// Culture code like "en-US" or "nl-NL"
            /// </summary>
            public string CultureName { get; set; }
        }
    }
   ```

1. In the BlazorExample.Api project, add a **LocalizationHelper** class:
   ```cs
    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore.Http;
    using Microsoft.AspNetCore.Localization;
    using System;
    using System.Globalization;
    using System.Linq;

    namespace BlazorExample.Api
    {
        public static class LocalizationHelper
        {
            public static string GetDefaultLanguage(this RequestLocalizationOptions localizationOptions)
                => localizationOptions.DefaultRequestCulture.UICulture.Name;

            public static string GetDefaultFormattingCulture(this RequestLocalizationOptions localizationOptions)
                => localizationOptions.DefaultRequestCulture.Culture.Name;

            public static string GetLanguageFromCookie(this HttpContext httpContext)
            {
                if (httpContext == null)
                    return null;

                if (!httpContext.Request.Cookies.TryGetValue(CookieRequestCultureProvider.DefaultCookieName, out var value))
                    return null;

                return CookieRequestCultureProvider.ParseCookieValue(value).UICultures.FirstOrDefault().Value;
            }

            public static CultureInfo GetRequestUICulture(this HttpContext httpContext)
                => httpContext.Features.Get<IRequestCultureFeature>().RequestCulture.UICulture;

            public static bool IsLanguageSupported(this RequestLocalizationOptions localizationOptions, string cultureName)
                => localizationOptions.SupportedUICultures.Any(l => l.Name == cultureName);

            public static void SetLanguageCookie(this HttpContext httpContext, string language, string defaultFormattingCulture)
            {
                if (httpContext == null) throw new ArgumentNullException(nameof(httpContext));
                if (!httpContext.Request.Cookies.TryGetValue(CookieRequestCultureProvider.DefaultCookieName, out var value))
                {
                    SetCultureCookie(httpContext, language, defaultFormattingCulture);
                    return;
                }

                var formattingCulture = CookieRequestCultureProvider.ParseCookieValue(value).Cultures.FirstOrDefault().Value;
                if (string.IsNullOrEmpty(formattingCulture))
                    formattingCulture = defaultFormattingCulture;

                SetCultureCookie(httpContext, language, formattingCulture);
            }

            private static void SetCultureCookie(HttpContext httpContext, string language, string formattingCulture)
            {
                var value = CookieRequestCultureProvider.MakeCookieValue(new RequestCulture(formattingCulture, language));
                var expiration = new CookieOptions { Expires = DateTime.UtcNow.AddYears(4) };
                httpContext.Response.Cookies.Delete(CookieRequestCultureProvider.DefaultCookieName);
                httpContext.Response.Cookies.Append(CookieRequestCultureProvider.DefaultCookieName, value, expiration);
            }
        }
    }
   ```

1. In the BlazorExample.Api project, add a **SettingsController** class:
   ```cs
    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Extensions.Localization;
    using System.Linq;
    using BlazorExample.Models;
    using BlazorExample.ResourceLibrary;

    namespace BlazorExample.Api.Controllers
    {
        [Route("api/[controller]")]
        [ApiController]
        public class SettingsController : ControllerBase
        {
            private readonly RequestLocalizationOptions _localizationOptions;
            private readonly IStringLocalizer _localizer;

            public SettingsController(IStringLocalizer<SharedResource> localizer, RequestLocalizationOptions localizationOptions)
            {
                _localizer = localizer;
                _localizationOptions = localizationOptions;
            }

            // GET api/settings/language
            [HttpGet("language")]
            public LanguageResources GetLanguage()
            {
                return new LanguageResources
                {
                    Language = HttpContext.GetRequestUICulture().Name,
                    AvailableLanguages = _localizationOptions.SupportedUICultures.Select(l => l.Name).ToList(),
                    Translations = _localizer.GetAllStrings(true).ToDictionary(ls => ls.Name, ls => ls.Value)
                };
            }

            // POST api/settings/language
            [HttpPost("language")]
            public IActionResult SetLanguage(LanguageModel model)
            {
                // If model.CultureName == null use current language cookie or create one with the default language:
                if (string.IsNullOrEmpty(model.CultureName))
                {
                    // Check for valid language cookie:
                    var cultureName = HttpContext.GetLanguageFromCookie();
                    if (_localizationOptions.IsLanguageSupported(cultureName))
                        return Ok();

                    model.CultureName = _localizationOptions.GetDefaultLanguage();
                }

                if (!_localizationOptions.IsLanguageSupported(model.CultureName))
                    return BadRequest($"Unsupported language: {model.CultureName}");

                HttpContext.SetLanguageCookie(model.CultureName, _localizationOptions.GetDefaultFormattingCulture());
                return Ok();
            }
        }
    }
   ```

## Add some abstractions

I like to use interfaces, but that is not a requirement. If you better like it, just use the full classes described in the [Create client helper services](#create-client-helper-services) section.

1. In the BlazorExample.Common project create folder **Abstractions**
1. Add an **ITranslationProvider** interface:
   ```cs
    using System;

    namespace BlazorExample.Abstractions
    {
        public interface ITranslationProvider
        {
            event EventHandler LanguageChanged;

            string this[string name] { get; }

            string this[string name, params object[] arguments] { get; }
        }
    }
   ```
1. Add an **ITranslationService** interface:
   ```cs
    using System.Collections.Generic;
    using System.Globalization;

    namespace BlazorExample.Abstractions
    {
        public interface ITranslationService
        {
            IReadOnlyCollection<CultureInfo> AvailableLanguages { get; }

            CultureInfo SelectedLanguage { get; }

            void ChangeLanguage(string languageCultureName);
        }
    }
   ```
1. Add an **ILanguageLoader** interface:
   ```cs
    using BlazorExample.Models;
    using System;

    namespace BlazorExample.Abstractions
    {
        public interface ILanguageLoader
        {
            void StartLoadLanguage(string languageCultureName, Action<LanguageResources> setLanguage);
        }
    }
   ```

## Create client helper services

We need some simple services and components at the client side to load and use the translations. In the BlazorLocalizationExample solution I use a separate Client Razor Class Library, but you can also just put them in the BlazorExample.WebApp project.

1. Create a Razor Class Library **BlazorExample.Client**. You might have to change the TargetFramework to **netstandard2.1** (or the netstandard version you need).

1. Add the Nuget package **Microsoft.AspNetCore.Blazor.HttpClient** (make sure to use the same version as in BlazorExample.WebApp).

1. Add references to the **BlazorExample.Common**

1. Add a folder **Localization**

1. In the folder add a new class **TranslationService**:
   ```cs
    using System;
    using System.Collections.Generic;
    using System.Globalization;
    using System.Linq;
    using BlazorExample.Abstractions;
    using BlazorExample.Models;

    namespace BlazorExample.Client.Localization
    {
        public class TranslationService : ITranslationProvider, ITranslationService
        {
            private readonly ILanguageLoader _languageLoader;
            private IReadOnlyDictionary<string, string> _translations;

            public TranslationService(ILanguageLoader languageLoader)
            {
                _languageLoader = languageLoader;
                _languageLoader.StartLoadLanguage(null, LoadLanguage);
            }

            public event EventHandler LanguageChanged;

            public IReadOnlyCollection<CultureInfo> AvailableLanguages { get; private set; }

            public CultureInfo SelectedLanguage { get; private set; } = CultureInfo.InvariantCulture;

            public string this[string name] => GetString(name);

            public string this[string name, params object[] arguments] => GetString(name, arguments);

            public void ChangeLanguage(string languageCultureName)
            {
                if (string.IsNullOrEmpty(languageCultureName)) throw new ArgumentNullException(nameof(languageCultureName));
                if (AvailableLanguages.All(ci => ci.Name != languageCultureName)) throw new ArgumentException($"Unsupported language: {languageCultureName}");
                if (languageCultureName != SelectedLanguage.Name)
                    _languageLoader.StartLoadLanguage(languageCultureName, LoadLanguage);
            }

            private void LoadLanguage(LanguageResources resources)
            {
                _translations = resources.Translations;
                SelectedLanguage = new CultureInfo(resources.Language);
                AvailableLanguages = resources.AvailableLanguages.Select(n => new CultureInfo(n)).ToList();
                LanguageChanged?.Invoke(this, EventArgs.Empty);
            }

            private string GetString(string name, params object[] arguments)
            {
                if (_translations == null || !_translations.TryGetValue(name, out var value))
                    value = name;

                if (arguments.Length > 0)
                    value = string.Format(SelectedLanguage, value, arguments);

                return value;
            }
        }
    }
   ```
1. Add a new class **LanguageLoader**:
   ```cs
    using BlazorExample.Abstractions;
    using BlazorExample.Models;
    using Microsoft.AspNetCore.Components;
    using System;
    using System.Net.Http;
    using System.Threading.Tasks;

    namespace BlazorExample.Client.Localization
    {
        public class LanguageLoader : ILanguageLoader
        {
            private const string LanguageApiRequestUri = "api/settings/language";
            private readonly HttpClient _httpClient;

            public LanguageLoader(HttpClient httpClient)
            {
                _httpClient = httpClient;
            }

            public async void StartLoadLanguage(string languageCultureName, Action<LanguageResources> setLanguage)
            {
                var languageResources = await SelectAndLoadLanguage(languageCultureName);
                setLanguage(languageResources);
            }

            private async Task<LanguageResources> SelectAndLoadLanguage(string languageCultureName)
            {
                // Post the selected language (null to use current default) to update the language cookie:
                await _httpClient.PostJsonAsync(LanguageApiRequestUri, new LanguageModel { CultureName = languageCultureName }).ConfigureAwait(false);

                // Load the selected language texts:
                return await _httpClient.GetJsonAsync<LanguageResources>(LanguageApiRequestUri).ConfigureAwait(false);
            }
        }
    }
   ```
1. Add references to the BlazorExample.Client project in the **BlazorExample.WebApp**

## Create a base class to support Translations for your Pages and Layouts

1. In the BlazorExample.Client project add a folder **Components**.

1. In the Components folder, add a new class **ComponentWithTranslations.cs**. I use the [Dispose pattern](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose){:target="_blank"}, so if you also need to dispose in page, you can override `Dispose(bool disposing)` (don't forget to call `base.Dispose(disposing)`:
   ```cs
    using BlazorExample.Abstractions;
    using Microsoft.AspNetCore.Components;
    using System;

    namespace BlazorExample.Client.Components
    {
        public abstract class ComponentWithTranslations : ComponentBase, IDisposable
        {
            [Inject]
            protected ITranslationProvider Translation { get; private set; }

            public void Dispose()
            {
                Dispose(true);
                GC.SuppressFinalize(this);
            }

            /// <summary>
            /// If you override this method, always call: base.Dispose(disposing);
            /// </summary>
            /// <param name="disposing">true if managed resources should be disposed; otherwise, false.</param>
            protected virtual void Dispose(bool disposing)
            {
                if (disposing)
                    Translation.LanguageChanged -= OnLanguageChanged;
            }

            protected override void OnInitialized()
            {
                Translation.LanguageChanged += OnLanguageChanged;
            }

            private void OnLanguageChanged(object sender, EventArgs e)
                => StateHasChanged();
        }
    }
   ```
1. Your pages derive by default from `ComponentBase`. Now if you add `@inherits ComponentWithTranslations` at the top of your page, this component is used as the base class, and you can use translations on a page by inserting something like **`@Translation["Welcome"]`**.

1. If you also want to use translations in a Layout (like MainLayout.razor), create another base class for layouts like:
`public abstract class LayoutWithTranslations : LayoutComponentBase, IDisposable`.

## Create a SelectLanguage Razor component

1. In the Components folder of the BlazorExample.Client project, add a Razor Component **SelectLanguage.razor**:
   ```cs
    @using BlazorExample.Abstractions
    @inherits ComponentWithTranslations
    @inject ITranslationService TranslationService

    @if (TranslationService.AvailableLanguages == null)
    {
        <div><p><em>Loading...</em></p></div>
    }
    else
    {
        <div title="@Translation["Select Language"] (@TranslationService.SelectedLanguage.NativeName)">
            <select @bind-value="LanguageCultureName" @bind-value:event="onchange">
                @foreach (var cultureInfo in TranslationService.AvailableLanguages)
                {
                    @if (cultureInfo.Name == TranslationService.SelectedLanguage.Name)
                    {
                        <option selected="selected" value="@cultureInfo.Name">@cultureInfo.NativeName</option>
                    }
                    else
                    {
                        <option value="@cultureInfo.Name">@cultureInfo.NativeName</option>
                    }
                }
            </select>
        </div>
    }

    @code
    {
        private string _languageCultureName;

        public string LanguageCultureName
        {
            get => _languageCultureName;
            set
            {
                _languageCultureName = value;
                TranslationService.ChangeLanguage(value);
            }
        }
    }
   ```

## Select and use the languages in the BlazorExample.WebApp

1. Edit **Program.cs**:
   ```cs
    using BlazorExample.Abstractions;
    using BlazorExample.Client.Localization;
    ...
    public static async Task Main(string[] args)
    {
        var builder = WebAssemblyHostBuilder.CreateDefault(args);

        // Localization:
        builder.Services.AddSingleton<ILanguageLoader, LanguageLoader>();
        builder.Services.AddSingleton<TranslationService>();
        builder.Services.AddSingleton<ITranslationProvider>(sp => sp.GetRequiredService<TranslationService>());
        builder.Services.AddSingleton<ITranslationService>(sp => sp.GetRequiredService<TranslationService>());

        builder.RootComponents.Add<App>("app");
        ...
   ```
1. In **_Imports.razor** add:
   ```cs
   @using BlazorExample.Client.Components
   ```
1. In **Shared/MainLayout.razor** replace
   ```html
    <div class="top-row px-4">
        <a href="http://blazor.net" target="_blank" class="ml-md-auto">About</a>
    </div>
   ```
   with:
   ```html
    <div class="top-row px-4">
        <SelectLanguage />
    </div>
   ```
1. Replace the content of **Index.razor** with:
   ```html
    @inherits ComponentWithTranslations
    @page "/"

    <h1>@Translation["Welcome"]</h1>

    @Translation["Welcome to your new app"].

    <SurveyPrompt Title="@Translation["How is Blazor working for you?"]" />
   ```
1. Add the translations to **SharedResource.nl.resx**:<br/>
   Welcome - Welkom<br/>
   Welcome to your new app - Welkom in uw nieuwe app<br/>
   How is Blazor working for you? - Hoe bevalt Blazor?<br/>
   Select Language - Selecteer taal

1. Set **BlazorExample.Api** as startup project and run. When you select **Nederlands** the app will show:
   ![Welkom.png](/assets/images/blazor-localization-welkom.png)
   Change to **English** and the app will show:
   ![Welcome.png](/assets/images/blazor-localization-welcome.png)

## References

See: [Blazor localization](https://github.com/aspnet/AspNetCore.Docs/issues/13436){:target="_blank"}

See: [https://gametorrahod.com/workaround-for-client-side-blazor-localization-with-resx/](https://gametorrahod.com/workaround-for-client-side-blazor-localization-with-resx/){:target="_blank"}

See: [https://dev.to/j_sakamoto/how-to-localize-texts-in-your-blazor-app-phn](https://dev.to/j_sakamoto/how-to-localize-texts-in-your-blazor-app-phn){:target="_blank"}

See: [https://remibou.github.io/I18n-with-Blazor-and-ASPNET-Core/](https://remibou.github.io/I18n-with-Blazor-and-ASPNET-Core/){:target="_blank"}

