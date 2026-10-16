# Lab 10 — Build a Semantic Search Application with Azure Cosmos DB for NoSQL

## Lab objective

Implement vector similarity search using Azure Cosmos DB for NoSQL. You will deploy a Cosmos DB account with vector search enabled, configure a container with vector embedding and indexing policies, store sample support tickets with pre-computed embeddings, and run semantic and filtered semantic searches through a Flask web application.

**Estimated duration:** 30 minutes

## Prerequisites

- An Azure subscription with permission and quota to provision the required resources.
- Visual Studio Code (VS Code).
- The latest Azure CLI.
- Python 3.12 or later.
- Optional: the Ruff extension for VS Code for Python formatting and linting.

**Command convention:** All terminal commands in this guide are single-line PowerShell commands. Python function implementations are multiline code to paste into the specified `.py` files; they are not shell commands.

---

## Task 1 — Download and configure the starter project

1. Download the official starter project ZIP:

   [Download cosmosdb-implement-vector-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/cosmosdb-implement-vector-python.zip)

2. Move the ZIP file to your working folder and extract it.
3. Launch VS Code. Select **File > Open Folder...** and open the extracted project folder.
4. Open `azdeploy.py`.
5. Change **only** the two values at the top of the script:
   - `"<your-resource-group-name>"` — choose a resource group name for this lab.
   - `"<your-azure-region>"` — choose an Azure region available to your subscription.
6. Save `azdeploy.py`. Do not change anything else in the deployment script.
7. Select **Terminal > New Terminal** in VS Code and use PowerShell from the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and select the intended Azure account and subscription.

Register the Cosmos DB resource provider:

```powershell
az provider register --namespace Microsoft.DocumentDB
```

Wait for the provider registration to complete before proceeding. If you need to check the current subscription, run:

```powershell
az account show --output table
```

If the wrong subscription is selected, set the intended one:

```powershell
az account set --subscription "<subscription-id-or-name>"
```

## Task 2 — Deploy the Cosmos DB account

Make sure the terminal is in the root directory of the extracted project. Start the deployment script:

```powershell
python azdeploy.py
```

When the menu appears, enter **`1`** to choose **Create Cosmos DB account**.

This creates the Azure Cosmos DB for NoSQL account with the `EnableNoSQLVectorSearch` capability and creates a database. Deployment can take **5–10 minutes**.

**Important:** Leave the deployment terminal open while deployment runs. You can continue to Task 3 in VS Code while the deployment proceeds.

---

## Task 3 — Implement the vector search functions

Open `client/vector_functions.py` in VS Code.

The file contains `BEGIN ... FUNCTION` comments. Add each function immediately after its corresponding marker and before the next marker. To preserve indentation, paste the code flush with the left margin (column 1), select the pasted block, and press **Tab** to align it with the surrounding `BEGIN / END` markers. Use **Shift+Tab** to outdent if needed.

### 3.1 Store a support ticket with its vector embedding

Find `BEGIN STORE VECTOR DOCUMENT FUNCTION` and add:

```python
def store_vector_document(
    document_id: str,
    chunk_id: str,
    content: str,
    embedding: list,
    metadata: dict = None
) -> dict:
    """Store a document with its vector embedding for similarity search."""
    container = get_container()
    document = {
        "id": chunk_id,
        "documentId": document_id,
        "content": content,
        "embedding": embedding,
        "metadata": metadata or {},
        "createdAt": datetime.utcnow().isoformat(),
        "chunkIndex": metadata.get("chunkIndex", 0) if metadata else 0
    }
    response = container.upsert_item(body=document)
    ru_charge = response.get_response_headers()["x-ms-request-charge"]
    return {
        "chunk_id": chunk_id,
        "document_id": document_id,
        "ru_charge": float(ru_charge)
    }
```

This stores the ticket text, embedding, and metadata. The required `id` must be unique within its partition. `documentId` is the partition key, grouping chunks from the same source document. `upsert_item` inserts a new item or updates an existing item with the same ID and partition key. The returned RU charge reports the request units consumed.

### 3.2 Perform vector similarity search

Find `BEGIN VECTOR SIMILARITY SEARCH FUNCTION` and add:

```python
def vector_similarity_search(
    query_embedding: list,
    top_n: int = 5
) -> list:
    """
    Find documents most similar to the query using vector distance.
    Results are ordered by distance, with the lowest distance first.
    """
    container = get_container()
    query = """
        SELECT TOP @topN
            c.id,
            c.documentId,
            c.content,
            c.metadata,
            VectorDistance(c.embedding, @queryVector) AS similarityScore
        FROM c
        ORDER BY VectorDistance(c.embedding, @queryVector)
    """
    items = container.query_items(
        query=query,
        parameters=[
            {"name": "@topN", "value": top_n},
            {"name": "@queryVector", "value": query_embedding}
        ],
        enable_cross_partition_query=True
    )
    return [
        {
            "chunk_id": item["id"],
            "document_id": item["documentId"],
            "content": item["content"],
            "metadata": item["metadata"],
            "similarity_score": item["similarityScore"]
        }
        for item in items
    ]
```

`VectorDistance` compares the query vector with each stored embedding. Results are sorted by distance ascending, so the closest matches appear first. The query vector supplied by the application is 256-dimensional.

### 3.3 Perform filtered vector search

Find `BEGIN FILTERED VECTOR SEARCH FUNCTION` and add:

```python
def filtered_vector_search(
    query_embedding: list,
    category: str = None,
    top_n: int = 5
) -> list:
    """
    Combine vector similarity search with optional metadata filtering.
    Filter by category first, then rank the matching items by vector distance.
    """
    container = get_container()
    where_clause = ""
    parameters = [
        {"name": "@topN", "value": top_n},
        {"name": "@queryVector", "value": query_embedding}
    ]
    if category:
        where_clause = "WHERE c.metadata.category = @category"
        parameters.append({"name": "@category", "value": category})
    query = f"""
        SELECT TOP @topN
            c.id,
            c.documentId,
            c.content,
            c.metadata,
            VectorDistance(c.embedding, @queryVector) AS similarityScore
        FROM c
        {where_clause}
        ORDER BY VectorDistance(c.embedding, @queryVector)
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
            "similarity_score": item["similarityScore"]
        }
        for item in items
    ]
```

This hybrid search optionally filters by `metadata.category` before ranking results by vector distance. If no category is provided, the search is unfiltered.

1. Save `vector_functions.py`.
2. Review the full file and ensure all three functions are under their correct markers.
3. Keep the starter file's existing imports and `get_container()` helper, which the functions use.

---

## Task 4 — Review the vector container configuration

Open `client/setup_container.py` in VS Code. Find `BEGIN CREATE VECTOR CONTAINER FUNCTION` and review the function below. The deployment script creates the container with these policies; this task is to understand the configuration.

```python
def create_vector_container():
    """Create a container with vector embedding and indexing policies."""
    database = get_database()
    container_name = os.environ.get("COSMOS_CONTAINER", "vectors")
    vector_embedding_policy = {
        "vectorEmbeddings": [
            {
                "path": "/embedding",
                "dataType": "float32",
                "distanceFunction": "cosine",
                "dimensions": 256
            }
        ]
    }
    indexing_policy = {
        "indexingMode": "consistent",
        "automatic": True,
        "includedPaths": [
            {"path": "/*"}
        ],
        "excludedPaths": [
            {"path": "/embedding/*"}
        ],
        "vectorIndexes": [
            {
                "path": "/embedding",
                "type": "diskANN"
            }
        ]
    }
    container = database.create_container_if_not_exists(
        id=container_name,
        partition_key=PartitionKey(path="/documentId"),
        indexing_policy=indexing_policy,
        vector_embedding_policy=vector_embedding_policy
    )
    return container
```

Understand the key settings:

| Setting | Value | Purpose |
|---|---|---|
| Vector embedding path | `/embedding` | Location of each vector in a stored document |
| Data type | `float32` | Data type used for vector values |
| Dimensions | `256` | Must match the embedding model's output dimension |
| Distance function | `cosine` | Metric used for vector similarity |
| Vector index | `diskANN` | Supports efficient approximate nearest-neighbor search |
| Excluded standard index path | `/embedding/*` | Vector values use a dedicated vector index rather than the standard index |
| Partition key | `/documentId` | Groups items by source document |

Do not replace or alter the starter code unless instructed by the lab.

---

## Task 5 — Complete the Azure resource deployment

Return to the terminal running `python azdeploy.py`. After **Create Cosmos DB account** has completed, enter the following menu options **in this order**:

1. Enter **`2`** — **Create container**. This creates the vector container with the embedding and indexing policies required for similarity search.
2. Enter **`3`** — **Configure Entra ID access**. This assigns your user the required data-plane role.
3. Enter **`4`** — **Check deployment status**. Verify the Cosmos DB account is ready and vector search capability is enabled.
4. Enter **`5`** — **Retrieve connection info**. This creates the environment-variable file.
5. Enter **`6`** — **Exit**.

Load the environment variables in the same PowerShell terminal from the project root:

```powershell
. .\.env.ps1
```

Keep this terminal open. If you close it or create another terminal, load the environment file in that new terminal too. Verify that the endpoint variable is set:

```powershell
$env:COSMOS_ENDPOINT
```

Do not share credentials or sensitive environment-variable values.

## Task 6 — Set up the Python environment

In the VS Code terminal, navigate to the client directory:

```powershell
Set-Location .\client
```

Create the virtual environment:

```powershell
python -m venv .venv
```

Activate it on Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks the activation script, allow it for the current process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the required dependencies:

```powershell
pip install -r requirements.txt
```

This installs the Flask, Azure Cosmos DB, and Azure Identity libraries. Ensure the environment variables are loaded in the terminal session that will run the application. From the `client` directory, run:

```powershell
. ..\.env.ps1
```

The terminal prompt should show `(.venv)` when the environment is active.

---

## Task 7 — Start the Flask application

Make sure the terminal is in the `client` directory, the virtual environment is active, and environment variables are loaded. Start the app:

```powershell
python app.py
```

Leave the app running. Open a browser and navigate to:

[http://127.0.0.1:5000](http://127.0.0.1:5000)

## Task 8 — Load sample vector data

The sample data contains **12 support tickets** across billing, technical, account, and shipping categories. Each ticket includes a pre-computed **256-dimensional embedding** stored in `sample_vectors.json`.

1. In the app's **Load Sample Data** section, select **Load Vector Data**.
2. Wait for the operation to finish.
3. Verify the Results section displays a success message, the number of tickets loaded, and the total RU (Request Unit) charge.

The app calls your `store_vector_document()` function to insert the sample tickets into Cosmos DB.

## Task 9 — Test vector similarity search

1. In **Vector Similarity Search**, select **I can't login to my account** from the **Select Query** dropdown.
2. Keep **Top 5** as the result count and select **Search**.
3. Review the results and their similarity scores. Tickets about authentication and account access should appear near the top, even if they use different words from the query.
4. Try other queries such as **My payment was charged twice** and **Package hasn't arrived yet**. Observe how semantic matching returns relevant cases even when exact terms differ.

The app uses `vector_similarity_search()` and the Cosmos DB `VectorDistance` function to rank results.

## Task 10 — Test filtered vector search

1. In **Filtered Vector Search**, select **I can't login to my account** from **Select Query**.
2. In **Filter by Category**, select **technical**.
3. Select **Search with Filter**.
4. Review the results. Only tickets in the `technical` category should be returned, ranked by semantic similarity.
5. Repeat the same query with the **account** category. Compare the results and observe that the search remains semantically relevant while being limited to the chosen category.

When finished, return to the terminal running Flask and press **Ctrl+C** to stop the application.

---

## Task 11 — Clean up Azure resources

Delete the resource group created for this lab. Replace `<rg-name>` with the exact resource group name configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes all resources it contains. Use a dedicated resource group for this lab. If you used an existing group with unrelated resources, those resources will also be deleted.

---

## Troubleshooting

### Flask app fails to start

- Ensure the virtual environment is active; the terminal prompt should show `(.venv)`.
- From the `client` directory, install dependencies with `pip install -r requirements.txt`.
- Ensure you are in `client` when running `python app.py`.
- Load environment variables in the same terminal. From `client`, run `. ..\.env.ps1`.

### Authentication or access denied errors

- Run `python azdeploy.py` from the project root and choose option **3** to configure Entra ID access.
- Verify your user has both the **Contributor** role and **Cosmos DB Built-in Data Contributor** role.
- Check that `$env:COSMOS_ENDPOINT` is populated.

### Vector search returns no results or errors

- Run `python azdeploy.py` from the project root and choose option **2** to verify/create the vector container.
- Check deployment status with option **4** and verify the vector embedding policy is configured.
- Load the sample tickets in the Flask app before searching.
- Verify the stored embedding vectors and query vectors are 256-dimensional and use the configured `/embedding` path.

### `setup_container.py` fails

- Ensure the virtual environment is active.
- Ensure `COSMOS_ENDPOINT`, `COSMOS_DATABASE`, and `COSMOS_CONTAINER` are set.
- The script uses an existing container if it already exists.

### Cosmos DB operations fail

- Use deployment script option **4** to verify that the Cosmos DB account is ready.
- Confirm the database was created during deployment.
- Confirm the account has the `EnableNoSQLVectorSearch` capability.

### Environment variables are missing

- Run `python azdeploy.py` from the project root and select option **5** to retrieve connection information.
- From the project root, load variables with `. .\.env.ps1`.
- From the `client` directory, load them with `. ..\.env.ps1`.
- Verify the endpoint with `$env:COSMOS_ENDPOINT`.
- Every new terminal session needs the environment file loaded separately.

### Python virtual environment activation fails

- From the `client` directory, use `.\.venv\Scripts\Activate.ps1`.
- If execution policy blocks activation, run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` and retry.
- If the activation script is missing, recreate the virtual environment with `python -m venv .venv` from the `client` directory.

---

## Lab completion checklist

- [ ] Starter project extracted and `azdeploy.py` configured.
- [ ] Azure login completed and `Microsoft.DocumentDB` provider registered.
- [ ] Cosmos DB account deployed with `EnableNoSQLVectorSearch`.
- [ ] All three vector functions implemented in `client/vector_functions.py`.
- [ ] Vector container embedding and indexing policies reviewed.
- [ ] Container created, Entra ID access configured, and deployment status verified.
- [ ] Environment variables loaded in the terminal session.
- [ ] Virtual environment created and dependencies installed.
- [ ] Flask app started and all 12 sample tickets loaded.
- [ ] Vector similarity search tested with multiple query examples.
- [ ] Filtered vector search tested with `technical` and `account` categories.
- [ ] Flask stopped and Azure resources deleted when no longer needed.

## Key takeaways

- Semantic search retrieves items by meaning rather than exact keyword overlap.
- Cosmos DB's `VectorDistance` function ranks stored embeddings against a query vector.
- The vector embedding policy must match the embedding path, data type, distance metric, and dimensions.
- A `diskANN` vector index supports efficient approximate nearest-neighbor search.
- Hybrid search combines metadata filters with vector ranking to narrow results to a relevant category.
