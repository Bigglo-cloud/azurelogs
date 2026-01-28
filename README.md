# Developer Implementation Guide: Migrating Legacy Logging to Azure Application Insights

**Version:** 2.0  
**Target Audience:** Mid-level C# .NET Core Developer  
**Technology Stack:** .NET Core, Azure Application Insights, Log Analytics  
**Last Updated:** January 28, 2026

---

## Table of Contents

1. [Prerequisites & Setup](#prerequisites--setup)
2. [Security & Compliance](#security--compliance)
3. [Implementation Logic](#implementation-logic)
4. [Async/Performance Benefits](#asyncperformance-benefits)
5. [Testing & Validation](#testing--validation)
6. [Troubleshooting](#troubleshooting)
7. [Advanced Topics](#advanced-topics)
8. [Cost Optimization](#cost-optimization)
9. [FAQ](#frequently-asked-questions)

---

## Overview

This guide covers the migration from a legacy MySQL-based logging system to Azure Application Insights for a Travel API (.NET Core). The migration implements an asynchronous "fire-and-forget" pattern to eliminate performance bottlenecks.

### Key Technical Requirements

1. **Asynchronous logging** to prevent blocking API threads
2. **Preserve stringified JSON** fields (requestModel, resultModel) without parsing in C#
3. **Enable KQL-based parsing** for geo-coordinate extraction by Data Architects
4. **GDPR compliance** with PII data handling
5. **Production-grade error handling** and resilience

### Architecture Overview
<img width="8191" height="1203" alt="Start Decision Options Flow-2026-01-28-182637" src="https://github.com/user-attachments/assets/206df9e1-e8ea-4eae-9392-d21411d9f0d2" />

  
```
Components:
- API Controller → Business Logic → ApiTelemetryService
- TelemetryClient → In-Memory Buffer → Batch Processing
- Azure Application Insights ← HTTP POST (every 30s or 500 items)
```

---

## Prerequisites & Setup

### Step 1: Install Required NuGet Packages

Open your terminal in the project root directory and run:

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
```

**Note:** 
- `Microsoft.ApplicationInsights.AspNetCore` includes TelemetryClient and all ASP.NET Core integration
- `Azure.Identity` and `Azure.Security.KeyVault.Secrets` are required for secure credential management

---

### Step 2: Environment-Specific Configuration

**CRITICAL:** Never commit connection strings to source control.

#### Development Environment

Create `appsettings.Development.json` (add to `.gitignore`):

```json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=dev-instrumentation-key;IngestionEndpoint=https://westeurope.in.applicationinsights.azure.com/;LiveEndpoint=https://westeurope.livediagnostics.monitor.azure.com/"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.ApplicationInsights": "Debug"
    }
  }
}
```

#### Production Environment (Azure Key Vault - Recommended)

Create `appsettings.Production.json`:

```json
{
  "KeyVault": {
    "VaultUri": "https://your-keyvault-name.vault.azure.net/"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Error"
    }
  }
}
```

Store the connection string in Azure Key Vault:
```bash
az keyvault secret set \
  --vault-name your-keyvault-name \
  --name AppInsights-ConnectionString \
  --value "InstrumentationKey=prod-key;IngestionEndpoint=https://westeurope.in.applicationinsights.azure.com/..."
```

---

### Step 3: Network Prerequisites

Application Insights requires outbound HTTPS access to Azure endpoints:

| Purpose | Endpoint | Port |
|---------|----------|------|
| Telemetry Ingestion | `https://{region}.in.applicationinsights.azure.com` | 443 |
| Live Metrics | `https://{region}.livediagnostics.monitor.azure.com` | 443 |
| Profiler/Snapshot | `https://{region}.profiler.monitor.azure.com` | 443 |

**For on-premise/VPN environments:**
1. Add these domains to your firewall whitelist
2. Verify proxy doesn't block HTTPS connections
3. Test connectivity:
   ```bash
   curl -v https://westeurope.in.applicationinsights.azure.com
   ```

---

### Step 4: Azure RBAC Permissions

Developers need the following Azure roles:

**On Application Insights resource:**
- `Monitoring Metrics Publisher` - to send telemetry
- `Monitoring Reader` - to view Live Metrics

**On Log Analytics workspace:**
- `Log Analytics Reader` - to run KQL queries

**Grant permissions via Azure CLI:**
```bash
az role assignment create \
  --assignee user@yourdomain.com \
  --role "Monitoring Metrics Publisher" \
  --scope /subscriptions/{subscription-id}/resourceGroups/{resource-group}/providers/Microsoft.Insights/components/{app-insights-name}
```

---

### Step 5: Register Application Insights in Program.cs

**For .NET 6+ (Minimal Hosting Model):**

```csharp
using Microsoft.ApplicationInsights.AspNetCore.Extensions;
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var builder = WebApplication.CreateBuilder(args);

// Configure Application Insights based on environment
if (builder.Environment.IsProduction())
{
    // Production: Load connection string from Azure Key Vault
    var keyVaultUri = new Uri(builder.Configuration["KeyVault:VaultUri"]);
    var credential = new DefaultAzureCredential();
    var secretClient = new SecretClient(keyVaultUri, credential);
    var connectionString = await secretClient.GetSecretAsync("AppInsights-ConnectionString");
    
    builder.Services.AddApplicationInsightsTelemetry(new ApplicationInsightsServiceOptions
    {
        ConnectionString = connectionString.Value.Value,
        EnableAdaptiveSampling = true,
        EnablePerformanceCounterCollectionModule = true
    });
}
else
{
    // Development: Load from appsettings
    builder.Services.AddApplicationInsightsTelemetry(new ApplicationInsightsServiceOptions
    {
        ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"],
        EnableAdaptiveSampling = false, // Capture all events in DEV
        EnableDebugLogger = true // Console output for debugging
    });
}

// Register custom telemetry service
builder.Services.AddSingleton<IApiTelemetryService, ApiTelemetryService>();

// Register your existing services
builder.Services.AddControllers();

var app = builder.Build();

// Configure middleware pipeline
app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**For .NET Core 3.1/5.0 (Startup.cs pattern):**

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddApplicationInsightsTelemetry(Configuration["ApplicationInsights:ConnectionString"]);
    services.AddSingleton<IApiTelemetryService, ApiTelemetryService>();
    services.AddControllers();
}
```

---

## Security & Compliance

### ⚠️ CRITICAL: GDPR & Data Privacy

Application Insights processes data globally and may store it in Microsoft datacenters. Proper data handling is mandatory for GDPR compliance.

#### What NOT to Log

**Never log the following PII without appropriate safeguards:**
- Full email addresses → Hash or use user IDs
- Credit card numbers → Never log, even partially
- Passwords, tokens, API keys → Redact completely
- Full IP addresses → Mask last octet (e.g., 192.168.1.xxx)
- Personal names in free text
- Social security numbers, passport IDs

#### Data Scrubbing Implementation

Create a telemetry initializer to automatically scrub PII:

```csharp
using Microsoft.ApplicationInsights.Channel;
using Microsoft.ApplicationInsights.Extensibility;
using System.Text.RegularExpressions;

namespace YourTravelApi.Telemetry
{
    public class PiiScrubbingTelemetryInitializer : ITelemetryInitializer
    {
        private static readonly Regex EmailRegex = new(@"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", RegexOptions.Compiled);
        private static readonly Regex CardRegex = new(@"\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b", RegexOptions.Compiled);
        private static readonly Regex PhoneRegex = new(@"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b", RegexOptions.Compiled);
        private static readonly Regex IpRegex = new(@"\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b", RegexOptions.Compiled);
        
        public void Initialize(ITelemetry telemetry)
        {
            if (telemetry is ISupportProperties propertiesTelemetry)
            {
                foreach (var kvp in propertiesTelemetry.Properties.ToList())
                {
                    var scrubbedValue = kvp.Value;
                    
                    // Redact email addresses
                    scrubbedValue = EmailRegex.Replace(scrubbedValue, "[EMAIL_REDACTED]");
                    
                    // Redact credit card numbers
                    scrubbedValue = CardRegex.Replace(scrubbedValue, "[CARD_REDACTED]");
                    
                    // Redact phone numbers
                    scrubbedValue = PhoneRegex.Replace(scrubbedValue, "[PHONE_REDACTED]");
                    
                    // Mask IP addresses (keep first 3 octets)
                    scrubbedValue = IpRegex.Replace(scrubbedValue, match =>
                    {
                        var parts = match.Value.Split('.');
                        return $"{parts[0]}.{parts[1]}.{parts[2]}.xxx";
                    });
                    
                    propertiesTelemetry.Properties[kvp.Key] = scrubbedValue;
                }
            }
        }
    }
}
```

**Register the initializer in Program.cs:**

```csharp
builder.Services.AddSingleton<ITelemetryInitializer, PiiScrubbingTelemetryInitializer>();
```

#### Azure Data Residency

Ensure compliance by selecting an appropriate Azure region:

**For GDPR compliance:**
- West Europe (Netherlands)
- North Europe (Ireland)
- Germany West Central (Frankfurt)

**Verify your connection string uses the correct region:**
```
InstrumentationKey=xxx;IngestionEndpoint=https://westeurope.in.applicationinsights.azure.com/
```

#### Data Retention Policy

Set appropriate retention based on compliance requirements:

```bash
# Set retention to 90 days (default = 90, max = 730)
az monitor app-insights component update \
  --app {app-insights-name} \
  --resource-group {resource-group} \
  --retention-time 90
```

**GDPR Consideration:** For data subject requests (right to erasure), use Log Analytics purge API:
```bash
az monitor log-analytics workspace data-export create \
  --workspace-name {workspace-name} \
  --resource-group {resource-group} \
  --name purge-request-{user-id} \
  --table customEvents \
  --filter "customDimensions.userId == '{user-id}'"
```

---

## Implementation Logic

### Understanding the Old vs. New Approach

#### OLD (Synchronous MySQL Insert)
```csharp
// ❌ Blocking operation - waits for database write
void LogToMySQL(ApiLogModel log)
{
    using var connection = new MySqlConnection(connectionString);
    connection.Open();
    var command = new MySqlCommand("INSERT INTO ApiLogs ...", connection);
    command.ExecuteNonQuery(); // Blocks thread until complete (~50-200ms)
}
```

**Problems:**
- Thread blocking for 50-200ms per request
- Database load impacts API responsiveness
- No built-in retry mechanism
- Scaling requires database capacity planning

#### NEW (Fire-and-Forget with Application Insights)
```csharp
// ✅ Non-blocking - buffered and batched automatically
void LogToAppInsights(ApiLogModel log)
{
    _telemetryClient.TrackEvent("ApiRequest", customDimensions);
    // Returns immediately (<1ms) - telemetry sent asynchronously in background
}
```

**Benefits:**
- Non-blocking (<1ms overhead)
- Automatic batching (every 30s or 500 items)
- Built-in retry with exponential backoff
- Auto-scales with Azure infrastructure

---

### Complete Implementation Code

Create the telemetry service with production-grade error handling:

```csharp
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;
using System.Diagnostics;

namespace YourTravelApi.Services
{
    public interface IApiTelemetryService
    {
        void LogApiRequest(ApiLogModel logData);
        void TrackMetric(string metricName, double value, IDictionary<string, string> properties = null);
    }

    public class ApiTelemetryService : IApiTelemetryService
    {
        private readonly TelemetryClient _telemetryClient;
        private readonly ILogger<ApiTelemetryService> _logger;
        private const int MAX_DIMENSION_LENGTH = 8000; // Safe margin below 8192 limit
        
        public ApiTelemetryService(
            TelemetryClient telemetryClient,
            ILogger<ApiTelemetryService> logger)
        {
            _telemetryClient = telemetryClient ?? throw new ArgumentNullException(nameof(telemetryClient));
            _logger = logger ?? throw new ArgumentNullException(nameof(logger));
        }

        public void LogApiRequest(ApiLogModel logData)
        {
            // CRITICAL: Logging failures must never crash the application
            try
            {
                // Validate input
                if (logData == null)
                {
                    _telemetryClient.TrackTrace(
                        "LogApiRequest called with null data", 
                        SeverityLevel.Warning);
                    return;
                }

                // Validate critical fields
                if (string.IsNullOrWhiteSpace(logData.Url))
                {
                    _telemetryClient.TrackTrace(
                        $"Missing URL in log data for dcKey: {logData.DcKey}", 
                        SeverityLevel.Warning);
                }

                // Create custom dimensions dictionary with pre-allocated capacity
                var customDimensions = new Dictionary<string, string>(12)
                {
                    { "dcKey", logData.DcKey.ToString() },
                    { "dcType", logData.DcType ?? string.Empty },
                    { "sourceType", logData.SourceType ?? string.Empty },
                    { "sourceId", logData.SourceId.ToString() },
                    { "url", logData.Url ?? string.Empty },
                    { "triggerType", logData.TriggerType ?? string.Empty },
                    { "requestType", logData.RequestType ?? string.Empty },
                    
                    // CRITICAL: Pass stringified JSON as-is (no parsing in C#)
                    // This allows Data Architects to use parse_json() in KQL
                    { "requestModel", TruncateIfNeeded(logData.RequestModel, "requestModel") },
                    
                    { "resultType", logData.ResultType ?? string.Empty },
                    
                    // CRITICAL: Pass stringified JSON as-is (no parsing in C#)
                    { "resultModel", TruncateIfNeeded(logData.ResultModel, "resultModel") },
                    
                    { "responseCode", logData.ResponseCode?.ToString() ?? string.Empty }
                };

                // Track the event - fires asynchronously
                _telemetryClient.TrackEvent("TravelApiRequest", customDimensions);

                // DO NOT call Flush() here - it blocks the thread
                // Let Application Insights handle batching automatically
            }
            catch (Exception ex)
            {
                // Log telemetry failure but don't propagate exception
                // This ensures API business logic continues even if logging fails
                _logger.LogWarning(ex, "Failed to send telemetry for dcKey: {DcKey}", logData?.DcKey);
                
                // Fallback: Write to trace for debugging
                System.Diagnostics.Trace.WriteLine($"Telemetry failed: {ex.Message}");
            }
        }

        public void TrackMetric(string metricName, double value, IDictionary<string, string> properties = null)
        {
            try
            {
                _telemetryClient.TrackMetric(metricName, value, properties);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Failed to track metric: {MetricName}", metricName);
            }
        }

        /// <summary>
        /// Truncates string to fit within Application Insights dimension limit (8,192 chars)
        /// </summary>
        private string TruncateIfNeeded(string value, string fieldName)
        {
            if (string.IsNullOrEmpty(value))
                return string.Empty;
            
            if (value.Length > MAX_DIMENSION_LENGTH)
            {
                var truncated = value.Substring(0, MAX_DIMENSION_LENGTH);
                
                // Log truncation for monitoring
                _telemetryClient.TrackTrace(
                    $"Truncated {fieldName}: original length {value.Length} chars", 
                    SeverityLevel.Warning,
                    new Dictionary<string, string>
                    {
                        { "fieldName", fieldName },
                        { "originalLength", value.Length.ToString() },
                        { "truncatedLength", MAX_DIMENSION_LENGTH.ToString() }
                    });
                
                return truncated + "...[TRUNCATED]";
            }
            
            return value;
        }
    }

    // Model class matching your JSON structure
    public class ApiLogModel
    {
        public long DcKey { get; set; }
        public string DcType { get; set; }
        public string SourceType { get; set; }
        public int SourceId { get; set; }
        public string Url { get; set; }
        public string TriggerType { get; set; }
        public string RequestType { get; set; }
        
        /// <summary>
        /// IMPORTANT: This is a STRING containing JSON, not a C# object
        /// Do not deserialize - pass as-is for KQL parsing
        /// </summary>
        public string RequestModel { get; set; }
        
        public string ResultType { get; set; }
        
        /// <summary>
        /// IMPORTANT: This is a STRING containing JSON, not a C# object
        /// Do not deserialize - pass as-is for KQL parsing
        /// </summary>
        public string ResultModel { get; set; }
        
        public int? ResponseCode { get; set; }
    }
}
```

---

### Register Services in Dependency Injection

**In `Program.cs` (.NET 6+):**

```csharp
// Application Insights already registered in Step 5
// builder.Services.AddApplicationInsightsTelemetry(...);

// Register PII scrubbing
builder.Services.AddSingleton<ITelemetryInitializer, PiiScrubbingTelemetryInitializer>();

// Register telemetry service
builder.Services.AddSingleton<IApiTelemetryService, ApiTelemetryService>();
```

---

### Usage in API Controller

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Text.Json;

[ApiController]
[Route("api/[controller]")]
public class SearchAvailabilityController : ControllerBase
{
    private readonly IApiTelemetryService _telemetryService;
    private readonly ILogger<SearchAvailabilityController> _logger;

    public SearchAvailabilityController(
        IApiTelemetryService telemetryService,
        ILogger<SearchAvailabilityController> logger)
    {
        _telemetryService = telemetryService;
        _logger = logger;
    }

    [HttpPost]
    public async Task<IActionResult> SearchAvailability([FromBody] SearchRequest request)
    {
        var logData = new ApiLogModel
        {
            DcKey = GenerateDcKey(), // Your key generation logic
            DcType = "ApiLogging",
            SourceType = "agency",
            SourceId = request.AgencyId,
            Url = $"{Request.Scheme}://{Request.Host}{Request.Path}",
            TriggerType = "OWTApi.Triggers.Handlers.agency.SearchAvailability",
            RequestType = "OWTWeb.OWTDB.Models.OWTApi.SearchAvailabilityRequest",
            
            // CRITICAL: Serialize to JSON string, do NOT parse or modify
            RequestModel = JsonSerializer.Serialize(request, new JsonSerializerOptions 
            { 
                WriteIndented = false // Compact JSON to save space
            }),
            
            ResultType = "OWTWeb.OWTDB.Models.OWTApi.SearchAvailabilityResponse"
        };

        try
        {
            // Execute business logic
            var result = await ProcessSearchAvailability(request);
            
            // Add result data before logging
            logData.ResultModel = JsonSerializer.Serialize(result, new JsonSerializerOptions 
            { 
                WriteIndented = false 
            });
            logData.ResponseCode = 200;
            
            // Fire-and-forget logging (non-blocking)
            _telemetryService.LogApiRequest(logData);
            
            return Ok(result);
        }
        catch (ValidationException vex)
        {
            logData.ResponseCode = 400;
            logData.ResultModel = JsonSerializer.Serialize(new { error = vex.Message });
            _telemetryService.LogApiRequest(logData);
            
            return BadRequest(new { error = vex.Message });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error in SearchAvailability");
            
            logData.ResponseCode = 500;
            logData.ResultModel = JsonSerializer.Serialize(new { error = "Internal server error" });
            _telemetryService.LogApiRequest(logData);
            
            return StatusCode(500, new { error = "Internal server error" });
        }
    }

    private long GenerateDcKey()
    {
        // Example: timestamp-based key
        return long.Parse(DateTime.UtcNow.ToString("yyyyMMddHHmmssfff") + Random.Shared.Next(1000, 9999));
    }
}
```

---

## Async/Performance Benefits

### Why TelemetryClient is Superior to Synchronous MySQL Inserts

| Aspect | Old MySQL Insert | New Application Insights |
|--------|------------------|--------------------------|
| **Execution Model** | Synchronous - blocks thread | Asynchronous - fire-and-forget |
| **Network I/O** | Waits for DB acknowledgment | Buffered in-memory queue |
| **Batching** | One call = one insert | Auto-batched every 30 seconds |
| **Thread Blocking** | Yes (~50-200ms per insert) | No (~<1ms overhead) |
| **Backpressure** | Database load affects API | Isolated - Azure handles load |
| **Retry Logic** | Manual implementation required | Built-in exponential backoff |
| **Scaling** | Vertical (larger DB instance) | Horizontal (Azure auto-scales) |

### How TelemetryClient Works Under the Hood

<img width="5694" height="5679" alt="Untitled diagram-2026-01-28-182105" src="https://github.com/user-attachments/assets/36069ac9-4b82-4559-b499-9df16fd54b0f" />


Flow:
1. TrackEvent() called → Add to in-memory channel (~microseconds)
2. Background thread batches events (up to 500 items or 30 seconds)
3. HTTP POST to Azure Application Insights
4. If failure → Exponential backoff retry (2s, 4s, 8s, 16s, 32s)
5. If persistent failure → Drop oldest events to prevent memory overflow
```

**Performance Impact:**

- **Before (MySQL):** 100ms+ per request (database insert time)
- **After (App Insights):** <1ms per request (in-memory queue add)
- **Result:** ~100x faster API response times for logging operations

**Memory Impact:**

- **Buffer size:** ~10-50 MB for typical usage
- **Max buffer:** 500,000 items (configurable)
- **Overflow behavior:** Oldest events dropped with warning logged

**Important Notes:**

- ✅ **DO NOT** call `_telemetryClient.Flush()` in request handlers - it blocks the thread
- ✅ **DO** call `Flush()` only during application shutdown
- ✅ Default batching behavior is optimal for 99% of scenarios
- ✅ Application Insights has built-in adaptive sampling for high-volume scenarios

---

## Testing & Validation

### Unit Testing

Create comprehensive unit tests to ensure telemetry service behaves correctly:

```csharp
using Xunit;
using Moq;
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;
using Microsoft.ApplicationInsights.Extensibility;
using Microsoft.Extensions.Logging;

namespace YourTravelApi.Tests
{
    public class ApiTelemetryServiceTests
    {
        private readonly Mock<TelemetryClient> _mockTelemetryClient;
        private readonly Mock<ILogger<ApiTelemetryService>> _mockLogger;
        private readonly ApiTelemetryService _service;

        public ApiTelemetryServiceTests()
        {
            var telemetryConfig = new TelemetryConfiguration();
            _mockTelemetryClient = new Mock<TelemetryClient>(telemetryConfig);
            _mockLogger = new Mock<ILogger<ApiTelemetryService>>();
            _service = new ApiTelemetryService(_mockTelemetryClient.Object, _mockLogger.Object);
        }

        [Fact]
        public void LogApiRequest_WithValidData_TrackEventCalled()
        {
            // Arrange
            var logData = new ApiLogModel
            {
                DcKey = 2026011418003096623,
                DcType = "ApiLogging",
                SourceType = "agency",
                SourceId = 109,
                Url = "https://owtapi.azurewebsites.net/api/SearchAvailability",
                RequestModel = "{\"from\":\"[25.08,55.14]\",\"to\":\"DXB\"}",
                ResponseCode = 200
            };

            // Act
            _service.LogApiRequest(logData);

            // Assert
            _mockTelemetryClient.Verify(
                x => x.TrackEvent(
                    "TravelApiRequest",
                    It.Is<IDictionary<string, string>>(d => 
                        d["dcKey"] == "2026011418003096623" &&
                        d["sourceType"] == "agency" &&
                        d["responseCode"] == "200"),
                    null),
                Times.Once);
        }

        [Fact]
        public void LogApiRequest_WithNullData_DoesNotThrow()
        {
            // Act & Assert
            var exception = Record.Exception(() => _service.LogApiRequest(null));
            Assert.Null(exception); // Should handle gracefully
        }

        [Fact]
        public void LogApiRequest_WithOversizedJson_TruncatesCorrectly()
        {
            // Arrange
            var largeJson = new string('x', 10000);
            var logData = new ApiLogModel
            {
                DcKey = 123,
                RequestModel = largeJson
            };

            // Act
            _service.LogApiRequest(logData);

            // Assert
            _mockTelemetryClient.Verify(
                x => x.TrackEvent(
                    "TravelApiRequest",
                    It.Is<IDictionary<string, string>>(d => 
                        d["requestModel"].Length <= 8020 && // 8000 + "[TRUNCATED]"
                        d["requestModel"].EndsWith("...[TRUNCATED]")),
                    null),
                Times.Once);
        }

        [Fact]
        public void LogApiRequest_WithMissingUrl_LogsWarning()
        {
            // Arrange
            var logData = new ApiLogModel
            {
                DcKey = 123,
                Url = null // Missing URL
            };

            // Act
            _service.LogApiRequest(logData);

            // Assert
            _mockTelemetryClient.Verify(
                x => x.TrackTrace(
                    It.Is<string>(s => s.Contains("Missing URL")),
                    SeverityLevel.Warning,
                    It.IsAny<IDictionary<string, string>>()),
                Times.Once);
        }
    }
}
```

---

### Integration Testing

Test the complete flow with TestHost:

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Xunit;

namespace YourTravelApi.IntegrationTests
{
    public class TelemetryIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
    {
        private readonly WebApplicationFactory<Program> _factory;

        public TelemetryIntegrationTests(WebApplicationFactory<Program> factory)
        {
            _factory = factory;
        }

        [Fact]
        public async Task SearchAvailability_LogsTelemetryCorrectly()
        {
            // Arrange
            var client = _factory.WithWebHostBuilder(builder =>
            {
                builder.ConfigureTestServices(services =>
                {
                    // Use in-memory telemetry channel for testing
                    services.AddSingleton<ITelemetryChannel, InMemoryTelemetryChannel>();
                });
            }).CreateClient();

            var request = new SearchRequest
            {
                From = "[25.085598897065,55.144039392471]",
                To = "DXB",
                Adults = 4
            };

            // Act
            var response = await client.PostAsJsonAsync("/api/SearchAvailability", request);

            // Assert
            Assert.True(response.IsSuccessStatusCode);

            // Verify telemetry was sent
            var telemetryChannel = _factory.Services.GetService<ITelemetryChannel>() as InMemoryTelemetryChannel;
            var events = telemetryChannel.SentItems.OfType<EventTelemetry>();

            Assert.Contains(events, e =>
                e.Name == "TravelApiRequest" &&
                e.Properties["url"].Contains("SearchAvailability") &&
                e.Properties["responseCode"] == "200");
        }
    }
}
```

---

### Step 1: Check Live Metrics (Real-Time Validation)

1. **Deploy your application** to Azure App Service or run locally.

2. **Navigate to Azure Portal:**
   - Go to your Application Insights resource
   - Click **"Live Metrics"** in the left sidebar

3. **Trigger an API request** (e.g., POST to `/api/SearchAvailability`).

4. **Verify in Live Metrics:**
   - You should see incoming requests in the "Incoming Requests" chart
   - Custom events will appear under "Custom Events" section
   - Look for event name: `TravelApiRequest`

**Expected Result:**  
Within 1-2 seconds, you'll see your custom event appear with a count of 1.


### Step 2: Query Logs in Log Analytics

**Note:** There's a 2-5 minute ingestion delay for Log Analytics queries.

1. Navigate to your Application Insights resource → **"Logs"**

2. Run this KQL query to see your logged events:

```kusto
customEvents
| where name == "TravelApiRequest"
| extend 
    dcKey = tostring(customDimensions.dcKey),
    url = tostring(customDimensions.url),
    responseCode = tostring(customDimensions.responseCode),
    requestModelRaw = tostring(customDimensions.requestModel)
| project timestamp, dcKey, url, responseCode, requestModelRaw
| order by timestamp desc
| take 10
```

---

### Step 3: Parse Geo-Coordinates (Data Architect's KQL Query)

This demonstrates how your Data Architect will extract geo-coordinates from the stringified JSON:

```kusto
customEvents
| where name == "TravelApiRequest"
| where timestamp > ago(1h)
| extend requestModelRaw = tostring(customDimensions.requestModel)
| extend requestJson = parse_json(requestModelRaw)
| extend 
    fromCoordinates = tostring(requestJson.from),
    toLocation = tostring(requestJson.to),
    travelDate = todatetime(requestJson.travel_date),
    adults = toint(requestJson.adults),
    children = toint(requestJson.children)
| project 
    timestamp, 
    fromCoordinates, 
    toLocation, 
    travelDate, 
    adults,
    children
| order by timestamp desc
```

**Why this works:**
- You passed `requestModel` as a **raw string** (no C# parsing)
- KQL's `parse_json()` function handles the parsing server-side in Azure
- This saves CPU cycles in your API and leverages Azure's query engine
- Data Architects can extract any field from the JSON without code changes

---

### Step 4: Validate Custom Dimensions

Run this query to inspect all custom dimensions for a single event:

```kusto
customEvents
| where name == "TravelApiRequest"
| take 1
| project customDimensions
```

**Expected Output:**
```json
{
  "dcKey": "2026011418003096623",
  "dcType": "ApiLogging",
  "sourceType": "agency",
  "sourceId": "109",
  "url": "https://owtapi.azurewebsites.net/api/SearchAvailability",
  "triggerType": "OWTApi.Triggers.Handlers.agency.SearchAvailability",
  "requestType": "OWTWeb.OWTDB.Models.OWTApi.SearchAvailabilityRequest",
  "requestModel": "{\n  \"from\": \"[25.085598897065,55.144039392471]\",\n  \"to\": \"DXB\", ...",
  "resultType": "OWTWeb.OWTDB.Models.OWTApi.SearchAvailabilityResponse",
  "resultModel": "{\n  \"vehicles\": [...",
  "responseCode": "200"
}
```

---

## Troubleshooting

### Issue 1: No Events Appearing in Live Metrics

```

Decision flow:
<img width="3870" height="3205" alt="Start Decision Options Flow-2026-01-28-182921" src="https://github.com/user-attachments/assets/d09c2fc9-3539-4211-ad87-1219b60ccbd7" />

1. Check connection string → 2. Check network → 3. Check DI registration → 4. Force flush test
```

**Possible Causes:**
- Incorrect connection string in `appsettings.json`
- Application Insights not registered in `Program.cs`
- Firewall blocking outbound HTTPS to Azure
- Network proxy interfering with connections

**Diagnostic Steps:**

1. **Verify connection string format:**
   ```csharp
   // Add to Program.cs temporarily
   var connectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
   Console.WriteLine($"AI Connection String: {connectionString?.Substring(0, 50)}...");
   ```

2. **Test network connectivity:**
   ```bash
   # From your server/container
   curl -v https://westeurope.in.applicationinsights.azure.com
   
   # Should return 200 OK or redirect
   # If timeout or connection refused → firewall issue
   ```

3. **Enable debug logging:**
   ```json
   // appsettings.Development.json
   {
     "Logging": {
       "LogLevel": {
         "Microsoft.ApplicationInsights": "Debug"
       }
     }
   }
   ```

4. **Force flush to test connectivity:**
   ```csharp
   // Temporary diagnostic code
   _telemetryClient.TrackTrace("Test connection", SeverityLevel.Information);
   _telemetryClient.Flush(); // Force immediate send
   await Task.Delay(5000); // Wait for transmission
   ```

---

### Issue 2: Custom Dimensions Missing or Truncated

**Cause:** Application Insights has limits:
- Max 100 custom dimensions per event
- Max 8,192 characters per dimension value

**Solution:**

The code already implements truncation (see `TruncateIfNeeded` method). To monitor truncations:

```kusto
// Find truncated events
traces
| where message contains "Truncated"
| extend 
    fieldName = tostring(customDimensions.fieldName),
    originalLength = toint(customDimensions.originalLength)
| summarize TruncationCount = count(), AvgOriginalSize = avg(originalLength) by fieldName
| order by TruncationCount desc
```

**If you see frequent truncations:**

1. **Reduce JSON payload size** in your application
2. **Compress JSON** before logging:
   ```csharp
   private string CompressJson(string json)
   {
       var bytes = Encoding.UTF8.GetBytes(json);
       using var output = new MemoryStream();
       using (var gzip = new GZipStream(output, CompressionMode.Compress))
       {
           gzip.Write(bytes, 0, bytes.Length);
       }
       return Convert.ToBase64String(output.ToArray());
   }
   ```
3. **Store full payloads in Blob Storage** and log only reference:
   ```csharp
   var blobUrl = await UploadToBlobStorage(largeJson);
   logData.RequestModel = $"{{\"blobUrl\":\"{blobUrl}\"}}";
   ```

---

### Issue 3: High Memory Usage

**Cause:** 
- Calling `Flush()` too frequently
- Buffer overflow due to high event volume
- Memory leak in custom code

**Diagnostic Query:**
```kusto
// Check event volume
customEvents
| where name == "TravelApiRequest"
| summarize EventsPerMinute = count() by bin(timestamp, 1m)
| order by timestamp desc
| take 100
```

**Solutions:**

1. **Remove all Flush() calls** except in shutdown handler:
   ```csharp
   // Program.cs
   var app = builder.Build();
   
   var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
   lifetime.ApplicationStopping.Register(() =>
   {
       var telemetryClient = app.Services.GetRequiredService<TelemetryClient>();
       telemetryClient.Flush();
       Task.Delay(5000).Wait(); // Allow time for flush
   });
   ```

2. **Enable adaptive sampling** for high-volume scenarios (>100,000 events/day):
   ```csharp
   builder.Services.AddApplicationInsightsTelemetry(options =>
   {
       options.EnableAdaptiveSampling = true;
       options.DeveloperMode = false;
   });
   
   builder.Services.Configure<TelemetryConfiguration>(config =>
   {
       config.DefaultTelemetrySink.TelemetryProcessorChainBuilder
           .UseAdaptiveSampling(maxTelemetryItemsPerSecond: 5)
           .Build();
   });
   ```

3. **Monitor buffer size:**
   ```csharp
   // Custom telemetry processor to monitor buffer
   public class BufferMonitorProcessor : ITelemetryProcessor
   {
       private static long _totalEvents = 0;
       
       public void Process(ITelemetry item)
       {
           Interlocked.Increment(ref _totalEvents);
           
           if (_totalEvents % 1000 == 0)
           {
               Console.WriteLine($"Total events processed: {_totalEvents}");
           }
           
           _next.Process(item);
       }
   }
   ```

---

### Issue 4: Events Not Parseable in KQL

**Problem:** `parse_json()` returns null for your requestModel field.

**Common Causes:**

1. **Escaped quotes in JSON:**
   ```kusto
   // Fix with replace_string
   customEvents
   | extend cleanJson = replace_string(tostring(customDimensions.requestModel), '\\\"', '"')
   | extend parsed = parse_json(cleanJson)
   ```

2. **Invalid JSON (missing brackets, trailing commas):**
   ```kusto
   // Diagnostic query to find invalid JSON
   customEvents
   | where name == "TravelApiRequest"
   | extend requestModelRaw = tostring(customDimensions.requestModel)
   | where parse_json(requestModelRaw) == ""
   | project timestamp, requestModelRaw
   | take 10
   ```

3. **Field doesn't exist in JSON:**
   ```kusto
   // Safe extraction with null handling
   customEvents
   | extend requestJson = parse_json(tostring(customDimensions.requestModel))
   | extend fromCoords = tostring(requestJson.from)
   | extend safeCoords = iif(isnull(fromCoords), "N/A", fromCoords)
   ```

---

### Issue 5: Slow Query Performance in Log Analytics

**Problem:** KQL queries taking >30 seconds to execute.

**Solutions:**

1. **Use time filters first:**
   ```kusto
   // ✅ GOOD - Filter by time first
   customEvents
   | where timestamp > ago(1h)
   | where name == "TravelApiRequest"
   
   // ❌ BAD - Scans entire table
   customEvents
   | where name == "TravelApiRequest"
   | where timestamp > ago(1h)
   ```

2. **Index frequently queried fields:**
   ```kusto
   // Project early to reduce data volume
   customEvents
   | where timestamp > ago(1h)
   | where name == "TravelApiRequest"
   | project timestamp, customDimensions
   | extend dcKey = tostring(customDimensions.dcKey)
   ```

3. **Use summarize for aggregations:**
   ```kusto
   // Instead of scanning all rows
   customEvents
   | where timestamp > ago(24h)
   | where name == "TravelApiRequest"
   | summarize 
       RequestCount = count(),
       AvgResponseCode = avg(toint(customDimensions.responseCode))
       by bin(timestamp, 1h)
   ```

---

## Advanced Topics

### Distributed Tracing for Microservices

If your Travel API calls other services, enable distributed tracing:

```csharp
using System.Diagnostics;

public class SearchAvailabilityController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> SearchAvailability([FromBody] SearchRequest request)
    {
        // Create activity for distributed tracing
        using var activity = new Activity("SearchAvailability").Start();
        activity.AddTag("user.agency_id", request.AgencyId.ToString());
        activity.AddTag("request.type", "availability");
        activity.AddTag("request.destination", request.To);
        
        try
        {
            // Call to Vehicle Service (automatically propagates trace context)
            var vehicles = await _httpClient.PostAsync(
                "https://vehicle-service/api/search", 
                content);
            
            // Call to Pricing Service (automatically propagates trace context)
            var pricing = await _httpClient.PostAsync(
                "https://pricing-service/api/calculate", 
                pricingContent);
            
            activity.AddTag("result.vehicle_count", vehicles.Count.ToString());
            
            return Ok(result);
        }
        catch (Exception ex)
        {
            activity.AddTag("error", true);
            activity.AddTag("error.message", ex.Message);
            throw;
        }
    }
}
```

**Query end-to-end trace in KQL:**

```kusto
// Find complete request flow across services
let traceId = "specific-operation-id";
dependencies
| where operation_Id == traceId
| union (requests | where operation_Id == traceId)
| project 
    timestamp, 
    itemType = iif(itemType == "dependency", "outbound", "inbound"),
    name, 
    operation_Name, 
    duration, 
    success,
    resultCode
| order by timestamp asc
```

**Visualize dependency graph:**
```kusto
dependencies
| where timestamp > ago(1h)
| summarize 
    CallCount = count(),
    AvgDuration = avg(duration),
    FailureRate = countif(success == false) * 100.0 / count()
    by target, name
| order by CallCount desc
```

---

### Custom Metrics for Business Intelligence

Beyond events, track business metrics for dashboards and alerts:

```csharp
public class ApiTelemetryService : IApiTelemetryService
{
    public void TrackSearchPerformed(SearchMetrics metrics)
    {
        // Track custom metrics for Power BI dashboards
        _telemetryClient.GetMetric("SearchRequests", "SourceType", "Destination")
            .TrackValue(1, metrics.SourceType, metrics.Destination);
        
        _telemetryClient.GetMetric("EstimatedRevenue", "Currency")
            .TrackValue(metrics.EstimatedValue, metrics.Currency);
        
        // Track response time percentiles
        _telemetryClient.TrackMetric(
            "SearchResponseTime", 
            metrics.Duration.TotalMilliseconds,
            new Dictionary<string, string> 
            {
                { "endpoint", metrics.Url },
                { "cache_hit", metrics.CacheHit.ToString() },
                { "vehicle_count", metrics.VehicleCount.ToString() }
            });
        
        // Track conversion funnel
        _telemetryClient.GetMetric("ConversionFunnel", "Stage")
            .TrackValue(1, "Search"); // "Search", "Select", "Book", "Payment"
    }
}
```

**Query metrics in KQL:**

```kusto
// Revenue trend by currency
customMetrics
| where name == "EstimatedRevenue"
| extend currency = tostring(customDimensions.Currency)
| summarize TotalRevenue = sum(value) by bin(timestamp, 1d), currency
| render timechart
```

```kusto
// P95 response time by endpoint
customMetrics
| where name == "SearchResponseTime"
| extend endpoint = tostring(customDimensions.endpoint)
| summarize P95ResponseTime = percentile(value, 95) by endpoint
| order by P95ResponseTime desc
```

---

### Azure Monitor Alerting

Set up proactive alerts for critical scenarios:

#### Alert 1: High Error Rate

```kusto
// Alert if error rate > 5% in last 5 minutes
customEvents
| where name == "TravelApiRequest"
| where timestamp > ago(5m)
| extend responseCode = toint(customDimensions.responseCode)
| summarize 
    TotalRequests = count(),
    Errors = countif(responseCode >= 500)
| extend ErrorRate = (Errors * 100.0) / TotalRequests
| where ErrorRate > 5
```

**Configure in Azure Portal:**
1. Go to Application Insights → Alerts → New alert rule
2. Select "Custom log search"
3. Paste the query above
4. Set threshold: Result count > 0
5. Add action group (email, SMS, webhook)

#### Alert 2: Missing Telemetry (Data Quality)

```kusto
// Alert if event volume drops below 80% of expected
let expected = 1000; // Expected events per hour based on historical data
customEvents
| where timestamp > ago(1h)
| where name == "TravelApiRequest"
| summarize ActualCount = count()
| where ActualCount < expected * 0.8
```

#### Alert 3: Geo-Coordinate Parsing Failures

```kusto
// Alert if more than 100 events have unparseable coordinates
customEvents
| where timestamp > ago(1h)
| where name == "TravelApiRequest"
| extend requestJson = parse_json(tostring(customDimensions.requestModel))
| extend fromCoords = tostring(requestJson.from)
| where isnull(fromCoords) or fromCoords == ""
| summarize FailedParses = count()
| where FailedParses > 100
```

#### Alert 4: High Latency

```kusto
// Alert if P95 response time > 2000ms
customMetrics
| where name == "SearchResponseTime"
| where timestamp > ago(15m)
| summarize P95 = percentile(value, 95)
| where P95 > 2000
```

---

## Cost Optimization

### Understanding Application Insights Pricing

**2026 Pricing (Pay-as-you-go):**
- First 5 GB/month: **Free**
- Beyond 5 GB: **$2.30 per GB**
- Data retention (90 days): Included
- Extended retention (91-730 days): $0.12 per GB/month

**Example Cost Calculation for Travel API:**

Assumptions:
- 1,000,000 requests/month
- Average event size: 5 KB (including requestModel and resultModel JSON)
- Monthly data ingestion: 1M × 5 KB = 5 GB

**Result: $0/month** (within free tier!)

**If you exceed 5 GB/month:**
- 10 GB/month → $11.50/month ($2.30 × 5 GB)
- 50 GB/month → $103.50/month ($2.30 × 45 GB)

---

### Cost Optimization Strategies

#### Strategy 1: Adaptive Sampling

Automatically reduce event volume during high-traffic periods:

```csharp
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.EnableAdaptiveSampling = true;
});

builder.Services.Configure<TelemetryConfiguration>(config =>
{
    config.DefaultTelemetrySink.TelemetryProcessorChainBuilder
        .UseAdaptiveSampling(
            maxTelemetryItemsPerSecond: 5, // Target rate
            excludedTypes: "Exception;Event", // Never sample errors
            includedTypes: "PageView;Request")
        .Build();
});
```

**How it works:**
- Dynamically adjusts sampling percentage based on volume
- Preserves statistical validity (all operation_Ids with same sampling rate)
- Never samples exceptions or errors

#### Strategy 2: Selective Logging

Log only business-critical events or a sample of successful requests:

```csharp
public void LogApiRequest(ApiLogModel logData)
{
    // Log all failed requests
    if (logData.ResponseCode >= 400)
    {
        _telemetryClient.TrackEvent("TravelApiRequest", customDimensions);
        return;
    }
    
    // Log only 10% of successful requests
    if (logData.DcKey % 10 == 0)
    {
        _telemetryClient.TrackEvent("TravelApiRequest", customDimensions);
    }
    
    // Always track metrics (much smaller than events)
    _telemetryClient.GetMetric("SearchRequests").TrackValue(1);
}
```

#### Strategy 3: Data Capping

Set a daily ingestion limit to prevent cost overruns:

```bash
# Set daily cap to 5 GB (prevents exceeding free tier)
az monitor app-insights component update \
  --app travel-api-insights \
  --resource-group travel-api-rg \
  --cap 5
```

**WARNING:** When cap is reached, no more data is ingested until next day. Use alerts to monitor approaching cap.

#### Strategy 4: Compress Large Payloads

For requestModel/resultModel > 1 KB, consider compression:

```csharp
private string CompressIfLarge(string json)
{
    if (json.Length < 1000)
        return json;
    
    var bytes = Encoding.UTF8.GetBytes(json);
    using var output = new MemoryStream();
    using (var gzip = new GZipStream(output, CompressionMode.Compress))
    {
        gzip.Write(bytes, 0, bytes.Length);
    }
    
    var compressed = Convert.ToBase64String(output.ToArray());
    
    // Only use if actually smaller
    return compressed.Length < json.Length 
        ? "GZIP:" + compressed 
        : json;
}
```

**Decompress in KQL:**
```kusto
customEvents
| extend requestModelRaw = tostring(customDimensions.requestModel)
| extend isCompressed = startswith(requestModelRaw, "GZIP:")
// Decompression requires custom function or external processing
```

#### Strategy 5: Archive to Blob Storage

For long-term retention, export old data to cheaper storage:

```bash
# Create continuous export to Blob Storage
az monitor app-insights component continuous-export create \
  --app travel-api-insights \
  --resource-group travel-api-rg \
  --dest-account myStorageAccount \
  --dest-container telemetry-archive \
  --record-types Event,Metric
```

**Cost comparison:**
- Application Insights: $2.30/GB/month
- Blob Storage (Cool tier): $0.01/GB/month
- **Savings: 99.6%** for archived data

---

### Monitoring Costs

Create a dashboard to track ingestion:

```kusto
// Daily ingestion volume
union *
| where timestamp > ago(30d)
| summarize DataSizeMB = sum(_BilledSize) / 1024 / 1024 by bin(timestamp, 1d)
| extend DataSizeGB = DataSizeMB / 1024
| extend EstimatedCost = iif(DataSizeGB > 5, (DataSizeGB - 5) * 2.30, 0.0)
| project timestamp, DataSizeGB, EstimatedCost
| render timechart
```

```kusto
// Top contributors to data volume
union *
| where timestamp > ago(7d)
| summarize DataSizeMB = sum(_BilledSize) / 1024 / 1024 by itemType
| order by DataSizeMB desc
```

---

## Frequently Asked Questions

### Q: Does Application Insights work offline?

**A:** No, Application Insights requires internet connectivity to Azure. However:
- Events are buffered in-memory (up to 50,000 items by default)
- When connectivity is restored, buffered events are sent automatically
- Persistent buffer can be enabled for longer outages:
  ```csharp
  builder.Services.Configure<TelemetryConfiguration>(config =>
  {
      config.TelemetryChannel.DeveloperMode = false;
      config.TelemetryChannel.EndpointAddress = "https://dc.services.visualstudio.com/v2/track";
  });
  ```

---

### Q: Can I use this with Docker/Kubernetes?

**A:** Yes. Pass the connection string via environment variables:

**Docker:**
```bash
docker run -e ApplicationInsights__ConnectionString="InstrumentationKey=xxx..." myapp
```

**Kubernetes:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: travel-api
spec:
  template:
    spec:
      containers:
      - name: api
        image: travelapi:latest
        env:
        - name: ApplicationInsights__ConnectionString
          valueFrom:
            secretKeyRef:
              name: app-insights-secret
              key: connection-string
```

**Create the secret:**
```bash
kubectl create secret generic app-insights-secret \
  --from-literal=connection-string="InstrumentationKey=xxx..."
```

---

### Q: How long does the migration take?

**A:** Typical timeline for a mid-level developer:

| Phase | Duration | Activities |
|-------|----------|-----------|
| **Day 1: Setup** | 2-3 hours | Install packages, configure environments, test connectivity |
| **Day 2: Implementation** | 4-6 hours | Write ApiTelemetryService, add error handling, replace MySQL calls |
| **Day 3: Testing** | 4-6 hours | Unit tests, integration tests, local validation |
| **Day 4: Staging** | 2-3 hours | Deploy to staging, smoke testing, performance validation |
| **Day 5: Production** | 1-2 hours | Production deployment, monitoring, parallel run verification |

**Total: 3-5 working days** (including testing and validation)

---

### Q: What happens to existing MySQL logs?

**A:** Recommended approach:

1. **Keep the MySQL table read-only** for historical queries
2. **Do NOT migrate historical data** to Application Insights (expensive and unnecessary)
3. **Optional:** Export historical data to Azure Blob Storage for archival
4. **Parallel run for 7 days** to verify data completeness:
   ```csharp
   // Temporary dual-logging during transition
   public void LogApiRequest(ApiLogModel logData)
   {
       // New system (primary)
       try
       {
           _telemetryClient.TrackEvent("TravelApiRequest", customDimensions);
       }
       catch (Exception ex)
       {
           _logger.LogWarning($"App Insights failed: {ex.Message}");
       }
       
       // Old system (fallback, to be removed after validation)
       try
       {
           _mysqlLogger.LogToDatabase(logData);
       }
       catch
       {
           // Ignore MySQL errors during transition
       }
   }
   ```
5. **After 7 days:** Remove MySQL logging code, archive old table

---

### Q: Does this work with .NET Framework 4.x?

**A:** Yes, but use a different package:

```bash
# For .NET Framework (not .NET Core)
Install-Package Microsoft.ApplicationInsights
Install-Package Microsoft.ApplicationInsights.Web
```

**Configuration (Web.config):**
```xml
<configuration>
  <appSettings>
    <add key="ApplicationInsights:InstrumentationKey" value="your-key-here" />
  </appSettings>
</configuration>
```

---

### Q: How do I handle high-traffic scenarios (1M+ requests/hour)?

**A:** Use adaptive sampling and metrics instead of events:

```csharp
// High traffic: Use metrics (much smaller than events)
public void LogApiRequest(ApiLogModel logData)
{
    // Track aggregated metrics (auto-summarized)
    _telemetryClient.GetMetric("SearchRequests", "SourceType", "ResponseCode")
        .TrackValue(1, logData.SourceType, logData.ResponseCode.ToString());
    
    // Only log full events for errors or 1% sample
    if (logData.ResponseCode >= 400 || Random.Shared.Next(100) == 0)
    {
        _telemetryClient.TrackEvent("TravelApiRequest", customDimensions);
    }
}
```

**Benefits:**
- Metrics use ~10x less storage than events
- Pre-aggregated for faster queries
- Still maintains statistical accuracy

---

### Q: Can I query data from multiple Application Insights resources?

**A:** Yes, using cross-resource queries:

```kusto
// Query both production and staging
union 
    app('travel-api-prod').customEvents,
    app('travel-api-staging').customEvents
| where name == "TravelApiRequest"
| extend environment = iif(app_name contains "prod", "Production", "Staging")
| summarize count() by environment, bin(timestamp, 1h)
```

---

### Q: What's the maximum request rate Application Insights can handle?

**A:** Application Insights can handle:
- **Ingestion:** Up to 32,000 events/second per instrumentation key
- **Queries:** Up to 1,000 concurrent queries
- **API calls:** 100 calls per 30 seconds per instrumentation key

For higher rates, use multiple instrumentation keys or metrics instead of events.

---

## Migration Checklist

Use this checklist to track your progress:

### Prerequisites
- [ ] Install `Microsoft.ApplicationInsights.AspNetCore` NuGet package
- [ ] Install `Azure.Identity` and `Azure.Security.KeyVault.Secrets` packages
- [ ] Create Application Insights resources in Azure (DEV, STAGING, PROD)
- [ ] Store connection strings in Azure Key Vault
- [ ] Configure `appsettings.Development.json` (add to `.gitignore`)
- [ ] Configure network firewall rules for Azure endpoints
- [ ] Verify RBAC permissions in Azure

### Security & Compliance
- [ ] Implement `PiiScrubbingTelemetryInitializer`
- [ ] Register PII scrubbing in DI container
- [ ] Verify data residency region (GDPR compliance)
- [ ] Set appropriate data retention policy (90 days)
- [ ] Document PII handling in privacy policy

### Implementation
- [ ] Create `ApiTelemetryService` class with error handling
- [ ] Implement `TruncateIfNeeded` method for large JSON payloads
- [ ] Create `ApiLogModel` class matching JSON structure
- [ ] Register `IApiTelemetryService` in DI container
- [ ] Update controllers to use `IApiTelemetryService`
- [ ] Verify `requestModel` and `resultModel` are passed as strings (no parsing)
- [ ] Remove old MySQL logging calls (after validation period)

### Testing
- [ ] Write unit tests for `ApiTelemetryService`
- [ ] Write integration tests with TestHost
- [ ] Test locally with Live Metrics
- [ ] Verify events appear in Log Analytics (allow 2-5 min delay)
- [ ] Run KQL queries to validate custom dimensions
- [ ] Confirm Data Architect can parse geo-coordinates with `parse_json()`
- [ ] Load test with expected production traffic

### Deployment
- [ ] Deploy to STAGING environment
- [ ] Smoke test all critical endpoints
- [ ] Verify telemetry in Live Metrics (STAGING)
- [ ] Run parallel logging (MySQL + App Insights) for 7 days in PROD
- [ ] Monitor costs during parallel run
- [ ] Compare MySQL vs App Insights data completeness
- [ ] Deploy to PRODUCTION (off-peak hours recommended)
- [ ] Monitor Live Metrics for first 30 minutes
- [ ] Disable MySQL logging after validation
- [ ] Archive old MySQL logs to Blob Storage

### Post-Deployment
- [ ] Create Azure Monitor dashboards
- [ ] Set up alerts (error rate, missing telemetry, high latency)
- [ ] Document KQL queries for common scenarios
- [ ] Train support team on Log Analytics basics
- [ ] Schedule knowledge transfer session with Data Architects
- [ ] Monitor costs weekly for first month
- [ ] Optimize sampling if costs exceed budget

---

## Appendix A: Glossary

| Term | Definition |
|------|----------|
| **Application Insights** | Azure service for application performance monitoring and telemetry collection |
| **Custom Dimensions** | Key-value pairs attached to telemetry for filtering and analysis |
| **DI (Dependency Injection)** | Design pattern for managing object dependencies and lifetimes |
| **Fire-and-Forget** | Asynchronous pattern where caller doesn't wait for operation completion |
| **Ingestion** | Process of sending telemetry data to Azure Application Insights |
| **KQL (Kusto Query Language)** | Query language for Azure Log Analytics and Application Insights |
| **Log Analytics** | Azure service for querying and analyzing log data |
| **PII (Personally Identifiable Information)** | Data that can identify an individual (email, name, address, etc.) |
| **Sampling** | Technique to reduce data volume by capturing only a percentage of events |
| **Telemetry** | Diagnostic data, metrics, and logs collected from applications |
| **TelemetryClient** | SDK class for sending telemetry to Application Insights |
| **Track*()** | Family of methods for sending different telemetry types (TrackEvent, TrackTrace, etc.) |

---

## Appendix B: Rollback Procedure

If you encounter critical issues after deployment, use this rollback plan:

### Immediate Rollback (< 5 minutes)

1. **Disable Application Insights:**
   ```csharp
   // Program.cs - Comment out Application Insights registration
   // builder.Services.AddApplicationInsightsTelemetry(...);
   ```

2. **Re-enable MySQL logging:**
   ```csharp
   // Uncomment old MySQL logging code
   _mysqlLogger.LogApiRequest(logData);
   ```

3. **Deploy hotfix immediately**

4. **Verify MySQL logging is working:**
   ```sql
   SELECT COUNT(*) FROM ApiLogs WHERE created_at > NOW() - INTERVAL 5 MINUTE;
   ```

---

### Partial Rollback (Dual Logging)

Keep both systems running while investigating issues:

```csharp
public void LogApiRequest(ApiLogModel logData)
{
    var loggedToAppInsights = false;
    
    // Try Application Insights first (new system)
    try
    {
        _telemetryClient.TrackEvent("TravelApiRequest", customDimensions);
        loggedToAppInsights = true;
    }
    catch (Exception ex)
    {
        _logger.LogWarning($"App Insights failed: {ex.Message}");
    }
    
    // Fallback to MySQL (old system)
    if (!loggedToAppInsights)
    {
        try
        {
            _mysqlLogger.LogToDatabase(logData);
        }
        catch (Exception ex)
        {
            _logger.LogError($"Both logging systems failed: {ex.Message}");
        }
    }
}
```

---

### Validation Before Full Cutover

Before disabling MySQL completely, verify:

- [ ] 7 days of parallel running without Application Insights failures
- [ ] Data completeness: Compare event counts in MySQL vs Application Insights
  ```sql
  -- MySQL count
  SELECT DATE(created_at), COUNT(*) FROM ApiLogs 
  WHERE created_at > DATE_SUB(NOW(), INTERVAL 7 DAY)
  GROUP BY DATE(created_at);
  ```
  ```kusto
  // Application Insights count
  customEvents
  | where timestamp > ago(7d)
  | where name == "TravelApiRequest"
  | summarize count() by bin(timestamp, 1d)
  ```
- [ ] KQL queries working correctly (geo-coordinate parsing)
- [ ] Dashboards and alerts configured
- [ ] Team trained on KQL basics
- [ ] Load tested with 2x normal traffic
- [ ] Cost monitoring shows expected ingestion volume

---

## Additional Resources

- [Application Insights Overview](https://docs.microsoft.com/azure/azure-monitor/app/app-insights-overview)
- [TelemetryClient API Reference](https://docs.microsoft.com/dotnet/api/microsoft.applicationinsights.telemetryclient)
- [Kusto Query Language (KQL) Reference](https://docs.microsoft.com/azure/data-explorer/kusto/query/)
- [Custom Events and Metrics](https://docs.microsoft.com/azure/azure-monitor/app/api-custom-events-metrics)
- [Application Insights Sampling](https://docs.microsoft.com/azure/azure-monitor/app/sampling)
- [GDPR Compliance in Azure](https://docs.microsoft.com/azure/compliance/gdpr/gdpr-guide)
- [Azure Monitor Pricing](https://azure.microsoft.com/pricing/details/monitor/)

---

**Document Version:** 2.0  
**Last Updated:** January 28, 2026  
**Maintained By:** Bigglo.pl - Arkady Zagdan  
**Feedback:** kontakt@bigglo.pl

---

**Document End**
