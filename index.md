# Forestbrook.net Knowledge Base


# Create the GetLanguage and SetLanguage Api actions

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

