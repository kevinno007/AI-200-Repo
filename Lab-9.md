# Lab 9 — Build a RAG Document Store on Azure Cosmos DB for NoSQL

## Lab objective

Build a document store for a retrieval-augmented generation (RAG) application using Azure Cosmos DB for NoSQL. You will deploy the database resources, implement Python functions for storing and retrieving document chunks, test them through a Flask application, and query the stored context using Cosmos DB SQL.

**Estimated duration:** 30 minutes

## Prerequisites

- Azure subscription with permissions and quota to create the required resources.
- Visual Studio Code (VS Code).
- Latest Azure CLI.
- Python 3.12 or later.
- Internet access to download the starter project.

**Command convention:** Terminal commands in this guide are single-line PowerShell commands. Python function implementations and SQL queries are code to paste into the specified editor/interface, not shell commands.

## Task 1 — Download and configure the starter project

1. Download the official starter ZIP: [cosmosdb-build-query-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/cosmosdb-build-query-python.zip).
2. Move the ZIP file to your working folder and extract it.
3. Open VS Code, select **File > Open Folder...**, and open the extracted project folder.
4. Open `azdeploy.py`. Update **only** the two values at the top of the script: the resource group name (`<your-resource-group-name>`) and Azure region (`<your-azure-region>`). Choose a region available to your subscription. Save the file; do not change anything else in the script.
5. In VS Code, select **Terminal > New Terminal** and use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and select the correct account and subscription. Verify the active subscription:

```powershell
az account show --output table
```

If needed, select the intended subscription:

```powershell
az account set --subscription "<subscription-id-or-name>"
```

Register the Cosmos DB resource provider:

```powershell
az provider register --namespace Microsoft.DocumentDB
```

Check its registration state if needed:

```powershell
az provider show --namespace Microsoft.DocumentDB --query registrationState --output tsv
```

Wait for `Registered` before proceeding.

## Task 2 — Start Azure resource deployment

From the project root, run:

```powershell
python azdeploy.py
```

When the menu appears, enter **`1`** for **Create Cosmos DB account**. This creates the Azure Cosmos DB for NoSQL account, database, and container. Deployment can take **5–10 minutes**.

**Keep the deployment terminal open** while the account is being created. You can work on the Python functions in Task 3 while deployment continues. Do not choose later menu options until account creation has completed.

## Task 3 — Implement the RAG document functions

In VS Code, open `client/rag_functions.py`. Find each `BEGIN ... FUNCTION` comment and paste the matching function immediately after it, before the next marker. Preserve the existing imports and helper functions. If indentation is incorrect, paste at column 1, select the pasted block, and use **Tab** to align it with the surrounding code.

### 3.1 Store a document chunk

Find `BEGIN STORE DOCUMENT CHUNK FUNCTION` and add:

```python
def store_document_chunk(
    document_id: str,
    chunk_id: str,
    content: str,
    metadata: dict = None,
    embedding: list = None
) -> dict:
    """Store a document chunk with metadata and optional embedding placeholder."""
    container = get_container()
    chunk = {
        "id": chunk_id,
        "documentId": document_id,
        "content": content,
        "metadata": metadata or {},
        "embedding": embedding or [],
        "createdAt": datetime.utcnow().isoformat(),
        "chunkIndex": metadata.get("chunkIndex", 0) if metadata else 0
    }
    response = container.upsert_item(body=chunk)
    ru_charge = response.get_response_headers()["x-ms-request-charge"]
    return {
        "chunk_id": chunk_id,
        "document_id": document_id,
        "ru_charge": float(ru_charge)
    }
```

This stores text and metadata. `upsert_item` inserts or updates an item with the same ID and partition key. The RU charge reports the request-unit cost of the operation.

### 3.2 Retrieve chunks by document ID

Find `BEGIN GET CHUNKS BY DOCUMENT FUNCTION` and add:

```python
def get_chunks_by_document(document_id: str, limit: int = 100) -> list:
    """Retrieve all chunks for a document, ordered by chunk index."""
    container = get_container()
    query = """
        SELECT c.id, c.content, c.metadata, c.chunkIndex, c.createdAt
        FROM c
        WHERE c.documentId = @documentId
        ORDER BY c.chunkIndex
        OFFSET 0 LIMIT @limit
    """
    items = container.query_items(
        query=query,
        parameters=[
            {"name": "@documentId", "value": document_id},
            {"name": "@limit", "value": limit}
        ],
        partition_key=document_id
    )
    return [
        {
            "chunk_id": item["id"],
            "content": item["content"],
            "metadata": item["metadata"],
            "chunk_index": item["chunkIndex"],
            "created_at": item["createdAt"]
        }
        for item in items
    ]
```

The `documentId` partition key scopes the query to one partition, making retrieval efficient when the document ID is known.

### 3.3 Search chunks by metadata

Find `BEGIN SEARCH CHUNKS BY METADATA FUNCTION` and add:

```python
def search_chunks_by_metadata(filters: dict, limit: int = 10) -> list:
    """Search for chunks across documents using metadata filters."""
    container = get_container()
    where_clauses = []
    parameters = []

    if "source" in filters and filters["source"]:
        where_clauses.append("c.metadata.source = @source")
        parameters.append({"name": "@source", "value": filters["source"]})

    if "category" in filters and filters["category"]:
        where_clauses.append("c.metadata.category = @category")
        parameters.append({"name": "@category", "value": filters["category"]})

    if "tags" in filters and filters["tags"]:
        where_clauses.append("ARRAY_CONTAINS(c.metadata.tags, @tag)")
        parameters.append({"name": "@tag", "value": filters["tags"][0]})

    where_clause = " AND ".join(where_clauses) if where_clauses else "1=1"
    parameters.append({"name": "@limit", "value": limit})
    query = f"""
        SELECT c.id, c.documentId, c.content, c.metadata, c.chunkIndex
        FROM c
        WHERE {where_clause}
        OFFSET 0 LIMIT @limit
    """
    items = container.query_items(
        query=query,
        parameters=parameters,
        enable_cross_partition_query=True
    )
    return [
        {
            "chunk_id": item["id"],
            "document_id": item["documentId"],
            "content": item["content"],
            "metadata": item["metadata"],
            "chunk_index": item["chunkIndex"]
        }
        for item in items
    ]
```

The function supports `source`, `category`, and `tags` filters. Because the target document is not known in advance, the query can search across partitions.

### 3.4 Retrieve one chunk by ID

Find `BEGIN GET CHUNK BY ID FUNCTION` and add:

```python
def get_chunk_by_id(document_id: str, chunk_id: str) -> dict:
    """Retrieve one chunk using an efficient point read."""
    container = get_container()
    try:
        item = container.read_item(item=chunk_id, partition_key=document_id)
        return {
            "chunk_id": item["id"],
            "document_id": item["documentId"],
            "content": item["content"],
            "metadata": item["metadata"],
            "chunk_index": item["chunkIndex"],
            "created_at": item["createdAt"],
            "embedding": item.get("embedding", [])
        }
    except exceptions.CosmosResourceNotFoundError:
        return None
```

A point read needs both the item ID and partition key. If the item does not exist, the function returns `None`.

1. Save `rag_functions.py`.
2. Review the rest of the application code.
3. Ensure each function is placed under the correct marker and that no code was pasted inside another function. Keep the starter file's existing imports and `get_container()` helper; these functions depend on them.

## Task 4 — Finish deployment and load environment variables

Return to the deployment terminal. After **Create Cosmos DB account** finishes, choose these menu options in order:

1. Enter **`2`** — **Configure Entra ID access**. This assigns your user the required data-plane access.
2. Enter **`3`** — **Check deployment status**. Verify resources are ready.
3. Enter **`4`** — **Retrieve connection info**. This creates the environment-variable file.
4. Enter **`5`** — **Exit**.

From the project root, load the environment variables in the same PowerShell terminal:

```powershell
. .\.env.ps1
```

Keep this terminal open. Variables loaded in one terminal are not automatically available in a newly opened terminal. To check the endpoint variable:

```powershell
$env:COSMOS_ENDPOINT
```

Do not share credentials or sensitive environment-variable values.

## Task 5 — Understand the RAG document schema

The container's partition key is **`/documentId`**. Each chunk uses this schema:

| Field | Purpose |
|---|---|
| `id` | Unique chunk ID within the partition; required by Cosmos DB |
| `documentId` | Source document ID and partition key |
| `content` | Text content of the chunk |
| `metadata` | Flexible object for source, category, tags, and custom attributes |
| `embedding` | Placeholder array for vector embeddings |
| `chunkIndex` | Position of the chunk within the source document |
| `createdAt` | Timestamp when the chunk was stored |

The schema supports point reads, efficient single-partition queries for known documents, cross-partition metadata searches, and a future vector-search design. **This lab does not generate embeddings or implement vector search**; `embedding` is only a placeholder.

## Task 6 — Create a virtual environment and start Flask

Open another PowerShell terminal if necessary. From the project root, enter the client directory:

```powershell
Set-Location .\client
```

Create a virtual environment:

```powershell
python -m venv .venv
```

Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

If script execution is blocked, allow scripts for the current PowerShell process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install dependencies:

```powershell
pip install -r requirements.txt
```

Load environment variables in this terminal. From the `client` directory, run:

```powershell
. ..\.env.ps1
```

Start Flask:

```powershell
python app.py
```

Leave this terminal running. Open a browser at [http://127.0.0.1:5000](http://127.0.0.1:5000).

## Task 7 — Load sample document chunks

In the Flask interface:

1. Find **Load Sample Data**.
2. Select **Load Sample Chunks**.
3. Verify the Results section reports success, the number of chunks loaded, and the total RU charge.

The sample dataset contains **12 chunks across four documents**, representing fictional Azure documentation articles.

## Task 8 — Run automated tests

1. Find **Run Test Workflow**.
2. Select **Run Tests**.
3. Confirm all five tests pass:
   - Store document chunks.
   - Get chunks by document ID.
   - Search by category.
   - Search by tag.
   - Point read by ID.

If a test fails, check function names, marker placement, indentation, and returned fields. Also verify that deployment completed, environment variables are loaded, and Entra ID access was configured.

## Task 9 — Retrieve chunks by document

1. In **Get Chunks by Document**, select a document from the dropdown, such as `doc-azure-overview`.
2. Select **Get Chunks**.
3. Verify that the results show chunks in `chunkIndex` order with content and metadata tags.

## Task 10 — Search by metadata

1. In **Search by Metadata**, select `ai-applications` from the **Category** dropdown.
2. Leave the **Tag** field empty and select **Search**.
3. Confirm that four chunks are returned, with tags such as `rag`, `embeddings`, `chunking`, and `metadata`.
4. Enter `embeddings` in the **Tag** field and select **Search** again.
5. Confirm that fewer results are returned: the chunks must match both the `ai-applications` category and the `embeddings` tag.

This demonstrates how combined metadata filters return more targeted RAG context.

## Task 11 — Query document context using Cosmos DB SQL

In the Flask app's **Query Explorer**, enter each query into the SQL Query field and select **Execute Query**.

### Query 1 — Retrieve chunks for one document in order

```sql
SELECT c.id, c.chunkIndex, c.content, c.metadata
FROM c
WHERE c.documentId = 'doc-azure-overview'
ORDER BY c.chunkIndex
```

Review the chunks for `doc-azure-overview`, ordered by their chunk index.

### Query 2 — Search for a category across documents

```sql
SELECT c.documentId, c.id, c.content, c.metadata.category
FROM c
WHERE c.metadata.category = 'cloud-services'
```

Review the chunks whose category is `cloud-services`.

### Query 3 — List documents with category and source metadata

```sql
SELECT DISTINCT c.documentId, c.metadata.category, c.metadata.source
FROM c
```

Review the distinct document/category/source combinations.

### Query 4 — Find chunks containing a tag

```sql
SELECT c.documentId, c.id, c.content, c.metadata.tags
FROM c
WHERE ARRAY_CONTAINS(c.metadata.tags, 'compute')
```

Review the chunks whose `metadata.tags` array contains `compute`.

When you have completed the queries, return to the PowerShell terminal running Flask and press **Ctrl+C** to stop the application.

## Task 12 — Clean up Azure resources

When finished, delete the dedicated resource group configured in `azdeploy.py`. Replace `<rg-name>` with the actual resource group name:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes all resources inside it. If you used an existing group containing unrelated resources, those resources will also be deleted.

## Troubleshooting

### Flask app fails to start

- Confirm the virtual environment is active; the prompt should show `(.venv)`.
- From `client`, install dependencies using `pip install -r requirements.txt`.
- Run `python app.py` from the `client` directory.
- Ensure the environment variables are loaded in that same terminal. From `client`, run `. ..\.env.ps1`.
- If port 5000 is already in use, stop the process using it or resolve the port conflict before restarting Flask.

### Authentication or access denied

- Run `python azdeploy.py` from the project root and select option **2** to configure Entra ID access.
- Verify your user has the **Contributor** role and **Cosmos DB Built-in Data Contributor** role, as required by the lab.
- Check that `$env:COSMOS_ENDPOINT` is populated.
- If access was just granted, allow time for the role assignment to take effect, then retry.

### Cosmos DB operations fail

- Run `python azdeploy.py` from the project root and choose option **3** to check deployment status.
- Confirm the database and container were created.
- Confirm the container partition key is `/documentId`.
- Ensure option **4** generated the environment-variable file and that it was loaded into the terminal session.

### Environment variables are missing

- From the project root, run `python azdeploy.py` and choose option **4** to retrieve connection info.
- From the project root, load the variables with `. .\.env.ps1`.
- From `client`, load them with `. ..\.env.ps1`.
- Verify the endpoint with `$env:COSMOS_ENDPOINT`.
- Each new terminal needs the environment file loaded separately.

### Virtual environment activation fails

- From the `client` directory, use `.\.venv\Scripts\Activate.ps1`.
- If execution policy blocks activation, run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` and retry.
- If the activation script is missing, recreate the environment with `python -m venv .venv` from the `client` directory.

## Lab completion checklist

- [ ] Starter project extracted and `azdeploy.py` configured.
- [ ] Azure login completed and `Microsoft.DocumentDB` registered.
- [ ] Cosmos DB account, database, and container deployed.
- [ ] All four functions implemented in `client/rag_functions.py`.
- [ ] Entra ID access configured and deployment status verified.
- [ ] Environment variables loaded in the terminal running Flask.
- [ ] Flask app started and sample chunks loaded.
- [ ] All five automated tests passed.
- [ ] Document retrieval and metadata searches tested.
- [ ] All four SQL queries executed successfully.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- A RAG document store persists chunked text and metadata for later retrieval by an AI application.
- The `/documentId` partition key groups chunks from the same source document.
- Parameterized queries retrieve known-document chunks; metadata searches may span partitions.
- Point reads require both the item ID and partition key.
- The embedding field is a placeholder in this lab; vector generation and vector search are not implemented.
