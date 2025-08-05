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


Steps to solve the problem

step-1 --> Create Azure Blob Storage for store the cold data (3 month ago)

bash Scripting for step-1 

az login
az storage account create --name mystorageaccount --resource-group myrg --location eastus --sku Standard_LRS
az storage container create --name billing-archive --account-name mystorageaccount

step-2 --> Set Up Durable Function for Archival

Trigger: Timer (daily)
Logic:

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


step-->3 Implement Facade Function

Logic: 

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

    API contracts are unchanged—this function handles routing internally.


 Cost Optimization Summary
| Component | Purpose | Cost Strategy | 
| Cosmos DB | Hot records (<3 months) | High performance, reduced size | 
| Azure Blob Storage | Cold records (>3 months) | Lower cost (Hot/Cold tier) | 
| Durable Functions | Archival automation | Serverless, minimal overhead | 


Let’s break it into four modular files:
- storage-account.json – Creates Blob Storage container for archived billing data
- cosmosdb.json – Sets up Cosmos DB for hot billing records
- function-app.json – Deploys your Azure Function App (Facade + Durable Functions)
- durable-functions-config.json – Configures Durable Functions for periodic archiva
