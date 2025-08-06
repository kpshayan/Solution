#Azure Cost Optimization Challenge: Billing Records Tiered Storage Solution

#Problem Statement:
The database currently holds over 2 million records and each record is as large as 300kb of size. Due to this cost of cosmos db has significantly increased over the time

#Constraints:
1. No downtime to existing setup
2. No Changes to API Contracts – The existing read/write APIs for billing records must remain unchanged

#Goal:
1. The records which are older than 3 months needs to be moved to blob storage account
2. The record needs to be retrived by the existing API without changes
3. The latency for fetching old records should be minimal

#Solution:

Using 2 tier architecture will do cost optimisation 

1. Initial records will be saved to cosmos db for low latency retrival which can be considered as Hot tier
2. Every day Azure functions will check for records which are older than 3 months and moves them to Blob storage which is Cool tier
3. If a user requests for records which are older than 3 months then first checks in cosmos db then fetches from blob storage

#Architecuture Diagram:
<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/a9d37f57-dfb3-4f25-a808-7685b3ab7944" />


#Benifits:

1. Cosmos DB provide low latency for records less than 3 months
2. Blob storage Cold tier provides 99.9% availability and low latency of milliseconds
3. Due to this cost is optimised. Below is the approx calculations in cost saving

#Calculation Details
Total records: 2,000,000

Record size: 300KB

Total data volume: ~572GB (2,000,000 × 300KB = 572GB)

Cosmos DB storage cost: ~$0.25/GB/month

Azure Blob Storage (Cool tier) cost: ~$0.01/GB/month

Data retention: 3 months in Cosmos DB (~143GB), 9 months in Blob (~429GB)

Monthly Storage Cost Comparison
Scenario	Monthly Cost
All data in Cosmos DB (~572GB)	~$143
Optimized (3mo in Cosmos, 9mo in Blob)	~$40
Monthly Cost Reduction	~$103 (72%)
Old solution: All data in Cosmos DB = ~$143/month

New solution: Only latest 3 months (~143GB) in Cosmos DB = ~$36/month
& Archived 9 months (~429GB) in Blob = ~$4/month
= Total ~$40/month

Savings per month: ~$103, or about 72% reduction in your Cosmos DB storage costs just by archival, not counting additional savings on RU/s for read/write throughput, which may also decrease as older data access drops.


Pseudocode for Logic layer and Archival 

Retrieval Logic layer

```
def get_data(item_id, partition_key):
    # Attempt to fetch from Cosmos DB
    try:
        item = cosmos_container.read_item(item_id, partition_key=partition_key)
        return item
    except Exception:
        pass  # Not found in Cosmos DB
    
    # Fetch from Blob Storage
    blob_name = f"{item_id}.json"
    blob_client = blob_container_client.get_blob_client(blob_name)
    if blob_client.exists():
        blob_data = blob_client.download_blob().readall()
        return json.loads(blob_data)
    # Item does not exist
    return None
```

Azure Function app logic

```
import os
import datetime
import azure.functions as func
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient

# Config values (set as environment variables or Azure Function App settings)
COSMOS_URL = os.getenv('COSMOS_URL')
COSMOS_KEY = os.getenv('COSMOS_KEY')
COSMOS_DB = os.getenv('COSMOS_DB')
COSMOS_CONTAINER = os.getenv('COSMOS_CONTAINER')
BLOB_CONN_STR = os.getenv('BLOB_CONN_STR')
BLOB_CONTAINER = os.getenv('BLOB_CONTAINER')

def main(mytimer: func.TimerRequest) -> None:
    # Connect to Cosmos DB
    client = CosmosClient(COSMOS_URL, credential=COSMOS_KEY)
    db = client.get_database_client(COSMOS_DB)
    container = db.get_container_client(COSMOS_CONTAINER)
    
    # Connect to Blob Storage
    blob_service_client = BlobServiceClient.from_connection_string(BLOB_CONN_STR)
    blob_container_client = blob_service_client.get_container_client(BLOB_CONTAINER)
    
    # Calculate timestamp cutoff (3 months ago)
    cutoff_date = datetime.datetime.utcnow() - datetime.timedelta(days=90)
    
    # Query older records
    query = "SELECT * FROM c WHERE c.timestamp < @cutoff"
    params = [{"name":"@cutoff", "value": cutoff_date.isoformat()}]
    items = list(container.query_items(query=query, parameters=params, enable_cross_partition_query=True))
    
    for item in items:
        item_id = item['id']
        # Write to blob as JSON
        blob_name = f"{item_id}.json"
        blob_container_client.upload_blob(blob_name, bytes(str(item), 'utf-8'), overwrite=True)
        # Optionally, you can compress first (gzip, etc.) before upload
        
        # Delete item from Cosmos DB
        container.delete_item(item=item_id, partition_key=item['partitionKey'])

    print(f"Archived and deleted {len(items)} items.")

```

Monitoring: Log all moves and errors for audit and support.

Data Retention: Configure Blob lifecycle management for further cost savings (cold/archive tier).

Attached the ChatGPT conversation: 
[a link](https://github.com/kpshayan/Solution/blob/2b24058765897a8c7e05f96e5acff01b1229bba0/I%20have%20a%20scenario%20where%20my%20cosmos%20db%20has%202%20million.pdfS)
