---
title: Lazy load assemblies in ASP.NET Core Blazor WebAssembly
author: guardrex
description: Discover how to lazy load assemblies in Blazor WebAssembly apps.
monikerRange: '>= aspnetcore-5.0'
ms.author: riande
ms.custom: mvc
ms.date: 11/12/2024
uid: blazor/webassembly-lazy-load-assemblies
---
# Lazy load assemblies in ASP.NET Core Blazor WebAssembly

[!INCLUDE[](~/includes/not-latest-version.md)]

Blazor WebAssembly app startup performance can be improved by waiting to load developer-created app assemblies until the assemblies are required, which is called *lazy loading*.

This article's initial sections cover the app configuration. For a working demonstration, see the [Complete example](#complete-example) section at the end of this article.

*This article only applies to Blazor WebAssembly apps.* Assembly lazy loading doesn't benefit server-side apps because server-rendered apps don't download assemblies to the client.

Lazy loading shouldn't be used for core runtime assemblies, which might be trimmed on publish and unavailable on the client when the app loads.

## File extension placeholder (`{FILE EXTENSION}`) for assembly files

:::moniker range=">= aspnetcore-8.0"

Assembly files use the [Webcil packaging format for .NET assemblies](xref:blazor/host-and-deploy/webassembly/index#webcil-packaging-format-for-net-assemblies) with a `.wasm` file extension.

Throughout the article, the `{FILE EXTENSION}` placeholder represents "`wasm`".

:::moniker-end

:::moniker range="< aspnetcore-8.0"

Assembly files are based on Dynamic-Link Libraries (DLLs) with a `.dll` file extension.

Throughout the article, the `{FILE EXTENSION}` placeholder represents "`dll`".

:::moniker-end

## Project file configuration

Mark assemblies for lazy loading in the app's project file (`.csproj`) using the `BlazorWebAssemblyLazyLoad` item. Use the assembly name with file extension. The Blazor framework prevents the assembly from loading at app launch.

```xml
<ItemGroup>
  <BlazorWebAssemblyLazyLoad Include="{ASSEMBLY NAME}.{FILE EXTENSION}" />
</ItemGroup>
```

The `{ASSEMBLY NAME}` placeholder is the name of the assembly, and the `{FILE EXTENSION}` placeholder is the file extension. The file extension is required.

Include one `BlazorWebAssemblyLazyLoad` item for each assembly. If an assembly has dependencies, include a `BlazorWebAssemblyLazyLoad` entry for each dependency.

## `Router` component configuration

The Blazor framework automatically registers a singleton service for lazy loading assemblies in client-side Blazor WebAssembly apps, <xref:Microsoft.AspNetCore.Components.WebAssembly.Services.LazyAssemblyLoader>. The <xref:Microsoft.AspNetCore.Components.WebAssembly.Services.LazyAssemblyLoader.LoadAssembliesAsync%2A?displayProperty=nameWithType> method:

* Uses [JS interop](xref:blazor/js-interop/call-dotnet-from-javascript) to fetch assemblies via a network call.
* Loads assemblies into the runtime executing on WebAssembly in the browser.

:::moniker range="< aspnetcore-8.0"

> [!NOTE]
> Guidance for *hosted* Blazor WebAssembly [solutions](xref:blazor/tooling#visual-studio-solution-file-sln) is covered in the [Lazy load assemblies in a hosted Blazor WebAssembly solution](#lazy-load-assemblies-in-a-hosted-blazor-webassembly-solution) section.

:::moniker-end

Blazor's <xref:Microsoft.AspNetCore.Components.Routing.Router> component designates the assemblies that Blazor searches for routable components and is also responsible for rendering the component for the route where the user navigates. The <xref:Microsoft.AspNetCore.Components.Routing.Router> component's [`OnNavigateAsync` method](xref:blazor/fundamentals/routing#handle-asynchronous-navigation-events-with-onnavigateasync) is used in conjunction with lazy loading to load the correct assemblies for endpoints that a user requests.

Logic is implemented inside <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> to determine the assemblies to load with <xref:Microsoft.AspNetCore.Components.WebAssembly.Services.LazyAssemblyLoader>. Options for how to structure the logic include:

* Conditional checks inside the <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> method.
* A lookup table that maps routes to assembly names, either injected into the component or implemented within the component's code.

In the following example:

* The namespace for <xref:Microsoft.AspNetCore.Components.WebAssembly.Services?displayProperty=fullName> is specified.
* The <xref:Microsoft.AspNetCore.Components.WebAssembly.Services.LazyAssemblyLoader> service is injected (`AssemblyLoader`).
* The `{PATH}` placeholder is the path where the list of assemblies should load. The example uses a conditional check for a single path that loads a single set of assemblies.
* The `{LIST OF ASSEMBLIES}` placeholder is the comma-separated list of assembly file name strings, including their file extensions (for example, `"Assembly1.{FILE EXTENSION}", "Assembly2.{FILE EXTENSION}"`).

`App.razor`:

:::moniker range=">= aspnetcore-8.0"

```razor
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject LazyAssemblyLoader AssemblyLoader
@inject ILogger<App> Logger

<Router AppAssembly="typeof(App).Assembly" 
    OnNavigateAsync="OnNavigateAsync">
    ...
</Router>

@code {
    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
           {
               if (args.Path == "{PATH}")
               {
                   var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                       [ {LIST OF ASSEMBLIES} ]);
               }
           }
           catch (Exception ex)
           {
               Logger.LogError("Error: {Message}", ex.Message);
           }
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-8.0"

```razor
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject LazyAssemblyLoader AssemblyLoader
@inject ILogger<App> Logger

<Router AppAssembly="typeof(App).Assembly" 
    OnNavigateAsync="OnNavigateAsync">
    ...
</Router>

@code {
    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
           {
               if (args.Path == "{PATH}")
               {
                   var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                       new[] { {LIST OF ASSEMBLIES} });
               }
           }
           catch (Exception ex)
           {
               Logger.LogError("Error: {Message}", ex.Message);
           }
    }
}
```

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```razor
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject LazyAssemblyLoader AssemblyLoader
@inject ILogger<App> Logger

<Router AppAssembly="typeof(Program).Assembly" 
    OnNavigateAsync="OnNavigateAsync">
    ...
</Router>

@code {
    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
           {
               if (args.Path == "{PATH}")
               {
                   var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                       new[] { {LIST OF ASSEMBLIES} });
               }
           }
           catch (Exception ex)
           {
               Logger.LogError("Error: {Message}", ex.Message);
           }
    }
}
```

:::moniker-end

> [!NOTE]
> The preceding example doesn't show the contents of the <xref:Microsoft.AspNetCore.Components.Routing.Router> component's Razor markup (`...`). For a demonstration with complete code, see the [Complete example](#complete-example) section of this article.

:::moniker range="= aspnetcore-5.0"

[!INCLUDE[](~/blazor/includes/prefer-exact-matches.md)]

:::moniker-end

## Assemblies that include routable components

When the list of assemblies includes routable components, the assembly list for a given path is passed to the <xref:Microsoft.AspNetCore.Components.Routing.Router> component's <xref:Microsoft.AspNetCore.Components.Routing.Router.AdditionalAssemblies> collection.

In the following example:

* The [List](xref:System.Collections.Generic.List%601)\<<xref:System.Reflection.Assembly>> in `lazyLoadedAssemblies` passes the assembly list to <xref:Microsoft.AspNetCore.Components.Routing.Router.AdditionalAssemblies>. The framework searches the assemblies for routes and updates the route collection if new routes are found. To access the <xref:System.Reflection.Assembly> type, the namespace for <xref:System.Reflection?displayProperty=fullName> is included at the top of the `App.razor` file.
* The `{PATH}` placeholder is the path where the list of assemblies should load. The example uses a conditional check for a single path that loads a single set of assemblies.
* The `{LIST OF ASSEMBLIES}` placeholder is the comma-separated list of assembly file name strings, including their file extensions (for example, `"Assembly1.{FILE EXTENSION}", "Assembly2.{FILE EXTENSION}"`).

`App.razor`:

:::moniker range=">= aspnetcore-8.0"

```razor
@using System.Reflection
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject ILogger<App> Logger
@inject LazyAssemblyLoader AssemblyLoader

<Router AppAssembly="typeof(App).Assembly" 
    AdditionalAssemblies="lazyLoadedAssemblies" 
    OnNavigateAsync="OnNavigateAsync">
    ...
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = [];

    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
        {
            if (args.Path == "{PATH}")
            {
                var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                    [ {LIST OF ASSEMBLIES} ]);
                lazyLoadedAssemblies.AddRange(assemblies);
            }
        }
        catch (Exception ex)
        {
            Logger.LogError("Error: {Message}", ex.Message);
        }
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-8.0"

```razor
@using System.Reflection
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject ILogger<App> Logger
@inject LazyAssemblyLoader AssemblyLoader

<Router AppAssembly="typeof(App).Assembly" 
    AdditionalAssemblies="lazyLoadedAssemblies" 
    OnNavigateAsync="OnNavigateAsync">
    ...
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = new();

    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
        {
            if (args.Path == "{PATH}")
            {
                var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                    new[] { {LIST OF ASSEMBLIES} });
                lazyLoadedAssemblies.AddRange(assemblies);
            }
        }
        catch (Exception ex)
        {
            Logger.LogError("Error: {Message}", ex.Message);
        }
    }
}
```

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```razor
@using System.Reflection
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject ILogger<App> Logger
@inject LazyAssemblyLoader AssemblyLoader

<Router AppAssembly="typeof(Program).Assembly" 
    AdditionalAssemblies="lazyLoadedAssemblies" 
    OnNavigateAsync="OnNavigateAsync">
    ...
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = new List<Assembly>();

    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
           {
               if (args.Path == "{PATH}")
               {
                   var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                       new[] { {LIST OF ASSEMBLIES} });
                   lazyLoadedAssemblies.AddRange(assemblies);
               }
           }
           catch (Exception ex)
           {
               Logger.LogError("Error: {Message}", ex.Message);
           }
    }
}
```

:::moniker-end

> [!NOTE]
> The preceding example doesn't show the contents of the <xref:Microsoft.AspNetCore.Components.Routing.Router> component's Razor markup (`...`). For a demonstration with complete code, see the [Complete example](#complete-example) section of this article.

:::moniker range="= aspnetcore-5.0"

[!INCLUDE[](~/blazor/includes/prefer-exact-matches.md)]

:::moniker-end

For more information, see <xref:blazor/fundamentals/routing#route-to-components-from-multiple-assemblies>.

## User interaction with `<Navigating>` content

While loading assemblies, which can take several seconds, the <xref:Microsoft.AspNetCore.Components.Routing.Router> component can indicate to the user that a page transition is occurring with the router's <xref:Microsoft.AspNetCore.Components.Routing.Router.Navigating> property.

For more information, see <xref:blazor/fundamentals/routing#user-interaction-with-navigating-content>.

## Handle cancellations in `OnNavigateAsync`

The <xref:Microsoft.AspNetCore.Components.Routing.NavigationContext> object passed to the <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> callback contains a <xref:Microsoft.AspNetCore.Components.Routing.NavigationContext.CancellationToken> that's set when a new navigation event occurs. The <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> callback must throw when the cancellation token is set to avoid continuing to run the <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> callback on an outdated navigation.

For more information, see <xref:blazor/fundamentals/routing#handle-cancellations-in-onnavigateasync>.

:::moniker range=">= aspnetcore-5.0 < aspnetcore-8.0"

## `OnNavigateAsync` events and renamed assembly files

The resource loader relies on the assembly names that are defined in the boot manifest file. If [assemblies are renamed](xref:blazor/host-and-deploy/webassembly/index#change-the-file-name-extension-of-dll-files), the assembly names used in an <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> callback and the assembly names in the boot manifest file are out of sync.

To rectify this:

* Check to see if the app is running in the `Production` environment when determining which assembly names to use.
* Store the renamed assembly names in a separate file and read from that file to determine what assembly name to use with the <xref:Microsoft.AspNetCore.Components.WebAssembly.Services.LazyAssemblyLoader> service and <xref:Microsoft.AspNetCore.Components.Routing.Router.OnNavigateAsync> callback.

:::moniker-end

:::moniker range="< aspnetcore-8.0"

## Lazy load assemblies in a hosted Blazor WebAssembly solution

The framework's lazy loading implementation supports lazy loading with prerendering in a hosted Blazor WebAssembly [solution](xref:blazor/tooling#visual-studio-solution-file-sln). During prerendering, all assemblies, including those marked for lazy loading, are assumed to be loaded. Manually register the <xref:Microsoft.AspNetCore.Components.WebAssembly.Services.LazyAssemblyLoader> service in the **:::no-loc text="Server":::** project.

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-8.0"

At the top of the `Program.cs` file of the **:::no-loc text="Server":::** project, add the namespace for <xref:Microsoft.AspNetCore.Components.WebAssembly.Services?displayProperty=fullName>:

```csharp
using Microsoft.AspNetCore.Components.WebAssembly.Services;
```

In `Program.cs` of the **:::no-loc text="Server":::** project, register the service:

```csharp
builder.Services.AddScoped<LazyAssemblyLoader>();
```

:::moniker-end

:::moniker range="< aspnetcore-6.0"

At the top of the `Startup.cs` file of the **:::no-loc text="Server":::** project, add the namespace for <xref:Microsoft.AspNetCore.Components.WebAssembly.Services?displayProperty=fullName>:

```csharp
using Microsoft.AspNetCore.Components.WebAssembly.Services;
```

In `Startup.ConfigureServices` (`Startup.cs`) of the **:::no-loc text="Server":::** project, register the service:

```csharp
services.AddScoped<LazyAssemblyLoader>();
```

:::moniker-end

## Complete example

The demonstration in this section:

* Creates a robot controls assembly (`GrantImaharaRobotControls.{FILE EXTENSION}`) as a [Razor class library (RCL)](xref:blazor/components/class-libraries) that includes a `Robot` component (`Robot.razor` with a route template of `/robot`).
* Lazily loads the RCL's assembly to render its `Robot` component when the `/robot` URL is requested by the user.

Create a standalone Blazor WebAssembly app to demonstrate lazy loading of a Razor class library's assembly. Name the project `LazyLoadTest`.

Add an ASP.NET Core class library project to the solution: 

* Visual Studio: Right-click the solution file in **Solution Explorer** and select **Add** > **New project**. From the dialog of new project types, select **Razor Class Library**. Name the project `GrantImaharaRobotControls`. Do **not** select the **Support pages and views** checkbox.
* Visual Studio Code/.NET CLI: Execute `dotnet new razorclasslib -o GrantImaharaRobotControls` from a command prompt. The `-o|--output` option creates a folder and names the project `GrantImaharaRobotControls`.

Create a `HandGesture` class in the RCL with a `ThumbUp` method that hypothetically makes a robot perform a thumbs-up gesture. The method accepts an argument for the axis, `Left` or `Right`, as an [`enum`](/dotnet/csharp/language-reference/builtin-types/enum). The method returns `true` on success.

`HandGesture.cs`:

:::moniker range=">= aspnetcore-6.0"

```csharp
using Microsoft.Extensions.Logging;

namespace GrantImaharaRobotControls;

public static class HandGesture
{
    public static bool ThumbUp(Axis axis, ILogger logger)
    {
        logger.LogInformation("Thumb up gesture. Axis: {Axis}", axis);

        // Code to make robot perform gesture

        return true;
    }
}

public enum Axis { Left, Right }
```

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```csharp
using Microsoft.Extensions.Logging;

namespace GrantImaharaRobotControls
{
    public static class HandGesture
    {
        public static bool ThumbUp(Axis axis, ILogger logger)
        {
            logger.LogInformation("Thumb up gesture. Axis: {Axis}", axis);

            // Code to make robot perform gesture

            return true;
        }
    }

    public enum Axis { Left, Right }
}
```

:::moniker-end

Add the following component to the root of the RCL project. The component permits the user to submit a left or right hand thumb-up gesture request.

`Robot.razor`:

:::moniker range=">= aspnetcore-8.0"

```razor
@page "/robot"
@using Microsoft.AspNetCore.Components.Forms
@using Microsoft.Extensions.Logging
@inject ILogger<Robot> Logger

<h1>Robot</h1>

<EditForm FormName="RobotForm" Model="robotModel" OnValidSubmit="HandleValidSubmit">
    <InputRadioGroup @bind-Value="robotModel.AxisSelection">
        @foreach (var entry in Enum.GetValues<Axis>())
        {
            <InputRadio Value="entry" />
            <text>&nbsp;</text>@entry<br>
        }
    </InputRadioGroup>

    <button type="submit">Submit</button>
</EditForm>

<p>
    @message
</p>

@code {
    private RobotModel robotModel = new() { AxisSelection = Axis.Left };
    private string? message;

    private void HandleValidSubmit()
    {
        Logger.LogInformation("HandleValidSubmit called");

        var result = HandGesture.ThumbUp(robotModel.AxisSelection, Logger);

        message = $"ThumbUp returned {result} at {DateTime.Now}.";
    }

    public class RobotModel
    {
        public Axis AxisSelection { get; set; }
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-8.0"

```razor
@page "/robot"
@using Microsoft.AspNetCore.Components.Forms
@using Microsoft.Extensions.Logging
@inject ILogger<Robot> Logger

<h1>Robot</h1>

<EditForm Model="robotModel" OnValidSubmit="HandleValidSubmit">
    <InputRadioGroup @bind-Value="robotModel.AxisSelection">
        @foreach (var entry in Enum.GetValues<Axis>())
        {
            <InputRadio Value="entry" />
            <text>&nbsp;</text>@entry<br>
        }
    </InputRadioGroup>

    <button type="submit">Submit</button>
</EditForm>

<p>
    @message
</p>

@code {
    private RobotModel robotModel = new() { AxisSelection = Axis.Left };
    private string? message;

    private void HandleValidSubmit()
    {
        Logger.LogInformation("HandleValidSubmit called");

        var result = HandGesture.ThumbUp(robotModel.AxisSelection, Logger);

        message = $"ThumbUp returned {result} at {DateTime.Now}.";
    }

    public class RobotModel
    {
        public Axis AxisSelection { get; set; }
    }
}
```

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```razor
@page "/robot"
@using Microsoft.AspNetCore.Components.Forms
@using Microsoft.Extensions.Logging
@inject ILogger<Robot> Logger

<h1>Robot</h1>

<EditForm Model="robotModel" OnValidSubmit="HandleValidSubmit">
    <InputRadioGroup @bind-Value="robotModel.AxisSelection">
        @foreach (var entry in (Axis[])Enum
            .GetValues(typeof(Axis)))
        {
            <InputRadio Value="entry" />
            <text>&nbsp;</text>@entry<br>
        }
    </InputRadioGroup>

    <button type="submit">Submit</button>
</EditForm>

<p>
    @message
</p>

@code {
    private RobotModel robotModel = new RobotModel() { AxisSelection = Axis.Left };
    private string message;

    private void HandleValidSubmit()
    {
        Logger.LogInformation("HandleValidSubmit called");

        var result = HandGesture.ThumbUp(robotModel.AxisSelection, Logger);

        message = $"ThumbUp returned {result} at {DateTime.Now}.";
    }

    public class RobotModel
    {
        public Axis AxisSelection { get; set; }
    }
}
```

:::moniker-end

In the `LazyLoadTest` project, create a project reference for the `GrantImaharaRobotControls` RCL:

* Visual Studio: Right-click the `LazyLoadTest` project and select **Add** > **Project Reference** to add a project reference for the `GrantImaharaRobotControls` RCL.
* Visual Studio Code/.NET CLI: Execute `dotnet add reference {PATH}` in a command shell from the project's folder. The `{PATH}` placeholder is the path to the RCL project.

Specify the RCL's assembly for lazy loading in the `LazyLoadTest` app's project file (`.csproj`):

```xml
<ItemGroup>
    <BlazorWebAssemblyLazyLoad Include="GrantImaharaRobotControls.{FILE EXTENSION}" />
</ItemGroup>
```

The following <xref:Microsoft.AspNetCore.Components.Routing.Router> component demonstrates loading the `GrantImaharaRobotControls.{FILE EXTENSION}` assembly when the user navigates to `/robot`. Replace the app's default `App` component with the following `App` component.

During page transitions, a styled message is displayed to the user with the `<Navigating>` element. For more information, see the [User interaction with `<Navigating>` content](#user-interaction-with-navigating-content) section.

The assembly is assigned to <xref:Microsoft.AspNetCore.Components.Routing.Router.AdditionalAssemblies>, which results in the router searching the assembly for routable components, where it finds the `Robot` component. The `Robot` component's route is added to the app's route collection. For more information, see the <xref:blazor/fundamentals/routing#route-to-components-from-multiple-assemblies> article and the [Assemblies that include routable components](#assemblies-that-include-routable-components) section of this article.

`App.razor`:

:::moniker range=">= aspnetcore-8.0"

```razor
@using System.Reflection
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject ILogger<App> Logger
@inject LazyAssemblyLoader AssemblyLoader

<Router AppAssembly="typeof(App).Assembly"
        AdditionalAssemblies="lazyLoadedAssemblies" 
        OnNavigateAsync="OnNavigateAsync">
    <Navigating>
        <div style="padding:20px;background-color:blue;color:white">
            <p>Loading the requested page&hellip;</p>
        </div>
    </Navigating>
    <Found Context="routeData">
        <RouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)" />
    </Found>
    <NotFound>
        <LayoutView Layout="typeof(MainLayout)">
            <p>Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = [];
    private bool grantImaharaRobotControlsAssemblyLoaded;

    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
        {
            if ((args.Path == "robot") && !grantImaharaRobotControlsAssemblyLoaded)
            {
                var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                    [ "GrantImaharaRobotControls.{FILE EXTENSION}" ]);
                lazyLoadedAssemblies.AddRange(assemblies);
                grantImaharaRobotControlsAssemblyLoaded = true;
            }
        }
        catch (Exception ex)
        {
            Logger.LogError("Error: {Message}", ex.Message);
        }
    }
}
```

:::moniker-end

:::moniker range=">= aspnetcore-6.0 < aspnetcore-8.0"

```razor
@using System.Reflection
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject ILogger<App> Logger
@inject LazyAssemblyLoader AssemblyLoader

<Router AppAssembly="typeof(App).Assembly"
        AdditionalAssemblies="lazyLoadedAssemblies" 
        OnNavigateAsync="OnNavigateAsync">
    <Navigating>
        <div style="padding:20px;background-color:blue;color:white">
            <p>Loading the requested page&hellip;</p>
        </div>
    </Navigating>
    <Found Context="routeData">
        <RouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)" />
    </Found>
    <NotFound>
        <LayoutView Layout="typeof(MainLayout)">
            <p>Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = new();
    private bool grantImaharaRobotControlsAssemblyLoaded;

    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
        {
            if ((args.Path == "robot") && !grantImaharaRobotControlsAssemblyLoaded)
            {
                var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                    new[] { "GrantImaharaRobotControls.{FILE EXTENSION}" });
                lazyLoadedAssemblies.AddRange(assemblies);
                grantImaharaRobotControlsAssemblyLoaded = true;
            }
        }
        catch (Exception ex)
        {
            Logger.LogError("Error: {Message}", ex.Message);
        }
    }
}
```

:::moniker-end

:::moniker range="< aspnetcore-6.0"

```razor
@using System.Reflection
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@using Microsoft.Extensions.Logging
@inject ILogger<App> Logger
@inject LazyAssemblyLoader AssemblyLoader

<Router AppAssembly="typeof(Program).Assembly"
        AdditionalAssemblies="lazyLoadedAssemblies" 
        OnNavigateAsync="OnNavigateAsync">
    <Navigating>
        <div style="padding:20px;background-color:blue;color:white">
            <p>Loading the requested page&hellip;</p>
        </div>
    </Navigating>
    <Found Context="routeData">
        <RouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)" />
    </Found>
    <NotFound>
        <LayoutView Layout="typeof(MainLayout)">
            <p>Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = new List<Assembly>();
    private bool grantImaharaRobotControlsAssemblyLoaded;

    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
        {
            if ((args.Path == "robot") && !grantImaharaRobotControlsAssemblyLoaded)
            {
                var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                    new[] { "GrantImaharaRobotControls.{FILE EXTENSION}" });
                lazyLoadedAssemblies.AddRange(assemblies);
                grantImaharaRobotControlsAssemblyLoaded = true;
            }
        }
        catch (Exception ex)
        {
            Logger.LogError("Error: {Message}", ex.Message);
        }
    }
}
```

:::moniker-end

Build and run the app.

When the `Robot` component from the RCL is requested at `/robot`, the `GrantImaharaRobotControls.{FILE EXTENSION}` assembly is loaded and the `Robot` component is rendered. You can inspect the assembly loading in the **Network** tab of the browser's developer tools.

## Troubleshoot

* If unexpected rendering occurs, such as rendering a component from a previous navigation, confirm that the code throws if the cancellation token is set.
* If assemblies configured for lazy loading unexpectedly load at app start, check that the assembly is marked for lazy loading in the project file.

:::moniker range="< aspnetcore-6.0"

> [!NOTE]
> A known issue exists for loading types from a lazily-loaded assembly. For more information, see [Blazor WebAssembly lazy loading assemblies not working when using @ref attribute in the component (dotnet/aspnetcore #29342)](https://github.com/dotnet/aspnetcore/issues/29342).

:::moniker-end

## Additional resources

* [Handle asynchronous navigation events with `OnNavigateAsync`](xref:blazor/fundamentals/routing#handle-asynchronous-navigation-events-with-onnavigateasync)
* <xref:blazor/performance/index>
