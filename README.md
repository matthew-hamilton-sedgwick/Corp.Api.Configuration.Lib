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
  - [Get Repository Connection String Name](#get-repository-connection-string-name)
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

This library is designed for **web applications that cannot access the database directly** and must communicate with the Configuration API via HTTP. It uses [Refit](https://github.com/reactiveui/refit) for type-safe HTTP client generation and includes built-in distributed caching for improved performance.

> **Note:** If your application has direct database access (e.g., another API in the same network), use `Corp.Api.Configuration.Lib.DbDirect` instead for better performance.

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
  - `Corp.Lib.Refit` (v10.0.2)
  - `Microsoft.Extensions.Caching.Hybrid` (v9.4.0)

## Configuration

### appsettings.json Setup

Add the following configuration to your `appsettings.json`:

```json
{
  "TargetedVoyagerInstance": "MyInstance",
  "TargetedVoyagerEnvironment": "Development",
  
  "MyInstance.Development.ConfigurationApiUrl": "https://config-api.example.com",
  "MyInstance.Development.ConfigurationApiCertificatePath": "C:\\Certificates\\client.pfx",
  "MyInstance.Development.ConfigurationApiPassword": "AES-encrypted-password-here"
}
```

### Configuration Keys Explained

| Key | Description | Required |
|-----|-------------|----------|
| `TargetedVoyagerInstance` | The instance identifier (e.g., client name, deployment name) | Yes |
| `TargetedVoyagerEnvironment` | The environment (Development, Staging, Production) | Yes |
| `{Instance}.{Environment}.ConfigurationApiUrl` | The base URL of the Configuration API | Yes |
| `{Instance}.{Environment}.ConfigurationApiCertificatePath` | Full path to the client certificate (.pfx file) | Yes |
| `{Instance}.{Environment}.ConfigurationApiPassword` | AES-encrypted password for the certificate | Yes |

### Encrypting the Certificate Password

Use `Corp.Lib.Cryptography.Aes` to encrypt your certificate password:

```csharp
using Corp.Lib.Cryptography;

// Encrypt the password before storing in configuration
string encryptedPassword = Aes.Encrypt("your-certificate-password");
Console.WriteLine($"Encrypted password: {encryptedPassword}");
```

### Environment Variables (Alternative)

You can also use environment variables instead of appsettings.json. Note that the `TargetedVoyagerInstance` and `TargetedVoyagerEnvironment` values must still be in appsettings.json:

```bash
# Windows
set MyInstance.Development.ConfigurationApiUrl=https://config-api.example.com
set MyInstance.Development.ConfigurationApiCertificatePath=C:\Certificates\client.pfx
set MyInstance.Development.ConfigurationApiPassword=AES-encrypted-password

# Linux/macOS (use double underscores)
export MyInstance__Development__ConfigurationApiUrl=https://config-api.example.com
export MyInstance__Development__ConfigurationApiCertificatePath=/certificates/client.pfx
export MyInstance__Development__ConfigurationApiPassword=AES-encrypted-password
```

## Service Registration

Register the services in your `Program.cs`:

```csharp
using Corp.Api.Configuration.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Add Configuration API services
builder.AddConfigurationApiAndLoadConfigurations("MyApplicationName");

var app = builder.Build();
```

The `AddConfigurationApiAndLoadConfigurations()` extension method:
- Reads configuration from appsettings.json
- Registers Refit HTTP clients with certificate-based authentication
- Configures `HybridCache` with an in-memory distributed cache (L1 + L2 caching)
- Loads configuration key-value pairs from the API into the application's `IConfiguration`

## Available Services

The library provides three main services:

| Service | Description |
|---------|-------------|
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

Retrieves all configuration settings for a specific application. Results are **cached for 90 days** using `HybridCache` (L1 in-memory + L2 distributed).

> **Note:** This method returns `List<Configuration>?` directly (not wrapped in `ApiResponse`) because the cached content is returned, not the HTTP response metadata.

```csharp
// First call: fetches from API and caches the result
List<Configuration>? configs = await _configurationService.GetByApplicationNameAsync("MyApp");

// Subsequent calls within 90 days: returns cached data (no API call)
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

### Get Repository Connection String Name

Returns the connection string name used by the API's repository layer. Useful for diagnostics.

```csharp
string connectionStringName = await _heartbeatService.GetMyRepositoryConnectionStringNameAsync();
Console.WriteLine($"API is using connection string: {connectionStringName}");
```

**API Endpoint:** `GET /Heartbeat/GetMyRepositoryConnectionStringName`

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
| `GetByApplicationNameAsync(string)` | **Yes** | 90 days | `List<Configuration>?` |
| `ResetCachedConfigurationAsync(string)` | Invalidates & reloads | 90 days | `List<Configuration>?` |
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
builder.AddConfigurationApiAndLoadConfigurations("MyApplicationName");

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
builder.AddConfigurationApiAndLoadConfigurations("MyApplicationName");

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
using Refit;

var builder = WebApplication.CreateBuilder(args);

// Optional: Configure Redis for load-balanced environments
// builder.Services.AddStackExchangeRedisCache(options =>
// {
//     options.Configuration = "localhost:6379";
//     options.InstanceName = "ConfigApi_";
// });

// Register Configuration API services and load configurations
builder.AddConfigurationApiAndLoadConfigurations("MyApplicationName");

var app = builder.Build();

// Example endpoint using the Configuration API
app.MapGet("/config/{appName}", async (
    string appName,
    IConfigurationService configService,
    IApplicationService appService) =>
{
    // Verify application exists
    var appResponse = await appService.GetByNameAsync(appName);
    
    if (!appResponse.IsSuccessStatusCode || appResponse.Content == null)
    {
        return Results.NotFound($"Application '{appName}' not found");
    }

    // Get cached configurations
    var configs = await configService.GetByApplicationNameAsync(appName);
    
    return Results.Ok(new
    {
        Application = appResponse.Content,
        Configurations = configs,
        Count = configs?.Count ?? 0
    });
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

**Error:** `ConfigurationErrorsException: {Instance}.{Environment}.ConfigurationApiUrl environment variable does not exist`

**Solution:** Ensure the composite configuration key exists:

```json
{
  "TargetedVoyagerInstance": "MyInstance",
  "TargetedVoyagerEnvironment": "Development",
  "MyInstance.Development.ConfigurationApiUrl": "https://config-api.example.com"
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
| GET | `/Heartbeat/GetMyRepositoryConnectionStringName` | Get connection string name |

---

## Version History

| Version | Changes |
|---------|---------|
| 10.0.3 | Updated all service methods to return `ApiResponse<T>` for access to HTTP response metadata. Migrated from `IDistributedCache` to `HybridCache` for improved caching with stampede protection. Cache duration increased to 90 days. |
| 10.0.0-beta-2 | Initial version targeting .NET 10 |

---

## License

Copyright © Sedgwick Consumer Claims. All rights reserved.

## See Also

- [Corp.Api.Configuration.Lib.DbDirect](../Corp.Api.Configuration.Lib.DbDirect/README.md) - For direct database access
- [Corp.Api.Configuration.Obj](../Corp.Api.Configuration.Obj/README.md) - Entity models and DTOs
