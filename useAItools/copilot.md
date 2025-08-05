Absolutely, Vishal! Here's a detailed yet easy-to-implement solution to reduce Azure Cosmos DB costs while meeting all your requirements. It leverages **Azure Functions**, **Azure Blob Storage**, and **Durable Functions** for a smooth archival mechanism.

---

## 🧠 Solution Overview

Store recent billing records (≤3 months) in **Cosmos DB**, and archive older records to **Azure Blob Storage (Hot tier)**. Use **Durable Functions** to orchestrate data movement in batches. Query routing is abstracted behind a **Facade Function**, maintaining API contract without downtime.

---

## 🏗️ Architecture Diagram

```
+------------------+
| Client/API Calls |
+------------------+
        |
        v
+------------------------+
| Azure Function (Facade)| <-- Reads/Writes to appropriate store
+------------------------+
   |               |
   v               v
Cosmos DB     Azure Blob Storage (Archived)
   |               ^
   v               |
+--------------------------+
| Durable Function (Archival Trigger) |
+--------------------------+
```

---

## 🚀 Step-by-Step Guide

### **Step 1: Create Azure Blob Storage**
```bash
az storage account create --name mystorageacct --resource-group myrg --location eastus --sku Standard_LRS
az storage container create --name billing-archive --account-name mystorageacct
```

### **Step 2: Set Up Durable Function for Archival**
Trigger: Timer (daily)  
Logic:
```python
# orchestrator.py
def orchestrator_function(context):
    cutoff_date = datetime.utcnow() - timedelta(days=90)
    query = f"SELECT * FROM c WHERE c.timestamp < '{cutoff_date.isoformat()}'"
    
    results = yield context.call_activity("FetchOldRecords", query)
    yield context.call_activity("ArchiveToBlob", results)
    yield context.call_activity("DeleteFromCosmos", results)
    return "Archived & Deleted"

# ArchiveToBlob.py
def main(records: List[dict]):
    blob_client = BlobServiceClient.from_connection_string(CONN_STR)
    container_client = blob_client.get_container_client("billing-archive")
    for record in records:
        blob_name = f"{record['id']}.json"
        container_client.upload_blob(blob_name, json.dumps(record), overwrite=True)

# DeleteFromCosmos.py
def main(records: List[dict]):
    client = CosmosClient(url, credential)
    container = client.get_container_client("BillingRecords")
    for record in records:
        container.delete_item(record, partition_key=record['partitionKey'])
```

### **Step 3: Implement Facade Function**
Logic:
```python
def main(req):
    record_id = req.params.get('id')
    record = query_cosmos(record_id)
    if record:
        return record
    else:
        return query_blob(record_id)

def query_blob(record_id):
    blob_client = BlobServiceClient.from_connection_string(CONN_STR)
    blob_data = blob_client.get_blob_client("billing-archive", f"{record_id}.json").download_blob().readall()
    return json.loads(blob_data)
```

API contracts are unchanged—this function handles routing internally.

---

## 💸 Cost Optimization Summary

| Component            | Purpose                        | Cost Strategy                    |
|---------------------|--------------------------------|----------------------------------|
| Cosmos DB           | Hot records (<3 months)         | High performance, reduced size  |
| Azure Blob Storage  | Cold records (>3 months)        | Lower cost (Hot/Cold tier)      |
| Durable Functions   | Archival automation             | Serverless, minimal overhead     |

To further reduce blob cost, you can move records to **Cool or Archive tiers** after 6–12 months using lifecycle policies:
```json
{
  "rules": [
    {
      "name": "MoveToCoolTier",
      "enabled": true,
      "filters": { "blobTypes": ["blockBlob"] },
      "actions": {
        "baseBlob": {
          "tierToCool": { "daysAfterModificationGreaterThan": 180 }
        }
      }
    }
  ]
}
```

---

Would you like me to generate ARM templates or Bicep code for automating this entire deployment? Or tailor this toward AKS or container-based processing? You're already strong with DevOps and containers, so we could level this up further.







Awesome, Vishal! Here's a complete set of **ARM templates** to automate the deployment of your cost-optimized billing architecture with Cosmos DB, Blob Storage, Azure Functions, and Durable Functions. This keeps things clean, scriptable, and deployable with CI/CD pipelines—which you already shine at 🌟

---

## 📦 ARM Template Structure Overview

Let’s break it into four modular files:
1. `storage-account.json` – Creates Blob Storage container for archived billing data  
2. `cosmosdb.json` – Sets up Cosmos DB for hot billing records  
3. `function-app.json` – Deploys your Azure Function App (Facade + Durable Functions)  
4. `durable-functions-config.json` – Configures Durable Functions for periodic archival  

---

## 1️⃣ Storage Account Template (`storage-account.json`)
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2022-09-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[resourceGroup().location]",
      "sku": { "name": "Standard_LRS" },
      "kind": "StorageV2",
      "properties": {}
    },
    {
      "type": "Microsoft.Storage/storageAccounts/blobServices/containers",
      "apiVersion": "2022-09-01",
      "name": "[format('{0}/default/billing-archive', parameters('storageAccountName'))]",
      "dependsOn": ["Microsoft.Storage/storageAccounts"],
      "properties": { "publicAccess": "None" }
    }
  ],
  "parameters": {
    "storageAccountName": { "type": "string" }
  }
}
```

---

## 2️⃣ Cosmos DB Template (`cosmosdb.json`)
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "resources": [
    {
      "type": "Microsoft.DocumentDB/databaseAccounts",
      "apiVersion": "2022-05-15",
      "name": "[parameters('cosmosDbName')]",
      "location": "[resourceGroup().location]",
      "kind": "GlobalDocumentDB",
      "properties": {
        "databaseAccountOfferType": "Standard",
        "locations": [
          { "locationName": "[resourceGroup().location]", "failoverPriority": 0 }
        ],
        "capabilities": [{ "name": "EnableServerless" }]
      }
    },
    {
      "type": "Microsoft.DocumentDB/databaseAccounts/sqlDatabases",
      "apiVersion": "2022-05-15",
      "name": "[format('{0}/BillingDB', parameters('cosmosDbName'))]",
      "dependsOn": ["Microsoft.DocumentDB/databaseAccounts"],
      "properties": { "resource": { "id": "BillingDB" } }
    },
    {
      "type": "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers",
      "apiVersion": "2022-05-15",
      "name": "[format('{0}/BillingDB/Records', parameters('cosmosDbName'))]",
      "dependsOn": ["Microsoft.DocumentDB/databaseAccounts/sqlDatabases"],
      "properties": {
        "resource": {
          "id": "Records",
          "partitionKey": {
            "paths": ["/partitionKey"],
            "kind": "Hash"
          }
        }
      }
    }
  ],
  "parameters": {
    "cosmosDbName": { "type": "string" }
  }
}
```

---

## 3️⃣ Function App Template (`function-app.json`)
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "resources": [
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2022-03-01",
      "name": "[parameters('functionAppName')]",
      "location": "[resourceGroup().location]",
      "kind": "functionapp",
      "dependsOn": ["Microsoft.Storage/storageAccounts"],
      "properties": {
        "siteConfig": {
          "appSettings": [
            { "name": "AzureWebJobsStorage", "value": "[parameters('storageConnectionString')]" },
            { "name": "FUNCTIONS_EXTENSION_VERSION", "value": "~4" },
            { "name": "FUNCTIONS_WORKER_RUNTIME", "value": "python" },
            { "name": "WEBSITE_RUN_FROM_PACKAGE", "value": "[parameters('functionZipUrl')]" },
            { "name": "COSMOS_DB_CONN", "value": "[parameters('cosmosDbConnection')]" }
          ]
        },
        "httpsOnly": true,
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('hostingPlanName'))]"
      }
    }
  ],
  "parameters": {
    "functionAppName": { "type": "string" },
    "storageConnectionString": { "type": "securestring" },
    "functionZipUrl": { "type": "string" },
    "hostingPlanName": { "type": "string" },
    "cosmosDbConnection": { "type": "securestring" }
  }
}
```

---

## 4️⃣ Durable Functions Archival Timer Configuration (`durable-functions-config.json`)
You’ll also need the Function App to host a timer-trigger Durable Function that invokes archival logic. Add this logic to your zipped Function code (`functionZipUrl` above). Example configuration:
```json
{
  "bindings": [
    {
      "name": "myTimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 0 * * * *"
    }
  ],
  "scriptFile": "__init__.py",
  "entryPoint": "main"
}
```

---

## ⚙️ Deployment Command
You can deploy any template with:
```bash
az deployment group create \
  --resource-group <your-rg> \
  --template-file <template>.json \
  --parameters <your-parameters>
```

---

Ready to tailor this into Azure DevOps pipelines with linked templates or make it AKS-ready if needed. Want me to bundle all four with Bicep support too?



this is the copilot answer my answer in the psudocode.md file 