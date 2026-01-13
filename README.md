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
- [Distributed Caching](#distributed-caching)
  - [Default Behavior](#default-behavior)
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
  - `Corp.Lib.Cryptography` (v10.0.0)
  - `Corp.Lib.Refit` (v10.0.2)

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
builder.AddConfigurationApi();

var app = builder.Build();
```

The `AddConfigurationApi()` extension method:
- Reads configuration from appsettings.json
- Registers Refit HTTP clients with certificate-based authentication
- Configures an in-memory distributed cache (suitable for single-server deployments)
- Registers the `IDistributedCacheAccessor` for internal caching operations

## Available Services

The library provides three main services:

| Service | Description |
|---------|-------------|
| `IConfigurationService` | CRUD operations for configuration key-value pairs with caching support |
| `IApplicationService` | CRUD operations for application management |
| `IHeartbeatService` | Health check and diagnostic operations |

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
List<Configuration>? configurations = await _configurationService.GetAllAsync();

if (configurations != null)
{
    foreach (var config in configurations)
    {
        Console.WriteLine($"[{config.ApplicationId}] {config.Key}: {config.Value}");
    }
}
```

**API Endpoint:** `GET /Configuration/GetAll`

### Get Configuration by ID

Retrieves a specific configuration setting by its unique identifier.

```csharp
Configuration? configuration = await _configurationService.GetByIdAsync(1);

if (configuration != null)
{
    Console.WriteLine($"Found: {configuration.Key} = {configuration.Value}");
}
else
{
    Console.WriteLine("Configuration not found");
}
```

**API Endpoint:** `GET /Configuration/GetById/{id}`

### Get Configurations by Application Name (Cached)

Retrieves all configuration settings for a specific application. Results are **cached for 30 days** in the distributed cache.

```csharp
// First call: fetches from API and caches the result
List<Configuration>? configs = await _configurationService.GetByApplicationNameAsync("MyApp");

// Subsequent calls within 30 days: returns cached data (no API call)
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

**Cache Key Format:** `Corp.Api.Configuration.Lib.Configurations.{applicationName}`

### Reset Cached Configuration

Invalidates the cache for the specified application and reloads fresh data from the API. Use this method after updating configurations to ensure consumers get the latest values.

```csharp
// After updating configurations, reset the cache to reload fresh data
List<Configuration>? freshConfigs = await _configurationService.ResetCachedConfigurationAsync("MyApp");

Console.WriteLine($"Reloaded {freshConfigs?.Count ?? 0} configurations");
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

int newId = await _configurationService.InsertAsync(newConfig);
Console.WriteLine($"Created configuration with ID: {newId}");
```

**API Endpoint:** `POST /Configuration/Insert`

**Required Fields:**
- `ApplicationId` - Foreign key to the Application table
- `Key` - Configuration key (max 100 characters)
- `Value` - Configuration value
- `CreatedBy` - Username or identifier of the creator

### Update Configuration

Updates an existing configuration setting. Returns the number of rows affected.

```csharp
Configuration? config = await _configurationService.GetByIdAsync(1);

if (config != null)
{
    config.Value = "updated-value";
    config.ModifiedBy = "admin@example.com";
    
    int rowsAffected = await _configurationService.UpdateAsync(config);
    
    if (rowsAffected > 0)
    {
        Console.WriteLine("Configuration updated successfully");
        
        // Don't forget to reset the cache if needed
        await _configurationService.ResetCachedConfigurationAsync("MyApp");
    }
}
```

**API Endpoint:** `PUT /Configuration/Update`

**Required Fields for Update:**
- `Id` - The configuration ID
- `ApplicationId` - Foreign key to the Application table
- `Key` - Configuration key
- `Value` - Configuration value
- `ModifiedBy` - Username or identifier of the modifier

### Delete Configuration

Deletes a configuration setting by its ID. Returns the number of rows affected.

```csharp
int rowsAffected = await _configurationService.DeleteAsync(configId);

if (rowsAffected > 0)
{
    Console.WriteLine("Configuration deleted successfully");
}
else
{
    Console.WriteLine("Configuration not found or already deleted");
}
```

**API Endpoint:** `DELETE /Configuration/Delete/{id}`

---

## IApplicationService

Provides CRUD operations for managing applications. Applications serve as containers for grouping related configuration settings.

### Get All Applications

Retrieves all applications from the API.

```csharp
List<Application>? applications = await _applicationService.GetAllAsync();

if (applications != null)
{
    foreach (var app in applications)
    {
        Console.WriteLine($"[{app.Id}] {app.Name}: {app.Description}");
    }
}
```

**API Endpoint:** `GET /Application/GetAll`

### Get Application by ID

Retrieves a specific application by its unique identifier.

```csharp
Application? app = await _applicationService.GetByIdAsync(1);

if (app != null)
{
    Console.WriteLine($"Found: {app.Name} - {app.Description}");
}
```

**API Endpoint:** `GET /Application/GetById/{id}`

### Get Application by Name

Retrieves a specific application by its name.

```csharp
Application? app = await _applicationService.GetByNameAsync("MyApp");

if (app != null)
{
    Console.WriteLine($"Application ID: {app.Id}");
    Console.WriteLine($"Description: {app.Description}");
    Console.WriteLine($"Created: {app.CreatedOn} by {app.CreatedBy}");
}
```

**API Endpoint:** `GET /Application/GetByName/{name}`

### Insert Application

Creates a new application. Returns the ID of the newly created record.

```csharp
var newApp = new Application
{
    Name = "MyNewApplication",
    Description = "A new application for testing",
    CreatedBy = "admin@example.com"
};

int newId = await _applicationService.InsertAsync(newApp);
Console.WriteLine($"Created application with ID: {newId}");
```

**API Endpoint:** `POST /Application/Insert`

**Required Fields:**
- `Name` - Application name (max 100 characters, unique)
- `CreatedBy` - Username or identifier of the creator

**Optional Fields:**
- `Description` - Application description (max 1000 characters)

### Update Application

Updates an existing application. Returns the number of rows affected.

```csharp
Application? app = await _applicationService.GetByIdAsync(1);

if (app != null)
{
    app.Description = "Updated description for the application";
    app.ModifiedBy = "admin@example.com";
    
    int rowsAffected = await _applicationService.UpdateAsync(app);
    
    if (rowsAffected > 0)
    {
        Console.WriteLine("Application updated successfully");
    }
}
```

**API Endpoint:** `PUT /Application/Update`

**Required Fields for Update:**
- `Id` - The application ID
- `ModifiedBy` - Username or identifier of the modifier

### Delete Application

Deletes an application by its ID. Returns the number of rows affected.

> **Warning:** Deleting an application may affect associated configuration settings. Ensure all related configurations are handled before deletion.

```csharp
int rowsAffected = await _applicationService.DeleteAsync(appId);

if (rowsAffected > 0)
{
    Console.WriteLine("Application deleted successfully");
}
```

**API Endpoint:** `DELETE /Application/Delete/{id}`

---

## IHeartbeatService

Provides health check and diagnostic operations for the Configuration API.

### Get Heartbeat

Returns the current server timestamp from the Configuration API. Use this to verify API connectivity.

```csharp
DateTime serverTime = await _heartbeatService.GetHeartbeatAsync();
Console.WriteLine($"API Server Time: {serverTime}");
Console.WriteLine($"Local Time: {DateTime.Now}");
Console.WriteLine($"Time Difference: {DateTime.Now - serverTime}");
```

**API Endpoint:** `GET /Heartbeat/Get`

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

## Distributed Caching

### Default Behavior

By default, this library uses **in-memory distributed cache**, which is suitable for single-server deployments. The cache is configured automatically when calling `AddConfigurationApi()`.

### Cache Behavior Summary

| Method | Cached | Cache Duration | Notes |
|--------|--------|----------------|-------|
| `GetAllAsync()` | No | - | Always fetches fresh data |
| `GetByIdAsync(int)` | No | - | Always fetches fresh data |
| `GetByApplicationNameAsync(string)` | **Yes** | 30 days | Primary caching method |
| `ResetCachedConfigurationAsync(string)` | Invalidates & reloads | 30 days | Use after updates |
| `InsertAsync()` | No | - | Does not invalidate cache |
| `UpdateAsync()` | No | - | Does not invalidate cache |
| `DeleteAsync(int)` | No | - | Does not invalidate cache |

> **Important:** After calling `InsertAsync()`, `UpdateAsync()`, or `DeleteAsync()`, you should call `ResetCachedConfigurationAsync()` to ensure cached data is refreshed.

### Load-Balanced Environments

For load-balanced or multi-server environments, configure a distributed cache **before** calling `AddConfigurationApi()` to ensure cache consistency across all servers.

#### Redis (Recommended)

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
using Corp.Api.Configuration.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Configure Redis BEFORE AddConfigurationApi()
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "ConfigApi_";
});

// This will use the Redis cache instead of in-memory
builder.AddConfigurationApi();

var app = builder.Build();
```

#### SQL Server

```bash
dotnet add package Microsoft.Extensions.Caching.SqlServer
```

```csharp
using Corp.Api.Configuration.Lib.Extensions;

var builder = WebApplication.CreateBuilder(args);

// Configure SQL Server cache BEFORE AddConfigurationApi()
builder.Services.AddDistributedSqlServerCache(options =>
{
    options.ConnectionString = builder.Configuration.GetConnectionString("CacheDb");
    options.SchemaName = "dbo";
    options.TableName = "ConfigurationCache";
});

// This will use the SQL Server cache instead of in-memory
builder.AddConfigurationApi();

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
/// {
//     options.Configuration = "localhost:6379";
//     options.InstanceName = "ConfigApi_";
// });

// Register Configuration API services
builder.AddConfigurationApi();

var app = builder.Build();

// Example endpoint using the Configuration API
app.MapGet("/config/{appName}", async (
    string appName,
    IConfigurationService configService,
    IApplicationService appService) =>
{
    try
    {
        // Verify application exists
        var application = await appService.GetByNameAsync(appName);
        if (application == null)
        {
            return Results.NotFound($"Application '{appName}' not found");
        }

        // Get cached configurations
        var configs = await configService.GetByApplicationNameAsync(appName);
        
        return Results.Ok(new
        {
            Application = application,
            Configurations = configs,
            Count = configs?.Count ?? 0
        });
    }
    catch (ApiException ex)
    {
        return Results.Problem(
            title: "Configuration API Error",
            detail: ex.Message,
            statusCode: (int)ex.StatusCode);
    }
});

// Health check endpoint
app.MapGet("/health", async (IHeartbeatService heartbeat) =>
{
    try
    {
        var serverTime = await heartbeat.GetHeartbeatAsync();
        return Results.Ok(new { Status = "Healthy", ServerTime = serverTime });
    }
    catch
    {
        return Results.Problem(title: "Unhealthy", statusCode: 503);
    }
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
| 10.0.0-beta-2 | Current version targeting .NET 10 |

---

## License

Copyright © Sedgwick Consumer Claims. All rights reserved.
