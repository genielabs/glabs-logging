[![Build status](https://ci.appveyor.com/api/projects/status/6genehqw9wtuuxkl?svg=true)](https://ci.appveyor.com/project/genemars/glabs-logging)
[![NuGet](https://img.shields.io/nuget/v/GLabs.Logging.svg)](https://www.nuget.org/packages/GLabs.Logging/)
![License](https://img.shields.io/github/license/genielabs/glabs-logging.svg)

# GLabs.Logging

GLabs.Logging provides a simple, static logging API, reminiscent of NLog or log4net, built on top of the modern `Microsoft.Extensions.Logging` (MEL) framework.

It is designed to help library authors transition to MEL without introducing breaking changes, and to provide a zero-configuration "it just works" experience for simple applications and tests, while remaining fully configurable for production environments.

## Why GLabs.Logging?

Modern .NET development encourages using Dependency Injection to provide `ILogger<T>` instances. While this is a powerful pattern, it presents challenges:
1.  **Breaking Changes:** Migrating a library that exposes a static logger (`MyLib.Log.Info(...)`) to an instance-based logger requires breaking changes for all consumers of that library.
2.  **Boilerplate for Simple Apps:** Setting up the full MEL host and service provider just to get a simple console logger in a test app or a small utility can be verbose.
3.  **Lack of a Unified Strategy:** When an application uses multiple independent libraries, each might have its own logging strategy, leading to fragmented and inconsistent log outputs.

GLabs.Logging solves these problems by providing a lightweight bridge that unifies logging under a single, simple-to-use static API.

## Features

-   **Static Logger API:** Familiar `LogManager.GetLogger()` and `Log.Info(...)` pattern.
-   **Zero-Configuration Fallback:** Works out-of-the-box for simple console applications and tests by automatically creating a default console logger.
-   **Production-Ready:** Fully integrates with the host application's logging system via a one-line `Initialize()` call.
-   **Built on MEL:** Leverages the power and flexibility of `Microsoft.Extensions.Logging`, allowing consumers to use any MEL-compatible provider (Serilog, NLog, Azure, etc.).
-   **Multi-Targeted:** Fully compatible with .NET Framework, .NET Standard, and modern .NET versions (.NET 6/8/9+).

## Installation

Install the package from NuGet:

```bash
dotnet add package GLabs.Logging```

## How to Use

GLabs.Logging supports two primary modes of operation.

### 1. Basic Usage (Zero-Configuration Fallback)

This is ideal for quick tests, examples, or simple console utilities. You don't need to configure anything.

**Example `Program.cs`:**
```csharp
using GLabs.Logging;

public class Program
{
    // Get a logger for this class. The category will be "Program".
    private static readonly Logger Log = LogManager.GetCurrentClassLogger();

    public static void Main(string[] args)
    {
        Log.Info("Application starting up.");
        Log.Warn("This is a warning message.");
        
        try
        {
            throw new InvalidOperationException("This is a test exception.");
        }
        catch (Exception ex)
        {
            Log.Error(ex);
        }
    }
}```

**Output:**
When you run this code, `LogManager` will detect that it hasn't been initialized and will create a default console logger. It will also print a one-time warning to inform you that it's using a fallback configuration.

```log
warn: GLabs.LogManager[0] LogManager was not initialized. A default console logger will be used...
info: Program[0] Application starting up.
warn: Program[0] This is a warning message.
fail: Program[0] This is a test exception.
      System.InvalidOperationException: This is a test exception.
         at Program.Main...
```

### 2. Advanced Usage (Production & Full Configuration)

This is the recommended approach for any real application (Web API, Worker Service, etc.). The host application sets up its logging as usual and then passes the configured `ILoggerFactory` to GLabs.Logging.

**Step 1: Configure your `appsettings.json`**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "MyWebApp": "Debug",
      "MIG": "Trace"
    },
    "Console": {
      "FormatterName": "simple",
      "FormatterOptions": {
        "SingleLine": true,
        "IncludeScopes": true,
        "TimestampFormat": "yyyy-MM-dd HH:mm:ss.fff "
      }
    }
  }
}```

**Step 2: Initialize `LogManager` at startup**
In your `Program.cs`, build your host and use the configured `ILoggerFactory` to initialize the `LogManager`.

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using GLabs.Logging;

var builder = Host.CreateApplicationBuilder(args);

// MEL is configured automatically from appsettings.json by the host builder

var host = builder.Build();

// --- The crucial step ---
// Get the factory from the host and initialize GLabs.Logging
var loggerFactory = host.Services.GetRequiredService<ILoggerFactory>();
LogManager.Initialize(loggerFactory);

// --- Now, all libraries using GLabs.Logging will use the host's configuration ---
var appLogger = LogManager.GetLogger("MyWebApp");
appLogger.Info("Application started. Logging is fully configured.");

// Any call from another library, like MIG.MigService.Log, will now write
// to the same targets with the rules defined in appsettings.json.

host.Run();
```

With this setup, all logs from your application and any G-Labs libraries will be processed by the same pipeline, respecting the formats and log levels defined in your `appsettings.json`.

## License

This project is licensed under the **Apache 2.0 License**. See the [LICENSE](LICENSE) file for details.
