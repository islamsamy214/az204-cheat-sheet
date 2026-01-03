# Event Hubs Capture

## What is Event Hubs Capture?

**Event Hubs Capture** is an integrated feature that automatically captures streaming data from Event Hubs and delivers it to **Azure Blob Storage** or **Azure Data Lake Storage** in **Apache Avro** format.

### Key Benefits

- **Automatic**: No code required, enable via Azure Portal or CLI
- **Scalable**: Handles millions of events per second
- **Cost-Effective**: Uses Event Hubs' internal storage (bypasses TU egress quota)
- **Time-Based**: Capture based on time window or data size
- **Durable**: Long-term storage for batch processing and compliance
- **Flexible**: Process captured files with any Avro-compatible tool

### Use Cases

| Use Case | Description | Example |
|----------|-------------|---------|
| **Data Lake** | Build data lake for analytics | Store IoT telemetry for ML training |
| **Compliance** | Long-term retention for regulatory requirements | Financial transaction logs (7 years) |
| **Batch Processing** | Periodic batch analytics | Daily aggregation with Spark/Databricks |
| **Backup** | Archive streaming data | Disaster recovery, replay scenarios |
| **Cold Storage** | Cost-effective long-term storage | Archive old events to Cool/Archive tier |
| **Hybrid Processing** | Combine real-time + batch (Lambda architecture) | Real-time alerts + daily reports |

---

## How Event Hubs Capture Works

### Capture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      EVENT PRODUCERS                         │
│   IoT Devices, Applications, Services, Kafka Clients        │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────────┐
        │      EVENT HUBS NAMESPACE          │
        │  ┌──────────────────────────────┐  │
        │  │   Event Hub: "telemetry"     │  │
        │  │  ┌────┬────┬────┬────┐      │  │
        │  │  │ P0 │ P1 │ P2 │ P3 │      │  │
        │  │  └────┴────┴────┴────┘      │  │
        │  │                              │  │
        │  │  Capture Enabled:            │  │
        │  │  • Time: 5 minutes           │  │
        │  │  • Size: 100 MB              │  │
        │  └──────────────────────────────┘  │
        └──────────────┬─────────────────────┘
                       │
                       │ (Automatic Capture)
                       │
                       ▼
        ┌────────────────────────────────────┐
        │  AZURE BLOB STORAGE / DATA LAKE    │
        │                                     │
        │  Container: mycaptures              │
        │  ├── telemetry/                     │
        │  │   ├── 0/                         │
        │  │   │   ├── 2024/01/15/10/        │
        │  │   │   │   ├── 00.avro           │
        │  │   │   │   ├── 05.avro           │
        │  │   │   │   └── 10.avro           │
        │  │   ├── 1/                         │
        │  │   │   └── 2024/01/15/10/        │
        │  │   ├── 2/                         │
        │  │   └── 3/                         │
        └────────────────────────────────────┘
                       │
                       ▼
        ┌────────────────────────────────────┐
        │     BATCH PROCESSING               │
        │  • Azure Databricks                │
        │  • Azure Synapse Analytics         │
        │  • Azure Data Factory              │
        │  • HDInsight (Spark, Hive)         │
        └────────────────────────────────────┘
```

### Capture Process Flow

1. **Events Arrive**: Producers send events to Event Hubs
2. **Internal Storage**: Events stored in Event Hubs internal time-retention store
3. **Capture Trigger**: Time window OR size threshold reached (first wins)
4. **File Creation**: Events written to Avro file
5. **Upload**: File uploaded to Blob Storage or Data Lake
6. **Empty Files**: Empty file created if no events during time window

**Important Notes:**
- ⚠️ Capture does NOT consume throughput units (egress quota)
- ✅ Capture operates directly from internal storage
- ✅ No impact on real-time consumers
- ✅ Capture happens per partition independently

---

## Apache Avro Format

**Apache Avro** is a compact, fast, binary data serialization format with inline schema.

### Why Avro?

| Feature | Benefit |
|---------|---------|
| **Compact** | Binary format (smaller than JSON/XML) |
| **Fast** | Efficient serialization/deserialization |
| **Schema Evolution** | Add/remove fields without breaking compatibility |
| **Self-Describing** | Schema embedded in file |
| **Cross-Language** | Supported by many languages (C#, Java, Python, etc.) |
| **Splittable** | Works well with MapReduce/Spark |

### Avro File Structure

```
┌──────────────────────────────────┐
│         Avro File Header         │
│  • Magic: "Obj1"                 │
│  • Schema (JSON)                 │
│  • Sync marker (16 bytes)        │
├──────────────────────────────────┤
│         Data Block 1             │
│  • Count: Number of events       │
│  • Size: Block size in bytes     │
│  • Events (binary)               │
│  • Sync marker                   │
├──────────────────────────────────┤
│         Data Block 2             │
│  • Events...                     │
│  • Sync marker                   │
├──────────────────────────────────┤
│            ...                   │
└──────────────────────────────────┘
```

### Event Hubs Avro Schema

**Schema for captured events:**

```json
{
  "type": "record",
  "name": "EventData",
  "namespace": "Microsoft.ServiceBus.Messaging",
  "fields": [
    {
      "name": "SequenceNumber",
      "type": "long"
    },
    {
      "name": "Offset",
      "type": "string"
    },
    {
      "name": "EnqueuedTimeUtc",
      "type": "string"
    },
    {
      "name": "SystemProperties",
      "type": {
        "type": "map",
        "values": ["long", "double", "string", "bytes"]
      }
    },
    {
      "name": "Properties",
      "type": {
        "type": "map",
        "values": ["long", "double", "string", "bytes", "null"]
      }
    },
    {
      "name": "Body",
      "type": ["null", "bytes"]
    }
  ]
}
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| `SequenceNumber` | long | Unique sequence number within partition |
| `Offset` | string | Event offset in partition log |
| `EnqueuedTimeUtc` | string | UTC timestamp when event was enqueued |
| `SystemProperties` | map | System-assigned properties |
| `Properties` | map | User-defined application properties |
| `Body` | bytes | Event body (payload) |

---

## Capture Configuration

### Capture Windowing

**Capture triggers** based on **first wins policy**:
- **Time Window**: Capture after X minutes (1-15 minutes)
- **Size Window**: Capture after X MB (10-500 MB)

**Example Scenarios:**

```
Scenario 1: Time=5 min, Size=100 MB
- High traffic: File created every ~2 minutes (100 MB reached)
- Low traffic: File created every 5 minutes (time reached)

Scenario 2: Time=15 min, Size=10 MB
- Steady traffic: File created every ~1 minute (10 MB reached)
- Very low traffic: File created every 15 minutes

Scenario 3: No events
- Empty file created every time window (maintains predictable cadence)
```

### Naming Convention

**File naming pattern:**
```
{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}
```

**Example file paths:**

```
mystorage.blob.core.windows.net/captures/
├── mynamespace/
│   ├── telemetry/
│   │   ├── 0/
│   │   │   ├── 2024/
│   │   │   │   ├── 01/
│   │   │   │   │   ├── 15/
│   │   │   │   │   │   ├── 10/
│   │   │   │   │   │   │   ├── 00/
│   │   │   │   │   │   │   │   └── 17.avro  ← File created at 10:00:17
│   │   │   │   │   │   │   ├── 05/
│   │   │   │   │   │   │   │   └── 42.avro  ← File created at 10:05:42
│   │   ├── 1/
│   │   │   └── 2024/01/15/10/00/23.avro
│   │   ├── 2/
│   │   └── 3/
```

**Custom Naming (Optional):**
You can add a custom prefix:
```
{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}
→ mydata/{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}
```

### Configuration Parameters

| Parameter | Description | Range | Recommendation |
|-----------|-------------|-------|----------------|
| **Time Window** | Minutes between captures | 1-15 minutes | 5-10 min (balance latency & file count) |
| **Size Window** | MB before capture | 10-500 MB | 100-300 MB (optimize for processing) |
| **Skip Empty** | Create empty files? | true/false | false (maintain predictable cadence) |

---

## Enable Capture

### Azure Portal

1. Navigate to Event Hubs namespace
2. Select your Event Hub
3. Select **Capture** under Settings
4. Toggle **On**
5. Configure:
   - **Time window**: 5 minutes (example)
   - **Size window**: 100 MB (example)
   - **Capture provider**: Azure Storage Blob or Data Lake
   - **Storage account**: Select or create
   - **Container**: Specify container name
   - **Naming format**: Default or custom
6. Click **Save**

### Azure CLI

**Create Event Hub with Capture:**

```bash
# Variables
RG="rg-eventhubs"
NAMESPACE="myeventhubns"
EVENTHUB="myeventhub"
STORAGE_ACCOUNT="mystorageaccount"
CONTAINER="captures"

# Create storage account (if not exists)
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --location eastus \
  --sku Standard_LRS

# Get storage account resource ID
STORAGE_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --query id \
  --output tsv)

# Create container
az storage container create \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT

# Create Event Hub with Capture enabled
az eventhubs eventhub create \
  --name $EVENTHUB \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --partition-count 4 \
  --message-retention 7 \
  --enable-capture true \
  --capture-interval 300 \
  --capture-size-limit 104857600 \
  --destination-name EventHubArchive.AzureBlockBlob \
  --storage-account $STORAGE_ID \
  --blob-container $CONTAINER \
  --archive-name-format "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
```

**Enable Capture on Existing Event Hub:**

```bash
az eventhubs eventhub update \
  --name $EVENTHUB \
  --namespace-name $NAMESPACE \
  --resource-group $RG \
  --enable-capture true \
  --capture-interval 300 \
  --capture-size-limit 104857600 \
  --destination-name EventHubArchive.AzureBlockBlob \
  --storage-account $STORAGE_ID \
  --blob-container $CONTAINER
```

**Capture Parameters Explained:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| `--enable-capture` | true | Enable Capture feature |
| `--capture-interval` | 300 | Time window in seconds (5 minutes) |
| `--capture-size-limit` | 104857600 | Size window in bytes (100 MB) |
| `--destination-name` | EventHubArchive.AzureBlockBlob | Capture destination type |
| `--storage-account` | Resource ID | Target storage account |
| `--blob-container` | captures | Container name |
| `--archive-name-format` | Pattern | File naming pattern |

### ARM Template

```json
{
  "type": "Microsoft.EventHub/namespaces/eventhubs",
  "apiVersion": "2021-11-01",
  "name": "[concat(parameters('namespaceName'), '/', parameters('eventHubName'))]",
  "properties": {
    "messageRetentionInDays": 7,
    "partitionCount": 4,
    "captureDescription": {
      "enabled": true,
      "skipEmptyArchives": false,
      "encoding": "Avro",
      "intervalInSeconds": 300,
      "sizeLimitInBytes": 104857600,
      "destination": {
        "name": "EventHubArchive.AzureBlockBlob",
        "properties": {
          "storageAccountResourceId": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]",
          "blobContainer": "captures",
          "archiveNameFormat": "{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}"
        }
      }
    }
  }
}
```

---

## Reading Captured Avro Files

### Using Python (Avro Library)

```python
from avro.datafile import DataFileReader
from avro.io import DatumReader
from azure.storage.blob import BlobServiceClient
import io

# Download Avro file from Blob Storage
connection_string = "<storage-connection-string>"
container_name = "captures"
blob_name = "mynamespace/myeventhub/0/2024/01/15/10/00/17.avro"

blob_service_client = BlobServiceClient.from_connection_string(connection_string)
blob_client = blob_service_client.get_blob_client(container_name, blob_name)

# Download to memory
blob_data = blob_client.download_blob().readall()
avro_stream = io.BytesIO(blob_data)

# Read Avro file
reader = DataFileReader(avro_stream, DatumReader())

for event in reader:
    sequence_number = event['SequenceNumber']
    offset = event['Offset']
    enqueued_time = event['EnqueuedTimeUtc']
    body = event['Body']
    
    # Decode body (UTF-8 string)
    body_str = body.decode('utf-8')
    
    print(f"Sequence: {sequence_number}")
    print(f"Offset: {offset}")
    print(f"Time: {enqueued_time}")
    print(f"Body: {body_str}")
    print("---")

reader.close()
```

### Using C# (Avro Library)

```csharp
using Avro.File;
using Avro.Generic;
using Azure.Storage.Blobs;

string connectionString = "<storage-connection-string>";
string containerName = "captures";
string blobName = "mynamespace/myeventhub/0/2024/01/15/10/00/17.avro";

// Download Avro file
var blobClient = new BlobClient(connectionString, containerName, blobName);
using var stream = new MemoryStream();
await blobClient.DownloadToAsync(stream);
stream.Position = 0;

// Read Avro file
using var reader = DataFileReader<GenericRecord>.OpenReader(stream);

foreach (var record in reader.NextEntries)
{
    long sequenceNumber = (long)record["SequenceNumber"];
    string offset = (string)record["Offset"];
    string enqueuedTime = (string)record["EnqueuedTimeUtc"];
    byte[] body = (byte[])record["Body"];
    
    // Decode body
    string bodyStr = Encoding.UTF8.GetString(body);
    
    Console.WriteLine($"Sequence: {sequenceNumber}");
    Console.WriteLine($"Offset: {offset}");
    Console.WriteLine($"Time: {enqueuedTime}");
    Console.WriteLine($"Body: {bodyStr}");
    Console.WriteLine("---");
}
```

### Using Azure Databricks (PySpark)

```python
# Mount storage account (one-time setup)
dbutils.fs.mount(
    source = "wasbs://captures@mystorageaccount.blob.core.windows.net",
    mount_point = "/mnt/captures",
    extra_configs = {
        "fs.azure.account.key.mystorageaccount.blob.core.windows.net": "<storage-key>"
    }
)

# Read Avro files with Spark
df = spark.read.format("avro").load("/mnt/captures/mynamespace/myeventhub/*/*/*/*/*/*/*/*.avro")

# Show schema
df.printSchema()

# Query events
df.select("SequenceNumber", "EnqueuedTimeUtc", "Body").show()

# Decode body and convert to JSON
from pyspark.sql.functions import col, decode

df_decoded = df.withColumn("BodyString", decode(col("Body"), "UTF-8"))

# Parse JSON body
from pyspark.sql.functions import from_json, schema_of_json

# Infer schema from sample
sample_json = df_decoded.select("BodyString").first()[0]
json_schema = schema_of_json(sample_json)

# Parse all events
df_parsed = df_decoded.withColumn("BodyJson", from_json(col("BodyString"), json_schema))

# Query specific fields
df_parsed.select("EnqueuedTimeUtc", "BodyJson.*").show()
```

### Using Azure Synapse Analytics

```sql
-- Create external data source
CREATE EXTERNAL DATA SOURCE CapturedEvents
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://captures@mystorageaccount.blob.core.windows.net',
    CREDENTIAL = AzureStorageCredential
);

-- Query Avro files
SELECT 
    SequenceNumber,
    Offset,
    EnqueuedTimeUtc,
    CAST(Body AS VARCHAR(MAX)) AS BodyText
FROM OPENROWSET(
    BULK '/mynamespace/myeventhub/*/*/*/*/*/*/*/*.avro',
    DATA_SOURCE = 'CapturedEvents',
    FORMAT = 'PARQUET'
) AS [Events];
```

---

## Throughput and Capacity

### Throughput Units and Capture

**Key Points:**
- ✅ **Capture does NOT consume egress TU quota**
- ✅ Operates directly from Event Hubs internal storage
- ✅ No impact on real-time consumers
- ✅ No additional TU cost for Capture operation

**Throughput Unit Reminder:**
- 1 TU = 1 MB/s ingress, 2 MB/s egress
- Standard tier: 1-40 TUs
- Capture bypasses egress quota

**Example Calculation:**

```
Scenario: 10 MB/s ingress, real-time consumers reading 5 MB/s

WITHOUT Capture:
- Ingress: 10 TUs (10 MB/s ÷ 1 MB/s per TU)
- Egress: 3 TUs (5 MB/s ÷ 2 MB/s per TU)
- Required: 10 TUs

WITH Capture (10 MB/s also captured):
- Ingress: 10 TUs (10 MB/s ÷ 1 MB/s per TU)
- Egress: 3 TUs (5 MB/s ÷ 2 MB/s per TU)
- Capture: 0 TUs (bypasses egress quota)
- Required: Still 10 TUs!

Capture is effectively FREE from TU perspective!
```

### Storage Costs

**Storage Tiers:**

| Tier | Cost (per GB/month) | Use Case |
|------|---------------------|----------|
| **Hot** | ~$0.02 | Recent data, frequent access |
| **Cool** | ~$0.01 | 30-90 day retention, infrequent access |
| **Archive** | ~$0.002 | Long-term compliance (>90 days) |

**Lifecycle Management:**
```bash
# Move to Cool after 30 days, Archive after 90 days
az storage account management-policy create \
  --account-name mystorageaccount \
  --resource-group myResourceGroup \
  --policy @lifecycle-policy.json
```

**lifecycle-policy.json:**
```json
{
  "rules": [
    {
      "enabled": true,
      "name": "MoveToArchive",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["captures/"]
        }
      }
    }
  ]
}
```

---

## Monitoring Capture

### Azure Portal Metrics

Navigate to: **Event Hub → Metrics**

**Key Metrics:**

| Metric | Description | Alert Threshold |
|--------|-------------|----------------|
| **Captured Bytes** | Total bytes captured | Monitor for gaps |
| **Captured Messages** | Total messages captured | Compare with ingress |
| **Capture Backlog** | Events waiting to be captured | > 0 (should be low) |
| **Capture Errors** | Failed capture attempts | > 0 |

### Azure CLI Monitoring

```bash
# View capture metrics
az monitor metrics list \
  --resource /subscriptions/{subscription}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{ns}/eventhubs/{eh} \
  --metric "CapturedMessages" \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --interval PT1H
```

### Diagnostic Logs

**Enable diagnostic logs:**

```bash
az monitor diagnostic-settings create \
  --name CaptureLogging \
  --resource /subscriptions/{subscription}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{ns}/eventhubs/{eh} \
  --logs '[{"category":"ArchiveLogs","enabled":true}]' \
  --workspace /subscriptions/{subscription}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{workspace}
```

**Query logs (KQL):**
```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.EVENTHUB"
| where Category == "ArchiveLogs"
| where OperationName == "Archive"
| project TimeGenerated, Status, Message, PartitionId
| order by TimeGenerated desc
```

---

## Best Practices

### Configuration

1. **Time Window**
   - Shorter (1-5 min): Lower latency, more files
   - Longer (10-15 min): Fewer files, higher latency
   - **Recommendation**: 5 minutes for most scenarios

2. **Size Window**
   - Smaller (10-50 MB): More files, faster processing startup
   - Larger (100-500 MB): Fewer files, better batch efficiency
   - **Recommendation**: 100-300 MB for Spark/Databricks

3. **Empty Files**
   - Keep enabled (skipEmptyArchives = false)
   - Provides predictable cadence
   - Easier monitoring and automation

4. **Partitions**
   - Each partition creates separate files
   - More partitions = more files
   - Balance partition count with processing parallelism

### Processing

1. **Incremental Processing**
   - Track last processed file timestamp
   - Process only new files
   - Use watermarking in Spark Structured Streaming

2. **Parallel Processing**
   - Process partitions in parallel
   - Use Spark/Databricks for scalability
   - One executor per partition for optimal performance

3. **Error Handling**
   - Handle corrupted/incomplete files
   - Implement retry logic
   - Log failed files for manual review

4. **Schema Evolution**
   - Use Avro schema evolution features
   - Handle backward/forward compatibility
   - Version your schemas

### Cost Optimization

1. **Storage Tier**
   - Use lifecycle management
   - Cool tier for 30-90 day retention
   - Archive tier for compliance (>90 days)

2. **Compression**
   - Avro files are already compressed
   - Consider gzip for additional compression
   - Trade-off: storage vs CPU

3. **Retention**
   - Delete old files if not needed
   - Balance between retention requirements and cost

4. **Naming Convention**
   - Use consistent naming for easier management
   - Include metadata in folder structure
   - Optimize for query patterns (partition by date)

---

## Troubleshooting

### Common Issues

**Issue 1: No files created**

**Symptoms:**
- Capture enabled, but no files in storage

**Possible Causes:**
- No events published to Event Hub
- Storage account connection issues
- Incorrect permissions

**Resolution:**
```bash
# Check Event Hub metrics
az monitor metrics list \
  --resource <event-hub-resource-id> \
  --metric "IncomingMessages"

# Verify storage account connection
az storage container show \
  --name captures \
  --account-name mystorageaccount

# Check Event Hub capture configuration
az eventhubs eventhub show \
  --name myeventhub \
  --namespace-name myeventhubns \
  --resource-group myResourceGroup \
  --query captureDescription
```

**Issue 2: Capture lag**

**Symptoms:**
- Events captured with significant delay

**Possible Causes:**
- Time window too long
- Size window too large
- Low throughput

**Resolution:**
- Reduce time window (e.g., 5 min → 2 min)
- Reduce size window (e.g., 500 MB → 100 MB)
- Monitor "CaptureBacklog" metric

**Issue 3: Cannot read Avro files**

**Symptoms:**
- Errors when reading Avro files

**Possible Causes:**
- Incorrect Avro library version
- File not fully written
- Corrupted file

**Resolution:**
```python
# Check file size (incomplete files may be 0 bytes)
from azure.storage.blob import BlobServiceClient

blob_client = BlobServiceClient.from_connection_string(conn_str)
blob = blob_client.get_blob_client("captures", blob_name)
properties = blob.get_blob_properties()
print(f"Size: {properties.size} bytes")

# Try reading with error handling
try:
    reader = DataFileReader(stream, DatumReader())
    for record in reader:
        print(record)
except Exception as e:
    print(f"Error reading file: {e}")
```

---

## Exam Tips for AZ-204

### Key Concepts to Remember

1. **Capture** = automatic capture to storage (no code required)
2. **Format** = Apache Avro (compact, fast, binary with schema)
3. **Windowing** = time OR size (first wins policy)
4. **No TU cost** = Capture bypasses egress quota
5. **Destinations** = Azure Blob Storage or Data Lake Storage
6. **Naming** = `{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}`
7. **Empty files** = Created when no events (predictable cadence)

### Common Exam Scenarios

**Scenario 1**: Need long-term storage for compliance
- ✅ Enable Event Hubs Capture
- ✅ Configure retention period
- ✅ Use lifecycle management (Cool/Archive tiers)

**Scenario 2**: Build data lake for analytics
- ✅ Capture to Data Lake Storage
- ✅ Process with Databricks/Synapse
- ✅ Query with Spark SQL

**Scenario 3**: Lambda architecture (real-time + batch)
- ✅ Real-time consumers for immediate processing
- ✅ Capture for batch analytics
- ✅ Both read same events independently

**Scenario 4**: Optimize capture costs
- ✅ Capture bypasses TU egress quota (no additional TU cost)
- ✅ Use storage lifecycle management
- ✅ Balance time/size window for file count

### Remember for Exam

- **Avro format**: Compact, fast, self-describing
- **Time window**: 1-15 minutes
- **Size window**: 10-500 MB
- **First wins**: Time OR size (whichever comes first)
- **No TU impact**: Capture uses internal storage
- **Per partition**: Each partition creates separate files
- **Empty files**: Maintain predictable cadence
- **Destinations**: Blob Storage or Data Lake

### Quick Command Reference

```bash
# Enable capture on new Event Hub
az eventhubs eventhub create \
  --name <eh> \
  --namespace-name <ns> \
  --resource-group <rg> \
  --enable-capture true \
  --capture-interval 300 \
  --capture-size-limit 104857600 \
  --storage-account <storage-id> \
  --blob-container <container>

# Enable capture on existing Event Hub
az eventhubs eventhub update \
  --name <eh> \
  --namespace-name <ns> \
  --resource-group <rg> \
  --enable-capture true

# Check capture status
az eventhubs eventhub show \
  --name <eh> \
  --namespace-name <ns> \
  --resource-group <rg> \
  --query captureDescription
```

---

## Summary

**Event Hubs Capture** automatically captures streaming data to storage for long-term retention and batch analytics.

**Key Features:**
- Automatic capture (no code)
- Apache Avro format
- Time OR size windowing (first wins)
- No TU egress cost
- Blob Storage or Data Lake destinations

**Configuration:**
- Time window: 1-15 minutes
- Size window: 10-500 MB
- Naming: Standard or custom format
- Empty files: Maintain predictable cadence

**Use Cases:**
- Data lake for batch analytics
- Compliance and archiving
- Lambda architecture (real-time + batch)
- Replay and reprocessing scenarios

**Benefits:**
- Cost-effective (bypasses TU egress quota)
- Scalable (millions events/second)
- Durable (long-term storage)
- Flexible (process with any Avro-compatible tool)