---
description: >-
  This how to uses  DNTCaptcha.Core Package from NuGet to implement Captcha in
  your application.
---

# Adding Captcha to Asp.Net Core MVC Application

The first thing we need to do is install the DNTCaptcha.Core into our site project using the Nuget Package manager. Open the Nuget package manager and browse for the DNTCaptcha.Core package and install it.&#x20;

Next, navigate to the views directory, edit the **\_ViewImports.cshtml** file, and add the following line:

```csharp
@addTagHelper *, DNTCaptcha.Core
```

If you are using Asp.Net Core 6.0 or greater, add the following line to your **program.cs** file right below the `builder.Services.AddControllersWithViews().AddRazorRuntimeCompilation(); builder.Services.AddRazorPages();`&#x20;

```csharp
builder.Services.AddDNTCaptcha(options =>
    options.UseCookieStorageProvider()
        .ShowThousandsSeparators(false)
);
```

If you are using an earlier version of asp.net core, edit your startup file and add the same code without the builder.

