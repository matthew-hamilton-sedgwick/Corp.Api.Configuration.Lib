# Corp.Api.Configuration.Lib

HTTP/Refit client library for accessing the Configuration API. This NuGet package provides a convenient way for web applications to interact with the Configuration API over HTTP using certificate-based authentication.

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
  - [appsettings.json Setup](#appsettingsjson-setup)
  - [Configuration Keys Explained](#configuration-keys-explained)
  - [Encrypting the Certificate Password](#encrypting-the-certificate-password)
  - [Environment Variables (Alternative)](#environment-variables-alternative)
- [Service Registration](#service-registration)
- [Available Services](#available-services)
- [Return Types](#return-types)
  - [ApiResponse\<T\>](#apiresponset)
  - [Handling API Responses](#handling-api-responses)
- [Usage](#usage)
  - [Dependency Injection](#dependency-injection)
- [IConfigurationService](#iconfigurationservice)
  - [Get All Configurations](#get-all-configurations)
  - [Get Configuration by ID](#get-configuration-by-id)
  - [Get Configurations by Application Name (Cached)](#get-configurations-by-application-name-cached)
  - [Reset Cached Configuration](#reset-cached-configuration)
  - [Insert Configuration](#insert-configuration)
  - [Update Configuration](#update-configuration)
  - [Delete Configuration](#delete-configuration)
- [IApplicationService](#iapplicationservice)
  - [Get All Applications](#get-all-applications)
  - [Get Application by ID](#get-application-by-id)
  - [Get Application by Name](#get-application-by-name)
  - [Insert Application](#insert-application)
  - [Update Application](#update-application)
  - [Delete Application](#delete-application)
- [IHeartbeatService](#iheartbeatservice)
  - [Get Heartbeat](#get-heartbeat)
- [IConfigurationAccessor (Recommended)](#iconfigurationaccessor-recommended)
  - [Why Use IConfigurationAccessor](#why-use-iconfigurationaccessor)
  - [Get a String Value](#get-a-string-value)
  - [Get a String Value with Default](#get-a-string-value-with-default)
  - [Get a Typed Value](#get-a-typed-value)
  - [Get a Typed Value with Default](#get-a-typed-value-with-default)
  - [Get All Configurations](#get-all-configurations-1)
  - [Refresh Cache](#refresh-cache)
  - [Check if Key Exists](#check-if-key-exists)
- [Entity Models](#entity-models)
  - [Application](#application)
  - [Configuration](#configuration-1)
- [Caching](#caching)
- [HybridCache](#hybridcache)
- [Cache Behavior Summary](#cache-behavior-summary)
- [Load-Balanced Environments](#load-balanced-environments)
- [Logging](#logging)
- [Error Handling](#error-handling)
  - [Refit API Exceptions](#refit-api-exceptions)
  - [Common Exceptions](#common-exceptions)
- [Complete Example](#complete-example)
- [Troubleshooting](#troubleshooting)
  - [Configuration Errors](#configuration-errors)
  - [Certificate Errors](#certificate-errors)
  - [Cache Issues](#cache-issues)
- [API Reference Quick Links](#api-reference-quick-links)
  - [Configuration Endpoints](#configuration-endpoints)
  - [Application Endpoints](#application-endpoints)
  - [Heartbeat Endpoints](#heartbeat-endpoints)
- [Version History](#version-history)
- [License](#license)
- [See Also](#see-also)

---

## Overview

This library is designed for **web applications that need to access centralized configuration settings** via HTTP. It uses [Refit](https://github.com/reactiveui/refit) for type-safe HTTP client generation and includes built-in caching using `HybridCache` for optimal performance with stampede protection.

> **Architecture Note:** In our infrastructure, web applications do not have direct database access. All configuration data must be accessed through the Configuration API using this library.

## Installation

```bash
dotnet add package Corp.Api.Configuration.Lib
```

Or via Package Manager:

```powershell
Install-Package Corp.Api.Configuration.Lib
```

## Prerequisites

- .NET 10.0 or later
- Access to the Configuration API endpoint
- Valid client certificate (.pfx) for API authentication
- AES-encrypted certificate password using `Corp.Lib.Cryptography.Aes`
- Required NuGet packages (installed automatically):
  - `Corp.Lib.Cryptography` (v10.0.1)
  - `Corp.Lib.DistributedCache` (v10.0.11)
  - `Corp.Lib.Refit` (v10.0.8)
  - `Microsoft.Extensions.Caching.Hybrid` (v10.2.0)

## Configuration

### appsettings.json Setup

Add the following configuration to your `appsettings.json`:

```json
{
  "TargetedVoyagerInstance": "MyInstance",
  "TargetedVoyagerEnvironment": "Development",
  "ApplicationName": "MyApplicationName",
  "CacheExpirationInDays": 1,
  
  "MyInstance.Development.Corp.Api.Configuration.Url": "https://config-api.example.com",
  "MyInstance.Development.Corp.Api.Configuration.CertificatePath": "C:\\Certificates\\client.pfx",
  "MyInstance.Development.Corp.Api.Configuration.Password": "AES-encrypted-password-here"
}
```

### Configuration Keys Explained

| Key | Description | Required |
|-----|-------------|----------|
| `TargetedVoyagerInstance` | The instance identifier (e.g., client name, deployment name) | Yes |
| `TargetedVoyagerEnvironment` | The environment (Development, Staging, Production) | Yes |
| `ApplicationName` | The name of the application (used to fetch configurations from the API) | Yes |
| `{Instance}.{Environment}.Corp.Api.Configuration.Url` | The base URL of the Configuration API | Yes |
| `{Instance}.{Environment}.Corp.Api.Configuration.CertificatePath` | Full path to the client certificate (.pfx file) | Yes |
| `{Instance}.{Environment}.Corp.Api.Configuration.Password` | AES-encrypted password for the certificate | Yes |
| `CacheExpirationInDays` | Number of days to cache configuration data (default: 1) | No |

### Encrypting the Certificate Password

Use `Corp.Lib.Cryptography.Aes` to encrypt your certificate password:

```csharp
using Corp.Lib.Cryptography;

// Encrypt the password before storing in configuration
string encryptedPassword = Aes.Encrypt("your-certificate-password");
Console.WriteLine($"Encrypted password: {encryptedPassword}");
```

### Environment Variables (Alternative)

You can also use environment variables instead of appsettings.json. Note that the `TargetedVoyagerInstance`, `TargetedVoyagerEnvironment`, and `ApplicationName` values must still be in appsettings.json:

```cmd
set MyInstance.Development.Corp.Api.Configuration.Url=https://config-api.example.com
set MyInstance.Development.Corp.Api.Configuration.CertificatePath=C:\Certificates\client.pfx
set MyInstance.Development.Corp.Api.Configuration.Password=AES-encrypted-password
```

## Service Registration

Register the services in your `Program.cs`:

```csharp
using Corp.Api.Configuration.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Add Configuration API services
builder.AddConfigurationApiAndLoadConfigurations();

var app = builder.Build();
```

The `AddConfigurationApiAndLoadConfigurations()` extension method:
- Reads `TargetedVoyagerInstance`, `TargetedVoyagerEnvironment`, and `ApplicationName` from appsettings.json
- Constructs configuration keys using the pattern `{Instance}.{Environment}.{ApplicationName}.*`
- Registers Refit HTTP clients with certificate-based authentication
- Configures `HybridCache` with an in-memory distributed cache (L1 + L2 caching)
- Loads configuration key-value pairs from the API into the application's `IConfiguration`

## Available Services

The library provides four main services:

| Service | Description |
|---------|-------------|
| `IConfigurationAccessor` | **Recommended** - Cached access to configuration values throughout the application lifecycle |
| `IConfigurationService` | CRUD operations for configuration key-value pairs with caching support |
| `IApplicationService` | CRUD operations for application management |
| `IHeartbeatService` | Health check and diagnostic operations |

## Return Types

All service methods return `Refit.ApiResponse<T>`, which provides access to both the response content and HTTP response metadata.

### ApiResponse\<T\>

The `ApiResponse<T>` type wraps the API response and provides:

| Property | Type | Description |
|----------|------|-------------|
| `Content` | `T` | The deserialized response body |
| `StatusCode` | `HttpStatusCode` | The HTTP status code (e.g., 200, 404, 500) |
| `IsSuccessStatusCode` | `bool` | `true` if status code is 2xx |
| `Headers` | `HttpResponseHeaders` | Response headers |
| `Error` | `ApiException?` | Exception details if the request failed |
| `ReasonPhrase` | `string?` | HTTP reason phrase |

### Handling API Responses

```csharp
using Refit;

// Get all applications with full response details
ApiResponse<List<Application>?> response = await _applicationService.GetAllAsync();

if (response.IsSuccessStatusCode)
{
    List<Application>? applications = response.Content;
    Console.WriteLine($"Retrieved {applications?.Count ?? 0} applications");
}
else
{
    Console.WriteLine($"API returned {response.StatusCode}: {response.ReasonPhrase}");
    
    if (response.Error != null)
    {
        Console.WriteLine($"Error details: {response.Error.Content}");
    }
}
```

**Accessing just the content:**

```csharp
// If you only need the content and want to handle errors via exceptions
var response = await _applicationService.GetByIdAsync(1);
Application? app = response.Content;
```

**Checking specific status codes:**

```csharp
var response = await _applicationService.GetByIdAsync(id);

switch (response.StatusCode)
{
    case HttpStatusCode.OK:
        Console.WriteLine($"Found: {response.Content?.Name}");
        break;
    case HttpStatusCode.NotFound:
        Console.WriteLine("Application not found");
        break;
    case HttpStatusCode.Unauthorized:
        Console.WriteLine("Authentication failed - check certificate");
        break;
    default:
        Console.WriteLine($"Unexpected status: {response.StatusCode}");
        break;
}
```

## Usage

### Dependency Injection

Inject the services into your classes:

```csharp
using Corp.Api.Configuration.Lib.Services.Interfaces;
using Corp.Api.Configuration.Obj.Entities;

public class MyService
{
    private readonly IConfigurationService _configurationService;
    private readonly IApplicationService _applicationService;
    private readonly IHeartbeatService _heartbeatService;

    public MyService(
        IConfigurationService configurationService,
        IApplicationService applicationService,
        IHeartbeatService heartbeatService)
    {
        _configurationService = configurationService;
        _applicationService = applicationService;
        _heartbeatService = heartbeatService;
    }
}
```

---

## IConfigurationService

Provides CRUD operations for managing configuration settings. Configuration settings are key-value pairs associated with specific applications.

### Get All Configurations

Retrieves all configuration settings from the API.

```csharp
var response = await _configurationService.GetAllAsync();

if (response.IsSuccessStatusCode && response.Content != null)
{
    foreach (var config in response.Content)
    {
        Console.WriteLine($"[{config.ApplicationId}] {config.Key}: {config.Value}");
    }
}
else
{
    Console.WriteLine($"Failed to retrieve configurations: {response.StatusCode}");
}
```

**API Endpoint:** `GET /Configuration/GetAll`

**Returns:** `ApiResponse<List<Configuration>?>`

### Get Configuration by ID

Retrieves a specific configuration setting by its unique identifier.

```csharp
var response = await _configurationService.GetByIdAsync(1);

if (response.IsSuccessStatusCode && response.Content != null)
{
    Console.WriteLine($"Found: {response.Content.Key} = {response.Content.Value}");
}
else
{
    Console.WriteLine($"Configuration not found (Status: {response.StatusCode})");
}
```

**API Endpoint:** `GET /Configuration/GetById/{id}`

**Returns:** `ApiResponse<Configuration?>`

### Get Configurations by Application Name (Cached)

Retrieves all configuration settings for a specific application. Results are **cached using `HybridCache`** (L1 in-memory + L2 distributed). The cache duration is configurable via the `CacheExpirationInDays` setting (default: 14 days).

> **Note:** This method returns `List<Configuration>?` directly (not wrapped in `ApiResponse`) because the cached content is returned, not the HTTP response metadata.

```csharp
// First call: fetches from API and caches the result
List<Configuration>? configs = await _configurationService.GetByApplicationNameAsync("MyApp");

// Subsequent calls within the cache duration: returns cached data (no API call)
List<Configuration>? cachedConfigs = await _configurationService.GetByApplicationNameAsync("MyApp");

if (configs != null)
{
    foreach (var config in configs)
    {
        Console.WriteLine($"{config.Key}: {config.Value}");
    }
}
```

**API Endpoint:** `GET /Configuration/GetByApplicationName/{applicationName}`

**Returns:** `List<Configuration>?` (cached content)

**Cache Key Format:** `Corp.Api.Configuration.Lib.Configurations.{applicationName}`

### Reset Cached Configuration

Invalidates the cache for the specified application and reloads fresh data from the API. Use this method after updating configurations to ensure consumers get the latest values.

> **Note:** This method returns `List<Configuration>?` directly (not wrapped in `ApiResponse`) because the cached content is returned.

```csharp
// After updating configurations, reset the cache to reload fresh data
List<Configuration>? freshConfigs = await _configurationService.ResetCachedConfigurationAsync("MyApp");

Console.WriteLine($"Reloaded {freshConfigs?.Count ?? 0} configurations");
```

**Returns:** `List<Configuration>?` (fresh cached content)

**Cache Duration:** Configurable via `CacheExpirationInDays` in app configuration (default: 1 day)
```

### Insert Configuration

Creates a new configuration setting. Returns the ID of the newly created record.

```csharp
var newConfig = new Configuration
{
    ApplicationId = 1,
    Key = "FeatureFlag.EnableNewUI",
    Value = "true",
    CreatedBy = "admin@example.com"
};

var response = await _configurationService.InsertAsync(newConfig);

if (response.IsSuccessStatusCode)
{
    Console.WriteLine($"Created configuration with ID: {response.Content}");
}
else
{
    Console.WriteLine($"Failed to create: {response.StatusCode}");
}
```

**API Endpoint:** `POST /Configuration/Insert`

**Returns:** `ApiResponse<int>` (the new configuration ID)

**Required Fields:**
- `ApplicationId` - Foreign key to the Application table
- `Key` - Configuration key (max 100 characters)
- `Value` - Configuration value
- `CreatedBy` - Username or identifier of the creator

### Update Configuration

Updates an existing configuration setting. Returns the number of rows affected.

```csharp
var getResponse = await _configurationService.GetByIdAsync(1);

if (getResponse.IsSuccessStatusCode && getResponse.Content != null)
{
    var config = getResponse.Content;
    config.Value = "updated-value";
    config.ModifiedBy = "admin@example.com";
    
    var updateResponse = await _configurationService.UpdateAsync(config);
    
    if (updateResponse.IsSuccessStatusCode && updateResponse.Content > 0)
    {
        Console.WriteLine("Configuration updated successfully");
        
        // Don't forget to reset the cache if needed
        await _configurationService.ResetCachedConfigurationAsync("MyApp");
    }
}
```

**API Endpoint:** `PUT /Configuration/Update`

**Returns:** `ApiResponse<int>` (rows affected)

**Required Fields for Update:**
- `Id` - The configuration ID
- `ApplicationId` - Foreign key to the Application table
- `Key` - Configuration key
- `Value` - Configuration value
- `ModifiedBy` - Username or identifier of the modifier

### Delete Configuration

Deletes a configuration setting by its ID. Returns the number of rows affected.

```csharp
var response = await _configurationService.DeleteAsync(configId);

if (response.IsSuccessStatusCode && response.Content > 0)
{
    Console.WriteLine("Configuration deleted successfully");
}
else
{
    Console.WriteLine($"Configuration not found or already deleted (Status: {response.StatusCode})");
}
```

**API Endpoint:** `DELETE /Configuration/Delete/{id}`

**Returns:** `ApiResponse<int>` (rows affected)

---

## IApplicationService

Provides CRUD operations for managing applications. Applications serve as containers for grouping related configuration settings.

### Get All Applications

Retrieves all applications from the API.

```csharp
var response = await _applicationService.GetAllAsync();

if (response.IsSuccessStatusCode && response.Content != null)
{
    foreach (var app in response.Content)
    {
        Console.WriteLine($"[{app.Id}] {app.Name}: {app.Description}");
    }
}
```

**API Endpoint:** `GET /Application/GetAll`

**Returns:** `ApiResponse<List<Application>?>`

### Get Application by ID

Retrieves a specific application by its unique identifier.

```csharp
var response = await _applicationService.GetByIdAsync(1);

if (response.IsSuccessStatusCode && response.Content != null)
{
    Console.WriteLine($"Found: {response.Content.Name} - {response.Content.Description}");
}
```

**API Endpoint:** `GET /Application/GetById/{id}`

**Returns:** `ApiResponse<Application?>`

### Get Application by Name

Retrieves a specific application by its name.

```csharp
var response = await _applicationService.GetByNameAsync("MyApp");

if (response.IsSuccessStatusCode && response.Content != null)
{
    var app = response.Content;
    Console.WriteLine($"Application ID: {app.Id}");
    Console.WriteLine($"Description: {app.Description}");
    Console.WriteLine($"Created: {app.CreatedOn} by {app.CreatedBy}");
}
```

**API Endpoint:** `GET /Application/GetByName/{name}`

**Returns:** `ApiResponse<Application?>`

### Insert Application

Creates a new application. Returns the ID of the newly created record.

```csharp
var newApp = new Application
{
    Name = "MyNewApplication",
    Description = "A new application for testing",
    CreatedBy = "admin@example.com"
};

var response = await _applicationService.InsertAsync(newApp);

if (response.IsSuccessStatusCode)
{
    Console.WriteLine($"Created application with ID: {response.Content}");
}
```

**API Endpoint:** `POST /Application/Insert`

**Returns:** `ApiResponse<int>` (the new application ID)

**Required Fields:**
- `Name` - Application name (max 100 characters, unique)
- `CreatedBy` - Username or identifier of the creator

**Optional Fields:**
- `Description` - Application description (max 1000 characters)

### Update Application

Updates an existing application. Returns the number of rows affected.

```csharp
var getResponse = await _applicationService.GetByIdAsync(1);

if (getResponse.IsSuccessStatusCode && getResponse.Content != null)
{
    var app = getResponse.Content;
    app.Description = "Updated description for the application";
    app.ModifiedBy = "admin@example.com";
    
    var updateResponse = await _applicationService.UpdateAsync(app);
    
    if (updateResponse.IsSuccessStatusCode && updateResponse.Content > 0)
    {
        Console.WriteLine("Application updated successfully");
    }
}
```

**API Endpoint:** `PUT /Application/Update`

**Returns:** `ApiResponse<int>` (rows affected)

**Required Fields for Update:**
- `Id` - The application ID
- `ModifiedBy` - Username or identifier of the modifier

### Delete Application

Deletes an application by its ID. Returns the number of rows affected.

> **Warning:** Deleting an application may affect associated configuration settings. Ensure all related configurations are handled before deletion.

```csharp
var response = await _applicationService.DeleteAsync(appId);

if (response.IsSuccessStatusCode && response.Content > 0)
{
    Console.WriteLine("Application deleted successfully");
}
```

**API Endpoint:** `DELETE /Application/Delete/{id}`

**Returns:** `ApiResponse<int>` (rows affected)

---

## IHeartbeatService

Provides health check and diagnostic operations for the Configuration API.

### Get Heartbeat

Returns the current server timestamp from the Configuration API. Use this to verify API connectivity.

```csharp
var response = await _heartbeatService.GetHeartbeatAsync();

if (response.IsSuccessStatusCode)
{
    DateTime serverTime = response.Content;
    Console.WriteLine($"API Server Time: {serverTime}");
    Console.WriteLine($"Local Time: {DateTime.Now}");
    Console.WriteLine($"Time Difference: {DateTime.Now - serverTime}");
}
else
{
    Console.WriteLine($"Heartbeat failed: {response.StatusCode}");
}
```

**API Endpoint:** `GET /Heartbeat/Get`

**Returns:** `ApiResponse<DateTime>`

---

## IConfigurationAccessor (Recommended)

`IConfigurationAccessor` provides **cached access to configuration values throughout the application lifecycle**. This is the **recommended way** to access configuration values as it leverages `HybridCache` for optimal performance with stampede protection.

### Why Use IConfigurationAccessor

When you call `AddConfigurationApiAndLoadConfigurations()`, configurations are loaded into `IConfiguration` at startup. However, this creates a **static copy** that:
- Never updates until the application restarts
- Doesn't benefit from HybridCache after the initial load
- Cannot reflect configuration changes made during runtime

`IConfigurationAccessor` solves these problems:

| Feature | `IConfiguration` (Startup Load) | `IConfigurationAccessor` |
|---------|--------------------------------|--------------------------|
| Cache used | Only at startup | Every access |
| Runtime updates | ❌ Requires restart | ✅ Call `RefreshAsync()` |
| Stampede protection | ❌ Only startup | ✅ Throughout lifecycle |
| Type conversion | Manual parsing | Built-in with `IParsable<T>` |

### Dependency Injection

Inject `IConfigurationAccessor` into your classes:

```csharp
using Corp.Api.Configuration.Lib.Services.Interfaces;

public class MyService
{
    private readonly IConfigurationAccessor _config;

    public MyService(IConfigurationAccessor config)
    {
        _config = config;
    }
}
```

### Get a String Value

Retrieves a configuration value by key. Returns `null` if the key doesn't exist.

```csharp
string? apiUrl = await _config.GetValueAsync("MyApp", "ApiUrl");

if (apiUrl != null)
{
    Console.WriteLine($"API URL: {apiUrl}");
}
else
{
    Console.WriteLine("ApiUrl configuration not found");
}
```

**Method Signature:**
```csharp
Task<string?> GetValueAsync(string applicationName, string key);
```

### Get a String Value with Default

Retrieves a configuration value by key, returning a default value if not found.

```csharp
// Returns "30" if "Timeout" key doesn't exist
string timeout = await _config.GetValueAsync("MyApp", "Timeout", "30");
Console.WriteLine($"Timeout: {timeout} seconds");

// Useful for feature flags with safe defaults
string featureEnabled = await _config.GetValueAsync("MyApp", "EnableNewUI", "false");
```

**Method Signature:**
```csharp
Task<string> GetValueAsync(string applicationName, string key, string defaultValue);
```

### Get a Typed Value

Retrieves a configuration value and converts it to the specified type using `IParsable<T>`. Returns `default(T)` if the key doesn't exist or conversion fails.

Supported types include: `int`, `long`, `double`, `decimal`, `bool`, `DateTime`, `DateTimeOffset`, `TimeSpan`, `Guid`, and any type implementing `IParsable<T>`.

```csharp
// Get an integer value
int? maxRetries = await _config.GetValueAsync<int>("MyApp", "MaxRetries");
if (maxRetries.HasValue)
{
    Console.WriteLine($"Max retries: {maxRetries.Value}");
}

// Get a boolean value
bool? enableLogging = await _config.GetValueAsync<bool>("MyApp", "EnableDetailedLogging");
if (enableLogging == true)
{
    Console.WriteLine("Detailed logging is enabled");
}

// Get a decimal value
decimal? taxRate = await _config.GetValueAsync<decimal>("MyApp", "TaxRate");

// Get a TimeSpan value (format: "hh:mm:ss" or "d.hh:mm:ss")
TimeSpan? sessionTimeout = await _config.GetValueAsync<TimeSpan>("MyApp", "SessionTimeout");

// Get a DateTime value
DateTime? maintenanceWindow = await _config.GetValueAsync<DateTime>("MyApp", "MaintenanceStartTime");

// Get a Guid value
Guid? tenantId = await _config.GetValueAsync<Guid>("MyApp", "DefaultTenantId");
```

**Method Signature:**
```csharp
Task<T?> GetValueAsync<T>(string applicationName, string key) where T : IParsable<T>;
```

> **Note:** If the value cannot be parsed to the specified type, a warning is logged and `default(T)` is returned.

### Get a Typed Value with Default

Retrieves a configuration value and converts it to the specified type, returning a default value if not found or conversion fails.

```csharp
// Get an integer with default
int maxConnections = await _config.GetValueAsync("MyApp", "MaxConnections", 100);
Console.WriteLine($"Max connections: {maxConnections}");

// Get a boolean with default (great for feature flags)
bool enableCache = await _config.GetValueAsync("MyApp", "EnableCache", true);
bool useDarkMode = await _config.GetValueAsync("MyApp", "UseDarkMode", false);

// Get a decimal with default
decimal discountRate = await _config.GetValueAsync("MyApp", "DiscountRate", 0.10m);

// Get a TimeSpan with default
TimeSpan cacheExpiration = await _config.GetValueAsync("MyApp", "CacheExpiration", TimeSpan.FromMinutes(30));

// Complex example: Building a connection string
string host = await _config.GetValueAsync("MyApp", "DbHost", "localhost");
int port = await _config.GetValueAsync("MyApp", "DbPort", 5432);
string database = await _config.GetValueAsync("MyApp", "DbName", "mydb");
```

**Method Signature:**
```csharp
Task<T> GetValueAsync<T>(string applicationName, string key, T defaultValue) where T : IParsable<T>;
```

### Get All Configurations

Retrieves all configuration key-value pairs for an application as a read-only dictionary. Useful when you need to access multiple values or iterate over all configurations.

```csharp
IReadOnlyDictionary<string, string> allConfigs = await _config.GetAllAsync("MyApp");

Console.WriteLine($"Found {allConfigs.Count} configurations:");

foreach (var (key, value) in allConfigs)
{
    Console.WriteLine($"  {key}: {value}");
}

// Check if a specific key exists and get its value
if (allConfigs.TryGetValue("ApiUrl", out var apiUrl))
{
    Console.WriteLine($"API URL: {apiUrl}");
}

// Filter configurations by prefix
var featureFlags = allConfigs
    .Where(kv => kv.Key.StartsWith("FeatureFlag."))
    .ToDictionary(kv => kv.Key, kv => kv.Value);

Console.WriteLine($"Found {featureFlags.Count} feature flags");
```

**Method Signature:**
```csharp
Task<IReadOnlyDictionary<string, string>> GetAllAsync(string applicationName);
```

**Returns:** A dictionary of all configuration key-value pairs, or an empty dictionary if none found.

### Refresh Cache

Invalidates the cache for the specified application and reloads fresh data from the API. Use this method after configuration changes to ensure all consumers get updated values.

```csharp
// After updating configurations via IConfigurationService
var updateResponse = await _configurationService.UpdateAsync(config);

if (updateResponse.IsSuccessStatusCode)
{
    // Refresh the cache so IConfigurationAccessor returns updated values
    await _config.RefreshAsync("MyApp");
    
    Console.WriteLine("Configuration updated and cache refreshed");
}

// You can also refresh on a schedule or trigger
public class ConfigurationRefreshBackgroundService : BackgroundService
{
    private readonly IConfigurationAccessor _config;
    private readonly TimeSpan _refreshInterval = TimeSpan.FromHours(1);

    public ConfigurationRefreshBackgroundService(IConfigurationAccessor config)
    {
        _config = config;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await Task.Delay(_refreshInterval, stoppingToken);
            await _config.RefreshAsync("MyApp");
        }
    }
}
```

**Method Signature:**
```csharp
Task RefreshAsync(string applicationName);
```

### Check if Key Exists

Checks whether a configuration key exists for the specified application without retrieving its value.

```csharp
bool hasApiKey = await _config.ContainsKeyAsync("MyApp", "ExternalApiKey");

if (hasApiKey)
{
    string? apiKey = await _config.GetValueAsync("MyApp", "ExternalApiKey");
    // Use the API key...
}
else
{
    Console.WriteLine("Warning: ExternalApiKey is not configured");
}

// Useful for conditional feature initialization
if (await _config.ContainsKeyAsync("MyApp", "RedisConnectionString"))
{
    // Initialize Redis connection
}
else
{
    // Fall back to in-memory cache
}
```

**Method Signature:**
```csharp
Task<bool> ContainsKeyAsync(string applicationName, string key);
```

### Complete IConfigurationAccessor Example

Here's a comprehensive example showing `IConfigurationAccessor` in a real-world service:

```csharp
using Corp.Api.Configuration.Lib.Services.Interfaces;

public class EmailService
{
    private readonly IConfigurationAccessor _config;
    private readonly ILogger<EmailService> _logger;
    private const string AppName = "EmailService";

    public EmailService(IConfigurationAccessor config, ILogger<EmailService> logger)
    {
        _config = config;
        _logger = logger;
    }

    public async Task<bool> SendEmailAsync(string to, string subject, string body)
    {
        // Get SMTP configuration with defaults
        string smtpHost = await _config.GetValueAsync(AppName, "SmtpHost", "localhost");
        int smtpPort = await _config.GetValueAsync(AppName, "SmtpPort", 587);
        bool useSsl = await _config.GetValueAsync(AppName, "SmtpUseSsl", true);
        
        // Get optional credentials
        string? username = await _config.GetValueAsync(AppName, "SmtpUsername");
        string? password = await _config.GetValueAsync(AppName, "SmtpPassword");
        
        // Get sender information
        string fromAddress = await _config.GetValueAsync(AppName, "FromAddress", "noreply@example.com");
        string fromName = await _config.GetValueAsync(AppName, "FromName", "System");
        
        // Check for feature flags
        bool enableEmailLogging = await _config.GetValueAsync(AppName, "EnableEmailLogging", false);
        
        // Get rate limiting settings
        int maxEmailsPerMinute = await _config.GetValueAsync(AppName, "MaxEmailsPerMinute", 60);
        
        if (enableEmailLogging)
        {
            _logger.LogInformation("Sending email to {To} via {Host}:{Port}", to, smtpHost, smtpPort);
        }

        // ... send email logic ...

        return true;
    }

    public async Task ReloadConfigurationAsync()
    {
        await _config.RefreshAsync(AppName);
        _logger.LogInformation("Email service configuration reloaded");
    }
}
```

### When to Use Each Service

| Use Case | Recommended Service |
|----------|-------------------|
| **Reading configuration values at runtime** | `IConfigurationAccessor` ✅ |
| **Feature flags and settings** | `IConfigurationAccessor` ✅ |
| **Creating/updating/deleting configurations** | `IConfigurationService` |
| **Admin UI for configuration management** | `IConfigurationService` |
| **Reading configuration at startup only** | `IConfiguration` (via startup load) |
| **Application CRUD operations** | `IApplicationService` |
| **Health checks** | `IHeartbeatService` |

---

## Entity Models

### Application

```csharp
public class Application
{
    public int Id { get; set; }              // Auto-generated primary key
    public string Name { get; set; }          // Required, max 100 chars, unique
    public string? Description { get; set; }  // Optional, max 1000 chars
    public DateTime CreatedOn { get; set; }   // Auto-set by database
    public string CreatedBy { get; set; }     // Required on insert
    public DateTime? ModifiedOn { get; set; } // Auto-set by database on update
    public string? ModifiedBy { get; set; }   // Required on update
}
```

### Configuration

```csharp
public class Configuration
{
    public int Id { get; set; }              // Auto-generated primary key
    public int ApplicationId { get; set; }    // Required, FK to Application
    public string Key { get; set; }           // Required, max 100 chars
    public string Value { get; set; }         // Required
    public DateTime CreatedOn { get; set; }   // Auto-set by database
    public string CreatedBy { get; set; }     // Required on insert
    public DateTime? ModifiedOn { get; set; } // Auto-set by database on update
    public string? ModifiedBy { get; set; }   // Required on update
}
```

---

## Caching

### HybridCache

This library uses `Microsoft.Extensions.Caching.Hybrid.HybridCache` for caching, which provides:

- **Two-tier caching (L1 + L2):** Fast in-memory cache (L1) backed by a distributed cache (L2)
- **Stampede protection:** Prevents multiple concurrent requests from all hitting the API when the cache expires
- **Automatic serialization:** No manual JSON serialization/deserialization required
- **Simplified API:** Uses `GetOrCreateAsync` pattern for cleaner code

By default, the library configures an in-memory distributed cache as L2, which is suitable for single-server deployments.

### Cache Behavior Summary

| Method | Cached | Cache Duration | Returns |
|--------|--------|----------------|--------|
| `GetAllAsync()` | No | - | `ApiResponse<List<Configuration>?>` |
| `GetByIdAsync(int)` | No | - | `ApiResponse<Configuration?>` |
| `GetByApplicationNameAsync(string)` | **Yes** | Configurable (default: 1 day) | `List<Configuration>?` |
| `ResetCachedConfigurationAsync(string)` | Invalidates & reloads | Configurable (default: 1 day) | `List<Configuration>?` |
| `InsertAsync()` | No | - | `ApiResponse<int>` |
| `UpdateAsync()` | No | - | `ApiResponse<int>` |
| `DeleteAsync(int)` | No | - | `ApiResponse<int>` |

> **Note:** Cached methods return the content directly (`List<Configuration>?`) rather than `ApiResponse<T>` because the cache stores the deserialized content, not the HTTP response metadata.

> **Important:** After calling `InsertAsync()`, `UpdateAsync()`, or `DeleteAsync()`, you should call `ResetCachedConfigurationAsync()` to ensure cached data is refreshed.

### Load-Balanced Environments

For load-balanced or multi-server environments, configure a distributed cache (L2) **before** calling `AddConfigurationApiAndLoadConfigurations()` to ensure cache consistency across all servers. `HybridCache` will automatically use it as the secondary cache.

#### Redis (Recommended)

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
using Corp.Api.Configuration.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Configure Redis BEFORE AddConfigurationApiAndLoadConfigurations()
// HybridCache will use this as L2 cache
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "ConfigApi_";
});

// HybridCache will use Redis as L2, with in-memory as L1
builder.AddConfigurationApiAndLoadConfigurations();

var app = builder.Build();
```

#### SQL Server

```bash
dotnet add package Microsoft.Extensions.Caching.SqlServer
```

```csharp
using Corp.Api.Configuration.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Configure SQL Server cache BEFORE AddConfigurationApiAndLoadConfigurations()
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("CacheDb");
    options.SchemaName = "dbo";
    options.TableName = "ConfigurationCache";
});

// HybridCache will use SQL Server as L2, with in-memory as L1
builder.AddConfigurationApiAndLoadConfigurations();

var app = builder.Build();
```

Create the cache table using:

```sql
CREATE TABLE [dbo].[ConfigurationCache] (
    [Id] NVARCHAR(449) NOT NULL PRIMARY KEY,
    [Value] VARBINARY(MAX) NOT NULL,
    [ExpiresAtTime] DATETIMEOFFSET NOT NULL,
    [SlidingExpirationInSeconds] BIGINT NULL,
    [AbsoluteExpiration] DATETIMEOFFSET NULL
);
CREATE NONCLUSTERED INDEX [Index_ExpiresAtTime] ON [dbo].[ConfigurationCache] ([ExpiresAtTime]);
```

---

## Logging

The library uses `Corp.Lib.Logging.Logger` for debug and error logging. All service methods log:
- Debug information for API calls
- Error details when exceptions occur

Ensure your application has configured `Corp.Lib.Logging` to capture these logs.

---

## Error Handling

### Refit API Exceptions

```csharp
using Refit;

try
{
    var configs = await _configurationService.GetByApplicationNameAsync("MyApp");
}
catch (ApiException ex)
{
    // HTTP error from API (4xx, 5xx responses)
    Console.WriteLine($"API Error: {ex.StatusCode}");
    Console.WriteLine($"Reason: {ex.ReasonPhrase}");
    Console.WriteLine($"Content: {ex.Content}");
    
    if (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
    {
        Console.WriteLine("Application not found");
    }
    else if (ex.StatusCode == System.Net.HttpStatusCode.Unauthorized)
    {
        Console.WriteLine("Certificate authentication failed");
    }
}
catch (HttpRequestException ex)
{
    // Network/connection error
    Console.WriteLine($"Connection Error: {ex.Message}");
}
```

### Common Exceptions

| Exception | Cause | Solution |
|-----------|-------|----------|
| `ConfigurationErrorsException` | Missing or invalid configuration | Verify appsettings.json has all required keys |
| `ApiException` (401) | Certificate authentication failed | Check certificate path and password |
| `ApiException` (404) | Resource not found | Verify the ID or name exists |
| `HttpRequestException` | Network connectivity issue | Check API URL and network access |

---

## Complete Example

```csharp
using Corp.Api.Configuration.Lib.Extensions;
using Corp.Api.Configuration.Lib.Services.Interfaces;
using Corp.Api.Configuration.Obj.Entities;

var builder = WebApplication.CreateBuilder(args);

// Optional: Configure Redis for load-balanced environments
// builder.Services.AddStackExchangeRedisCache(options =>
// {
//     options.Configuration = "localhost:6379";
//     options.InstanceName = "ConfigApi_";
// });

// Register Configuration API services and load configurations
builder.AddConfigurationApiAndLoadConfigurations();

var app = builder.Build();

// Example: Using IConfigurationAccessor for runtime configuration (RECOMMENDED)
app.MapGet("/settings", async (IConfigurationAccessor config) =>
{
    // Get typed configuration values with defaults
    string apiUrl = await config.GetValueAsync("MyApp", "ExternalApiUrl", "https://api.example.com");
    int timeout = await config.GetValueAsync("MyApp", "TimeoutSeconds", 30);
    bool enableFeature = await config.GetValueAsync("MyApp", "EnableNewFeature", false);
    
    return Results.Ok(new
    {
        ApiUrl = apiUrl,
        Timeout = timeout,
        FeatureEnabled = enableFeature
    });
});

// Example: Get all configurations for an application
app.MapGet("/config/{appName}", async (
    string appName,
    IConfigurationAccessor configAccessor,
    IApplicationService appService) =>
{
    // Verify application exists
    var appResponse = await appService.GetByNameAsync(appName);
    
    if (!appResponse.IsSuccessStatusCode || appResponse.Content == null)
    {
        return Results.NotFound($"Application '{appName}' not found");
    }

    // Get all configurations using IConfigurationAccessor (cached)
    var configs = await configAccessor.GetAllAsync(appName);
    
    return Results.Ok(new
    {
        Application = appResponse.Content,
        Configurations = configs,
        Count = configs.Count
    });
});

// Example: Refresh configuration cache (useful after updates)
app.MapPost("/config/{appName}/refresh", async (
    string appName,
    IConfigurationAccessor configAccessor) =>
{
    await configAccessor.RefreshAsync(appName);
    
    return Results.Ok(new { Message = $"Cache refreshed for {appName}" });
});

// Example: CRUD operations using IConfigurationService
app.MapPost("/config", async (
    Configuration newConfig,
    IConfigurationService configService,
    IConfigurationAccessor configAccessor) =>
{
    var response = await configService.InsertAsync(newConfig);
    
    if (response.IsSuccessStatusCode)
    {
        // Refresh the cache so IConfigurationAccessor sees the new value
        // Note: You'd need to resolve the application name from ApplicationId
        // await configAccessor.RefreshAsync("MyApp");
        
        return Results.Created($"/config/{response.Content}", new { Id = response.Content });
    }
    
    return Results.Problem($"Failed to create configuration: {response.StatusCode}");
});

// Health check endpoint
app.MapGet("/health", async (IHeartbeatService heartbeat) =>
{
    var response = await heartbeat.GetHeartbeatAsync();
    
    if (response.IsSuccessStatusCode)
    {
        return Results.Ok(new { Status = "Healthy", ServerTime = response.Content });
    }
    
    return Results.Problem(
        title: "Unhealthy", 
        detail: $"API returned {response.StatusCode}",
        statusCode: 503);
});

app.Run();
```

---

## Troubleshooting

### Configuration Errors

**Error:** `ConfigurationErrorsException: TargetedVoyagerInstance configuration element does not exist`

**Solution:** Ensure `TargetedVoyagerInstance` is defined in your `appsettings.json`:

```json
{
  "TargetedVoyagerInstance": "MyInstance"
}
```

**Error:** `ConfigurationErrorsException: {Instance}.{Environment}.Corp.Api.Configuration.Url environment variable does not exist`

**Solution:** Ensure all required configuration keys exist in your `appsettings.json`:

```json
{
  "TargetedVoyagerInstance": "MyInstance",
  "TargetedVoyagerEnvironment": "Development",
  "ApplicationName": "MyApplicationName",
  "MyInstance.Development.Corp.Api.Configuration.Url": "https://config-api.example.com"
}
```

### Certificate Errors

**Error:** `The certificate cannot be found` or `The certificate password is invalid`

**Solutions:**
1. Verify the certificate path is correct and the file exists
2. Ensure the certificate password is properly AES-encrypted using `Corp.Lib.Cryptography.Aes.Encrypt()`
3. Check that the application has read permissions on the certificate file
4. Verify the certificate is not expired

### Cache Issues

**Issue:** Configuration changes not reflected across servers

**Solution:** Configure Redis or SQL Server distributed cache for load-balanced environments. The default in-memory cache is not shared across servers.

**Issue:** Stale configuration data

**Solution:** Call `ResetCachedConfigurationAsync()` after making changes:

```csharp
await _configurationService.UpdateAsync(config);
await _configurationService.ResetCachedConfigurationAsync("MyApp");
```

---

## API Reference Quick Links

### Configuration Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/Configuration/GetAll` | Get all configurations |
| GET | `/Configuration/GetById/{id}` | Get configuration by ID |
| GET | `/Configuration/GetByApplicationName/{name}` | Get configurations by app name |
| POST | `/Configuration/Insert` | Create new configuration |
| PUT | `/Configuration/Update` | Update existing configuration |
| DELETE | `/Configuration/Delete/{id}` | Delete configuration |

### Application Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/Application/GetAll` | Get all applications |
| GET | `/Application/GetById/{id}` | Get application by ID |
| GET | `/Application/GetByName/{name}` | Get application by name |
| POST | `/Application/Insert` | Create new application |
| PUT | `/Application/Update` | Update existing application |
| DELETE | `/Application/Delete/{id}` | Delete application |

### Heartbeat Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/Heartbeat/Get` | Get server timestamp |

---

## Version History

| Version | Changes |
|---------|---------|
| 10.0.28 | Code quality improvements: fixed inconsistent `ConfigureAwait` usage, updated cache expiration default to 1 day, `ConfigurationAccessor` now reads `CacheExpirationInDays` from configuration for consistency with `ConfigurationService`. |
| 10.0.5 | Made cache expiration configurable via `CacheExpirationInDays` setting. |
| 10.0.4 | Added `IConfigurationAccessor` for cached runtime configuration access with typed value support. This is now the recommended way to access configuration values throughout the application lifecycle. |
| 10.0.3 | Updated all service methods to return `ApiResponse<T>` for access to HTTP response metadata. Migrated from `IDistributedCache` to `HybridCache` for improved caching with stampede protection. |
| 10.0.0-beta-2 | Initial version targeting .NET 10 |

---

## License

Copyright © Sedgwick Consumer Claims. All rights reserved.

## See Also

- [Corp.Api.Configuration.Obj](../Corp.Api.Configuration.Obj/README.md) - Entity models and DTOs
- [Microsoft HybridCache Documentation](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid) - Learn more about HybridCache
- [Refit Documentation](https://github.com/reactiveui/refit) - Type-safe REST library for .NET
