# API Logs Pipeline — Upstream Implementation Guide

> **Scope:** This document covers the **upstream** part of the pipeline — from the application sending logs to raw data landing in Azure Data Lake Gen2 in the correct format. The downstream part (Synapse, Power BI) is handled separately.

> **Audience:** Backend developers responsible for replacing MySQL log writes with Azure Queue Storage integration and building the collector Azure Function.

---

## Table of Contents

- [1. Architecture Overview](#1-architecture-overview)
- [2. Infrastructure Setup (Azure)](#2-infrastructure-setup-azure)
- [3. Component A: Application → Queue (Producer)](#3-component-a-application--queue-producer)
- [4. Component B: Queue → Data Lake (Collector Function)](#4-component-b-queue--data-lake-collector-function)
- [5. Data Lake Structure & File Format](#5-data-lake-structure--file-format)
- [6. Log Message Schema](#6-log-message-schema)
- [7. Error Handling & Resilience](#7-error-handling--resilience)
- [8. Monitoring & Alerts](#8-monitoring--alerts)
- [9. Testing Checklist](#9-testing-checklist)
- [10. Rollout Plan](#10-rollout-plan)
- [11. Troubleshooting & FAQ](#11-troubleshooting--faq)
- [12. Reference Links](#12-reference-links)

---

## 1. Architecture Overview
<img width="7523" height="520" alt="Start Decision Options Flow-2026-02-10-135235" src="https://github.com/user-attachments/assets/2de5b458-bbeb-41ca-8aa5-badca34e0b3f" />



### Diagram descriptions for Mermaid generation

You'll want to create the following Mermaid diagrams:

1. **Main data flow (flowchart LR):** Application API → Azure Queue Storage → Azure Function (timer, 10 min) → Data Lake Gen2 → [downstream boundary]. Label the edges with: "HTTP PUT (per log event)", "QueueTrigger / timer", "Blob write (batch JSON/NDJSON)".

2. **Sequence diagram — happy path:** Application sends log message to Queue. Queue stores message. Function triggers on timer. Function reads batch of messages (up to 32). Function writes NDJSON file to Data Lake. Function deletes processed messages from Queue. Repeat.

3. **Sequence diagram — failure path:** Application sends log to Queue. Function reads messages. Function fails to write to Data Lake. Messages become visible again in Queue after visibility timeout (10 min). Next Function invocation retries. After 5 failures, message goes to poison queue `api-logs-queue-poison`.

4. **Component diagram:** Show Azure Resource Group containing: Storage Account (with Queue + Data Lake Gen2 containers), Function App (Consumption plan), Application Insights, Key Vault (connection strings). Draw arrows between them.

5. **Data Lake folder structure (tree diagram):** Show the partition layout:
   ```
   logs/
   ├── 2026/
   │   ├── 02/
   │   │   ├── 10/
   │   │   │   ├── 08-00.ndjson
   │   │   │   ├── 08-10.ndjson
   │   │   │   ├── 08-20.ndjson
   │   │   │   └── ...
   │   │   ├── 11/
   │   │   └── ...
   │   └── 03/
   └── ...
   ```

---

<img width="5105" height="3975" alt="Start Decision Options Flow-2026-02-10-135610" src="https://github.com/user-attachments/assets/3f301055-351b-49ed-97e0-8c341193b4cb" />

## 2. Infrastructure Setup (Azure)

### 2.1 Resources to Create

| Resource | SKU / Tier | Name Convention | Notes |
|----------|-----------|-----------------|-------|
| Resource Group | — | `rg-logs-analytics-{env}` | e.g., `rg-logs-analytics-prod` |
| Storage Account | Standard LRS, General Purpose v2 | `stlogsanalytics{env}` | **Enable hierarchical namespace** (this makes it Data Lake Gen2) |
| Queue | — | `api-logs-queue` | Created inside the Storage Account |
| Blob Container | Hot tier | `logs` | For raw NDJSON files |
| Function App | Consumption plan (Linux) | `func-logs-collector-{env}` | Runtime: Python 3.11+ or Node.js 20+ |
| Application Insights | — | `appi-logs-collector-{env}` | Auto-attached to Function App |
| Key Vault | Standard | `kv-logs-{env}` | Store connection strings |

### 2.2 Azure CLI Setup Script

```bash
# Variables
RG="rg-logs-analytics-prod"
LOCATION="westeurope"  # same region as your MariaDB VM
STORAGE="stlogsanalyticsprod"
FUNC="func-logs-collector-prod"

# 1. Resource Group
az group create --name $RG --location $LOCATION


# 2. Storage Account (with Data Lake Gen2 enabled)
az storage account create \
  --name $STORAGE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true \
  --access-tier Hot

# 3. Queue
az storage queue create \
  --name api-logs-queue \
  --account-name $STORAGE

# 4. Blob container for raw logs
az storage fs create \
  --name logs \
  --account-name $STORAGE

# 5. Lifecycle policy — auto-delete files older than 30 days
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "delete-old-logs",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "delete": { "daysAfterModificationGreaterThan": 30 }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        }
      }
    }
  ]
}
EOF

az storage account management-policy create \
  --account-name $STORAGE \
  --resource-group $RG \
  --policy @lifecycle-policy.json

# 6. Function App
az functionapp create \
  --resource-group $RG \
  --consumption-plan-location $LOCATION \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --name $FUNC \
  --storage-account $STORAGE \
  --os-type Linux

# 7. Get connection string for app configuration
az storage account show-connection-string \
  --name $STORAGE \
  --resource-group $RG \
  --output tsv
```

### 2.3 Required IAM Roles

| Principal | Role | Scope | Purpose |
|-----------|------|-------|---------|
| Application (API backend) | `Storage Queue Data Message Sender` | Queue | Send log messages |
| Function App (managed identity) | `Storage Queue Data Message Processor` | Queue | Read + delete messages |
| Function App (managed identity) | `Storage Blob Data Contributor` | Container `logs` | Write NDJSON files |
| Data Factory (downstream) | `Storage Blob Data Reader` | Container `logs` | Read files for ETL |

> **Security note:** Use Managed Identity for the Function App — never embed connection strings in code. For the application, use either connection string (simpler) or Managed Identity (recommended for production).

---

## 3. Component A: Application → Queue (Producer)

### 3.1 What the Application Must Do

Replace every `INSERT INTO mysql_logs_table ...` with a single HTTP call (or SDK call) to Azure Queue Storage.

**The rule is simple:** one API request = one queue message.

### 3.2 Message Format

Every log message sent to the queue must be a **Base64-encoded JSON string**. This is required because Azure Queue Storage messages are XML-wrapped, and raw JSON with special characters can break parsing.

```json
{
  "timestamp": "2026-02-10T14:32:05.123Z",
  "requestId": "req_abc123def456",
  "method": "GET",
  "path": "/api/v2/destinations/search",
  "statusCode": 200,
  "durationMs": 142,
  "sourceId": "usr_789xyz",
  "clientIp": "203.0.113.42",
  "userAgent": "MyApp/2.1.0 (iOS 18.2)",
  "requestBody": null,
  "responseBodySize": 4521,
  "queryParams": {
    "lat": "52.2297",
    "lng": "21.0122",
    "radius": "50km"
  },
  "metadata": {
    "region": "eu-west",
    "apiVersion": "v2",
    "authMethod": "bearer_token"
  }
}
```

### 3.3 Queue Message Constraints

| Constraint | Value | How to Handle |
|-----------|-------|---------------|
| Max message size | 64 KB (48 KB after Base64 encoding) | Your avg log is ~3 KB — well within limits. If a log exceeds 48 KB (e.g., huge request body), truncate `requestBody` or `responseBody` fields. |
| Max TTL | 7 days (default) | Set to 7 days. If a message isn't processed in 7 days, something is very wrong. |
| Encoding | Base64 recommended | Use SDK — it handles encoding automatically. |

### 3.4 Code Examples

#### Python

```python
import json
import base64
from datetime import datetime
from azure.storage.queue import QueueClient

# Initialize once at app startup
queue_client = QueueClient.from_connection_string(
    conn_str="YOUR_CONNECTION_STRING",    # or use DefaultAzureCredential
    queue_name="api-logs-queue",
    message_encode_policy=None  # we'll encode manually for control
)

def send_log(log_data: dict) -> bool:
    """
    Send a single API log event to Azure Queue.
    Call this after each API request is processed.
    Returns True on success, False on failure.
    """
    try:
        # Ensure timestamp is set
        if "timestamp" not in log_data:
            log_data["timestamp"] = datetime.utcnow().isoformat() + "Z"

        # Serialize and Base64-encode
        message_json = json.dumps(log_data, separators=(",", ":"))  # compact

        # Safety check: truncate if too large (48KB limit after Base64)
        if len(message_json.encode("utf-8")) > 45000:
            log_data.pop("requestBody", None)
            log_data.pop("responseBody", None)
            log_data["_truncated"] = True
            message_json = json.dumps(log_data, separators=(",", ":"))

        encoded = base64.b64encode(message_json.encode("utf-8")).decode("utf-8")

        queue_client.send_message(encoded, time_to_live=604800)  # 7 days TTL
        return True

    except Exception as e:
        # LOG THE ERROR but do NOT crash the API request
        # The log is lost — this is acceptable (see section 7)
        print(f"[WARN] Failed to send log to queue: {e}")
        return False
```

#### Node.js / TypeScript

```typescript
import { QueueClient } from "@azure/storage-queue";

// Initialize once at app startup
const queueClient = new QueueClient(
  process.env.AZURE_STORAGE_CONNECTION_STRING!,
  "api-logs-queue"
);

interface ApiLogEvent {
  timestamp: string;
  requestId: string;
  method: string;
  path: string;
  statusCode: number;
  durationMs: number;
  sourceId: string;
  clientIp: string;
  userAgent: string;
  queryParams?: Record<string, string>;
  metadata?: Record<string, string>;
  requestBody?: unknown;
  responseBodySize?: number;
}

export async function sendLog(logData: ApiLogEvent): Promise<boolean> {
  try {
    if (!logData.timestamp) {
      logData.timestamp = new Date().toISOString();
    }

    let messageJson = JSON.stringify(logData);

    // Safety: truncate if too large
    if (Buffer.byteLength(messageJson, "utf-8") > 45000) {
      delete (logData as any).requestBody;
      delete (logData as any).responseBody;
      (logData as any)._truncated = true;
      messageJson = JSON.stringify(logData);
    }

    const encoded = Buffer.from(messageJson, "utf-8").toString("base64");

    await queueClient.sendMessage(encoded, {
      messageTimeToLive: 604800, // 7 days
    });

    return true;
  } catch (err) {
    // LOG but do NOT crash the API request
    console.warn("[WARN] Failed to send log to queue:", err);
    return false;
  }
}
```

#### Java

```java
import com.azure.storage.queue.QueueClient;
import com.azure.storage.queue.QueueClientBuilder;
import java.util.Base64;
import com.fasterxml.jackson.databind.ObjectMapper;

public class LogQueueSender {
    private final QueueClient queueClient;
    private final ObjectMapper mapper = new ObjectMapper();

    public LogQueueSender(String connectionString) {
        this.queueClient = new QueueClientBuilder()
            .connectionString(connectionString)
            .queueName("api-logs-queue")
            .buildClient();
    }

    public boolean sendLog(Map<String, Object> logData) {
        try {
            if (!logData.containsKey("timestamp")) {
                logData.put("timestamp", Instant.now().toString());
            }

            String json = mapper.writeValueAsString(logData);

            // Truncate if too large
            if (json.getBytes("UTF-8").length > 45000) {
                logData.remove("requestBody");
                logData.remove("responseBody");
                logData.put("_truncated", true);
                json = mapper.writeValueAsString(logData);
            }

            String encoded = Base64.getEncoder().encodeToString(json.getBytes("UTF-8"));
            queueClient.sendMessage(encoded);
            return true;
        } catch (Exception e) {
            System.err.println("[WARN] Failed to send log to queue: " + e.getMessage());
            return false;
        }
    }
}
```

#### C# / .NET

```csharp
using Azure.Storage.Queues;
using System.Text;
using System.Text.Json;

public class LogQueueSender
{
    private readonly QueueClient _queueClient;

    public LogQueueSender(string connectionString)
    {
        _queueClient = new QueueClient(
            connectionString,
            "api-logs-queue",
            new QueueClientOptions { MessageEncoding = QueueMessageEncoding.Base64 }
        );
    }

    public async Task<bool> SendLogAsync(Dictionary<string, object> logData)
    {
        try
        {
            if (!logData.ContainsKey("timestamp"))
                logData["timestamp"] = DateTime.UtcNow.ToString("o");

            var json = JsonSerializer.Serialize(logData);

            // When using Base64 encoding in QueueClientOptions,
            // the SDK handles encoding automatically
            await _queueClient.SendMessageAsync(
                json,
                timeToLive: TimeSpan.FromDays(7)
            );
            return true;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[WARN] Failed to send log to queue: {ex.Message}");
            return false;
        }
    }
}
```

### 3.5 Critical Rules for the Producer

1. **Never let a queue failure crash the API request.** Wrap in try/catch. A lost log is acceptable; a crashed user request is not.
2. **Send asynchronously (fire-and-forget).** Don't make the user wait for the queue write. Use async/non-blocking calls.
3. **One message per log event.** Don't batch multiple logs into one message — it complicates error handling.
4. **Always include `timestamp` in UTC ISO 8601** (`2026-02-10T14:32:05.123Z`).
5. **Always include `requestId`** — this is your correlation key for debugging.
6. **Truncate, don't fail.** If a message is too large, drop large fields (`requestBody`, `responseBody`) and add `_truncated: true`.

---

## 4. Component B: Queue → Data Lake (Collector Function)

### 4.1 What the Function Does

An Azure Function on a **timer trigger** (every 10 minutes):

1. Reads up to 32 messages from the queue (max per batch)
2. Loops until the queue is empty (or a max iteration limit is hit)
3. Decodes each message (Base64 → JSON)
4. Writes all messages as a single **NDJSON file** to Data Lake Gen2
5. Deletes processed messages from the queue

> **Why timer trigger instead of QueueTrigger?** Both work. QueueTrigger is simpler (auto-triggers per message) but creates one file per message, which leads to millions of tiny files — terrible for downstream Synapse queries. The timer approach batches messages into fewer, larger files.

### 4.2 Function Code (Python)

```python
# function_app.py
import azure.functions as func
import json
import base64
import logging
from datetime import datetime, timezone
from azure.storage.queue import QueueClient
from azure.storage.blob import BlobServiceClient

app = func.FunctionApp()

STORAGE_CONN_STR = "YOUR_CONNECTION_STRING"  # Use app settings in production
QUEUE_NAME = "api-logs-queue"
CONTAINER_NAME = "logs"
MAX_MESSAGES_PER_BATCH = 32
MAX_ITERATIONS = 50  # safety limit: 50 * 32 = 1,600 messages max per run

@app.timer_trigger(schedule="0 */10 * * * *", arg_name="timer")  # every 10 min
def collect_logs(timer: func.TimerRequest) -> None:
    queue_client = QueueClient.from_connection_string(STORAGE_CONN_STR, QUEUE_NAME)
    blob_service = BlobServiceClient.from_connection_string(STORAGE_CONN_STR)
    container_client = blob_service.get_container_client(CONTAINER_NAME)

    all_logs = []
    processed_messages = []
    iteration = 0

    while iteration < MAX_ITERATIONS:
        messages = queue_client.receive_messages(
            max_messages=MAX_MESSAGES_PER_BATCH,
            visibility_timeout=300  # 5 minutes to process
        )
        messages_list = list(messages)

        if not messages_list:
            break  # queue is empty

        for msg in messages_list:
            try:
                # Decode Base64 → JSON
                decoded = base64.b64decode(msg.content).decode("utf-8")
                log_entry = json.loads(decoded)
                all_logs.append(log_entry)
                processed_messages.append(msg)
            except Exception as e:
                logging.error(f"Failed to decode message {msg.id}: {e}")
                # Message will become visible again after timeout
                # After 5 failures, it goes to poison queue

        iteration += 1

    if not all_logs:
        logging.info("No messages in queue. Nothing to write.")
        return

    # Build NDJSON content (one JSON object per line)
    ndjson_content = "\n".join(json.dumps(log, separators=(",", ":")) for log in all_logs)

    # Generate blob path: logs/YYYY/MM/DD/HH-mm-ss.ndjson
    now = datetime.now(timezone.utc)
    blob_path = now.strftime("logs/%Y/%m/%d/%H-%M-%S.ndjson")

    # Upload to Data Lake
    blob_client = container_client.get_blob_client(blob_path)
    blob_client.upload_blob(
        ndjson_content.encode("utf-8"),
        overwrite=True,
        content_settings={"content_type": "application/x-ndjson"}
    )

    logging.info(f"Wrote {len(all_logs)} logs to {blob_path}")

    # Delete processed messages
    for msg in processed_messages:
        try:
            queue_client.delete_message(msg)
        except Exception as e:
            logging.warning(f"Failed to delete message {msg.id}: {e}")

    logging.info(f"Processed {len(all_logs)} messages in {iteration} iterations.")
```

### 4.3 Function Code (Node.js / TypeScript)

```typescript
// src/functions/collectLogs.ts
import { app, Timer } from "@azure/functions";
import { QueueClient } from "@azure/storage-queue";
import { BlobServiceClient } from "@azure/storage-blob";

const CONN_STR = process.env.AzureWebJobsStorage!;
const QUEUE_NAME = "api-logs-queue";
const CONTAINER_NAME = "logs";
const MAX_MESSAGES_PER_BATCH = 32;
const MAX_ITERATIONS = 50;

app.timer("collectLogs", {
  schedule: "0 */10 * * * *", // every 10 min
  handler: async (timer: Timer) => {
    const queueClient = new QueueClient(CONN_STR, QUEUE_NAME);
    const blobService = BlobServiceClient.fromConnectionString(CONN_STR);
    const containerClient = blobService.getContainerClient(CONTAINER_NAME);

    const allLogs: object[] = [];
    const toDelete: { messageId: string; popReceipt: string }[] = [];
    let iteration = 0;

    while (iteration < MAX_ITERATIONS) {
      const response = await queueClient.receiveMessages({
        numberOfMessages: MAX_MESSAGES_PER_BATCH,
        visibilityTimeout: 300,
      });

      if (!response.receivedMessageItems.length) break;

      for (const msg of response.receivedMessageItems) {
        try {
          const decoded = Buffer.from(msg.messageText, "base64").toString("utf-8");
          allLogs.push(JSON.parse(decoded));
          toDelete.push({ messageId: msg.messageId, popReceipt: msg.popReceipt });
        } catch (err) {
          console.error(`Failed to decode message ${msg.messageId}:`, err);
        }
      }
      iteration++;
    }

    if (!allLogs.length) {
      console.log("No messages in queue.");
      return;
    }

    // Write NDJSON
    const ndjson = allLogs.map((l) => JSON.stringify(l)).join("\n");
    const now = new Date();
    const pad = (n: number) => String(n).padStart(2, "0");
    const blobPath = `logs/${now.getUTCFullYear()}/${pad(now.getUTCMonth() + 1)}/${pad(now.getUTCDate())}/${pad(now.getUTCHours())}-${pad(now.getUTCMinutes())}-${pad(now.getUTCSeconds())}.ndjson`;

    const blobClient = containerClient.getBlockBlobClient(blobPath);
    await blobClient.upload(Buffer.from(ndjson, "utf-8"), Buffer.byteLength(ndjson, "utf-8"), {
      blobHTTPHeaders: { blobContentType: "application/x-ndjson" },
    });

    console.log(`Wrote ${allLogs.length} logs to ${blobPath}`);

    // Delete processed messages
    for (const msg of toDelete) {
      try {
        await queueClient.deleteMessage(msg.messageId, msg.popReceipt);
      } catch (err) {
        console.warn(`Failed to delete ${msg.messageId}:`, err);
      }
    }

    console.log(`Processed ${allLogs.length} messages in ${iteration} iterations.`);
  },
});
```

### 4.4 Function Configuration

**`host.json`:**
```json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "maxTelemetryItemsPerSecond": 5
      }
    }
  },
  "functionTimeout": "00:10:00"
}
```

**App Settings (environment variables):**

| Setting | Value | Notes |
|---------|-------|-------|
| `AzureWebJobsStorage` | Connection string to Storage Account | Required by Functions runtime |
| `FUNCTIONS_WORKER_RUNTIME` | `python` or `node` | Depends on your choice |

---

## 5. Data Lake Structure & File Format

### 5.1 Folder Partitioning

```
logs/
└── {YYYY}/
    └── {MM}/
        └── {DD}/
            ├── 00-00-00.ndjson
            ├── 00-10-00.ndjson
            ├── 00-20-00.ndjson
            ├── ...
            └── 23-50-00.ndjson
```

This gives **~144 files per day** (one every 10 minutes). Each file contains all messages collected in that 10-minute window.

At ~1M logs/day: each file contains ~6,900 log entries ≈ **~20 MB per file**. This is an ideal file size for Synapse Serverless (not too small, not too large).

### 5.2 File Format: NDJSON

**NDJSON** (Newline-Delimited JSON) = one JSON object per line, no outer array.

```
{"timestamp":"2026-02-10T14:32:05.123Z","requestId":"req_001","method":"GET","path":"/api/v2/search","statusCode":200,"durationMs":142,"sourceId":"usr_789"}
{"timestamp":"2026-02-10T14:32:05.456Z","requestId":"req_002","method":"POST","path":"/api/v2/bookings","statusCode":201,"durationMs":523,"sourceId":"usr_456"}
{"timestamp":"2026-02-10T14:32:06.789Z","requestId":"req_003","method":"GET","path":"/api/v2/destinations","statusCode":200,"durationMs":89,"sourceId":"usr_123"}
```

**Why NDJSON and not regular JSON array or CSV?**

| Format | Synapse Serverless | Streamable | Schema Flex | Our Choice |
|--------|-------------------|------------|-------------|------------|
| NDJSON | ✅ Native `OPENROWSET` | ✅ Append-friendly | ✅ | ✅ |
| JSON Array | ✅ But slower | ❌ Must buffer all | ✅ | ❌ |
| CSV | ✅ Fast | ✅ | ❌ Nested fields lost | ❌ |
| Parquet | ✅ Best perf | ❌ Complex to write | ⚠️ Schema locked | ❌ for raw* |

> *Parquet is used downstream — Data Factory converts NDJSON → Parquet for the analytics table. Raw logs stay NDJSON for flexibility.

### 5.3 Why This Structure Matters for Downstream

Synapse Serverless will query these files with partition elimination:

```sql
SELECT *
FROM OPENROWSET(
    BULK 'logs/2026/02/10/*.ndjson',
    DATA_SOURCE = 'datalake',
    FORMAT = 'CSV',
    FIELDTERMINATOR = '0x0b',
    FIELDQUOTE = '0x0b'
) WITH (doc NVARCHAR(MAX)) AS logs
```

The `YYYY/MM/DD` folder structure allows Synapse to scan only the days it needs — this keeps query costs minimal.

---

## 6. Log Message Schema

### 6.1 Required Fields

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `timestamp` | string (ISO 8601 UTC) | `"2026-02-10T14:32:05.123Z"` | **Required.** Always UTC. Include milliseconds. |
| `requestId` | string | `"req_abc123def456"` | **Required.** Unique per request. Correlation key. |
| `method` | string | `"GET"` | **Required.** HTTP method. |
| `path` | string | `"/api/v2/destinations/search"` | **Required.** API endpoint path. |
| `statusCode` | integer | `200` | **Required.** HTTP response code. |
| `durationMs` | integer | `142` | **Required.** Request processing time in ms. |
| `sourceId` | string | `"usr_789xyz"` | **Required.** User/client identifier. Used for clustering. |

### 6.2 Recommended Fields

| Field | Type | Example | Notes |
|-------|------|---------|-------|
| `clientIp` | string | `"203.0.113.42"` | Client IP address |
| `userAgent` | string | `"MyApp/2.1.0 (iOS 18.2)"` | Client user-agent string |
| `responseBodySize` | integer | `4521` | Response size in bytes |
| `queryParams` | object | `{"lat":"52.23","lng":"21.01"}` | URL query parameters (key-value) |
| `metadata` | object | `{"region":"eu-west"}` | Any additional context |

### 6.3 Optional Fields

| Field | Type | Notes |
|-------|------|-------|
| `requestBody` | object/string | Include only if needed for analytics. **Truncate if >10 KB.** |
| `responseBody` | object/string | Generally omit. Include only for error debugging. |
| `errorMessage` | string | Include when `statusCode >= 400` |
| `_truncated` | boolean | Set to `true` if fields were dropped due to size |

### 6.4 Schema Validation

**Do not enforce strict schema at the queue level.** NDJSON is schema-flexible — the downstream ETL (Data Factory) will handle missing/extra fields. This allows you to evolve the schema without breaking the pipeline.

However, validate at the application level that required fields are present before sending.

---

## 7. Error Handling & Resilience

<img width="8192" height="5344" alt="Start Decision Options Flow-2026-02-10-140022" src="https://github.com/user-attachments/assets/1ae7efa0-9b31-426c-9279-070857c5ae9a" />




### 7.1 Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Queue write fails (network) | Log event lost | Try/catch, log warning. Accept occasional loss. Consider local file fallback (see 7.2). |
| Queue Storage outage | All logs lost during outage | Azure Queue SLA: 99.9%. Outages are rare and short. |
| Function fails to process | Messages stay in queue | Built-in retry: messages become visible again after `visibilityTimeout`. |
| Function crashes repeatedly | Messages stuck | After 5 dequeue attempts, messages move to `api-logs-queue-poison`. |
| Data Lake write fails | Batch lost | Function retries next run. Messages are still in queue. |
| Malformed message | Single log lost | Skip and log error. Don't block the batch. |
| Message too old (>7d TTL) | Log expired | Indicates Function hasn't run in 7 days. Alerts should catch this first. |

### 7.2 Optional: Local File Fallback

For extra resilience, implement a local fallback when the queue is unreachable:

```python
import os
from datetime import datetime

FALLBACK_DIR = "/var/log/api-logs-fallback/"

def send_log_with_fallback(log_data: dict) -> None:
    success = send_log(log_data)  # try queue first
    if not success:
        # Write to local file as fallback
        os.makedirs(FALLBACK_DIR, exist_ok=True)
        filename = datetime.utcnow().strftime("%Y-%m-%d.ndjson")
        filepath = os.path.join(FALLBACK_DIR, filename)
        with open(filepath, "a") as f:
            f.write(json.dumps(log_data) + "\n")
```

Then set up a cron job or secondary Function to periodically upload fallback files to the Data Lake.

### 7.3 Poison Queue

Azure Queue Storage automatically creates a poison queue named `api-logs-queue-poison`. Messages that fail to process after **5 attempts** are moved there.

**Action required:** Monitor the poison queue. If messages appear there, investigate:
- Is the message malformed? (bad JSON)
- Is the Data Lake unreachable? (network/permissions)
- Is the Function timing out? (too many messages)

---

## 8. Monitoring & Alerts

### 8.1 Application Insights Queries

The Function App automatically logs to Application Insights. Use these KQL queries:

**Messages processed per hour:**
```kql
traces
| where message contains "Processed"
| summarize count() by bin(timestamp, 1h)
| render timechart
```

**Errors in the last 24h:**
```kql
exceptions
| where timestamp > ago(24h)
| summarize count() by outerMessage
| order by count_ desc
```

**Queue depth (approximate):**
```kql
customMetrics
| where name == "QueueMessageCount"
| summarize avg(value) by bin(timestamp, 5m)
| render timechart
```

### 8.2 Alerts to Configure

| Alert | Condition | Severity | Action |
|-------|-----------|----------|--------|
| Function failure | Function execution failed > 3 times in 15 min | High | Email + Slack |
| Queue depth growing | Approximate message count > 10,000 | Medium | Email |
| Poison queue non-empty | Messages in `api-logs-queue-poison` > 0 | High | Email + Slack |
| No files written | No new blobs in `logs/` for 30 min | Medium | Email |
| Function not running | No successful executions for 20 min | High | Email + Slack |

### 8.3 Health Check Endpoint

Add a simple health check to verify the pipeline is working:

```python
@app.route(route="health")
def health_check(req: func.HttpRequest) -> func.HttpResponse:
    try:
        queue_client = QueueClient.from_connection_string(CONN_STR, QUEUE_NAME)
        props = queue_client.get_queue_properties()
        count = props.approximate_message_count

        return func.HttpResponse(
            json.dumps({
                "status": "healthy",
                "queue_depth": count,
                "timestamp": datetime.utcnow().isoformat()
            }),
            mimetype="application/json"
        )
    except Exception as e:
        return func.HttpResponse(
            json.dumps({"status": "unhealthy", "error": str(e)}),
            status_code=500,
            mimetype="application/json"
        )
```

---

## 9. Testing Checklist

### 9.1 Pre-Deployment

- [ ] Storage Account created with hierarchical namespace enabled (Data Lake Gen2)
- [ ] Queue `api-logs-queue` exists
- [ ] Container `logs` exists
- [ ] Lifecycle policy active (30-day auto-delete on `logs/` prefix)
- [ ] Function App deployed and running
- [ ] Application Insights connected
- [ ] IAM roles assigned correctly (see section 2.3)
- [ ] Connection strings stored in App Settings (not in code)

### 9.2 Integration Tests

- [ ] **Happy path:** Send 100 test messages → wait for Function trigger → verify NDJSON file in Data Lake with 100 lines
- [ ] **Empty queue:** Trigger Function manually → verify no error, no file created
- [ ] **Large message:** Send a message close to 48 KB → verify it's processed correctly
- [ ] **Malformed message:** Send a non-JSON string → verify it's skipped, rest of batch processes, error logged
- [ ] **Base64 encoding:** Verify messages with special characters (`"`, `<`, `>`, `&`, unicode) survive encode/decode
- [ ] **File path format:** Verify blob path matches `logs/YYYY/MM/DD/HH-MM-SS.ndjson`
- [ ] **NDJSON format:** Download a file, verify each line is valid JSON, no trailing comma, no array wrapper
- [ ] **Timestamp correctness:** All timestamps in UTC, ISO 8601 format

### 9.3 Load Test

- [ ] Send 10,000 messages in 1 minute → verify all processed within 2 Function cycles (20 min)
- [ ] Send 100,000 messages → verify all processed within ~1 hour
- [ ] Verify no message loss (count messages sent vs. lines in NDJSON files)

### 9.4 Failure Tests

- [ ] Kill Function during processing → verify messages reappear in queue
- [ ] Send 6 identical failing messages → verify they land in poison queue
- [ ] Temporarily block Data Lake access → verify messages stay in queue and are retried

---

## 10. Rollout Plan

### Phase 1: Parallel Write (1 week)

Application writes to **both** MySQL and Queue Storage simultaneously.

```python
# Pseudo-code for parallel period
def log_api_request(log_data):
    # Existing MySQL write (keep for now)
    mysql_insert(log_data)

    # New: Queue Storage write
    send_log(log_data)
```

**Validation during Phase 1:**
- Compare daily counts: MySQL rows vs. NDJSON lines in Data Lake
- Tolerance: < 0.1% difference is acceptable (queue failures are fire-and-forget)

### Phase 2: Cut Over

- Stop MySQL writes
- Verify Queue-only path works for 48 hours
- Drop MySQL log table and connection

### Phase 3: Decommission MySQL

- Remove MySQL logging code from application
- Shut down MySQL instance (if not used for anything else)

---

## 11. Troubleshooting & FAQ

### Common Issues

**Q: Messages are piling up in the queue and not being processed.**
Check: Is the Function App running? Go to Azure Portal → Function App → Functions → `collectLogs` → Monitor. Look for recent invocations. If there are none, the Function may have stopped. Restart the Function App.

**Q: I see "AuthenticationFailed" errors when sending to the queue.**
Check: Connection string is correct. If using Managed Identity, ensure the identity has `Storage Queue Data Message Sender` role. Check that the Storage Account firewall allows traffic from your application's IP/VNet.

**Q: NDJSON files are empty or have very few records.**
Check: Is the application actually sending messages? Send a test message manually using Azure Storage Explorer. Then check the queue depth. If messages are there but the Function isn't processing them, check Function logs.

**Q: Some log entries have garbled/broken characters.**
Check: Ensure you're Base64-encoding the JSON string as UTF-8. If using the SDK with `MessageEncoding.Base64`, it handles this automatically. If using the REST API directly, you must encode manually.

**Q: The Function times out.**
Check: Default timeout on Consumption plan is 5 minutes. Our `host.json` sets it to 10 minutes. If queue depth is very large (>50,000 messages), consider increasing `MAX_ITERATIONS` or switching to a Premium plan.

**Q: Messages end up in the poison queue.**
Check: Read messages from `api-logs-queue-poison` using Azure Storage Explorer. Decode them (Base64 → JSON) and inspect. Common causes: malformed JSON, non-Base64 content, extremely large messages.

**Q: How do I replay messages from the poison queue?**
Use Azure Storage Explorer or a script to read messages from `api-logs-queue-poison`, then re-send them to `api-logs-queue`. Fix the root cause first.

**Q: What happens during a Storage Account outage?**
Azure Queue Storage has 99.9% SLA. During the rare outage, log messages will be lost (fire-and-forget pattern). If this is unacceptable, implement the local file fallback (section 7.2).

**Q: Can I change the log schema without breaking anything?**
Yes. NDJSON is schema-flexible. Adding new fields is safe — downstream ETL will simply ignore unknown fields until updated. Removing required fields will break downstream — coordinate with the Synapse team first.

**Q: How do I verify the Data Lake files are correct?**
Use Azure Storage Explorer to navigate to the `logs` container. Open an NDJSON file. Each line should be a valid JSON object. You can also use `az storage blob download` to fetch a file and inspect locally.

---

## 12. Reference Links

### Azure Queue Storage
- [Queue Storage overview](https://learn.microsoft.com/en-us/azure/storage/queues/storage-queues-introduction)
- [Queue REST API reference](https://learn.microsoft.com/en-us/rest/api/storageservices/queue-service-rest-api)
- [Put Message API](https://learn.microsoft.com/en-us/rest/api/storageservices/put-message)
- [Python SDK quickstart](https://learn.microsoft.com/en-us/azure/storage/queues/storage-quickstart-queues-python)
- [Node.js SDK](https://www.npmjs.com/package/@azure/storage-queue)
- [.NET SDK (NuGet)](https://www.nuget.org/packages/Azure.Storage.Queues)
- [Java SDK](https://learn.microsoft.com/en-us/azure/storage/queues/storage-quickstart-queues-java)
- [Queue vs Service Bus comparison](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted)

### Azure Data Lake Gen2
- [Data Lake Gen2 overview](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
- [Blob lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [NDJSON format spec](https://github.com/ndjson/ndjson-spec)

### Azure Functions
- [Timer trigger reference](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer)
- [Queue trigger reference](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-queue-trigger)
- [Python developer guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python)
- [Node.js developer guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-node)
- [Host.json reference](https://learn.microsoft.com/en-us/azure/azure-functions/functions-host-json)

### Monitoring
- [Application Insights for Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-monitoring)
- [KQL query language](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)

### Pricing
- [Queue Storage pricing](https://azure.microsoft.com/en-us/pricing/details/storage/queues/)
- [Functions pricing (Consumption)](https://azure.microsoft.com/en-us/pricing/details/functions/)
- [Data Lake Gen2 pricing](https://azure.microsoft.com/en-us/pricing/details/storage/data-lake/)

---

## Appendix: Quick Reference Card

```
QUEUE NAME:          api-logs-queue
POISON QUEUE:        api-logs-queue-poison
CONTAINER:           logs
FILE FORMAT:         NDJSON (one JSON per line)
FILE PATH PATTERN:   logs/{YYYY}/{MM}/{DD}/{HH}-{mm}-{ss}.ndjson
FUNCTION SCHEDULE:   Every 10 minutes (0 */10 * * * *)
MESSAGE ENCODING:    Base64
MAX MESSAGE SIZE:    48 KB (after Base64)
MESSAGE TTL:         7 days
RAW LOG RETENTION:   30 days (auto-delete via lifecycle policy)
BATCH SIZE:          Up to 32 messages per receive call
MAX PER RUN:         ~1,600 messages (50 iterations × 32)
```
