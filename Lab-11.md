# Lab 11 — Optimize Query Performance with Vector Indexes on Azure Cosmos DB for NoSQL

## Lab objective

Compare three Azure Cosmos DB for NoSQL vector indexing strategies—`flat`, `quantizedFlat`, and `diskANN`—using identical sample data and search queries. You will deploy a Cosmos DB account with vector search enabled, implement Python functions that measure query latency and Request Unit (RU) consumption, and use a Flask application to compare search results.

**Estimated duration:** 30 minutes

## Prerequisites

- An Azure subscription with permission and quota to create the required resources.
- Visual Studio Code (VS Code).
- The latest Azure CLI.
- Python 3.12 or later.
- Optional: the Ruff extension for VS Code.

**Command convention:** All terminal commands in this guide are single-line PowerShell commands. The Python functions below are multiline code to paste into `.py` files, not terminal commands.

---

## Task 1 — Download and configure the starter project

1. Download the official starter project ZIP file:

   [Download cosmosdb-optimize-query-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/cosmosdb-optimize-query-python.zip)

2. Move the ZIP file to a working folder and extract it.
3. Open VS Code. Select **File > Open Folder...** and open the extracted project folder.
4. Open `azdeploy.py`.
5. Update **only** the two values at the top of the script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name for this exercise.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save `azdeploy.py`. Do not change any other deployment-script settings.
7. Select **Terminal > New Terminal** in VS Code. Make sure the terminal is PowerShell and its current directory is the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and choose the correct Azure account and subscription. Verify the selected subscription:

```powershell
az account show --output table
```

If required, select the intended subscription:

```powershell
az account set --subscription "<subscription-id-or-name>"
```

Register the Cosmos DB resource provider:

```powershell
az provider register --namespace Microsoft.DocumentDB
```

Wait for registration to complete before proceeding.

## Task 2 — Deploy the Cosmos DB account

From the project root, start the deployment script:

```powershell
python azdeploy.py
```

When the menu appears, enter **`1`** to select **Create Cosmos DB account**. This creates an Azure Cosmos DB for NoSQL account with the `EnableNoSQLVectorSearch` capability and creates a database. Deployment can take **5–10 minutes**.

**Leave the deployment terminal open.** You can work on the Python functions in Task 3 while deployment continues.

---

## Task 3 — Implement the vector search and comparison functions

In VS Code, open `client/index_functions.py`. The starter file has `BEGIN ... FUNCTION` markers. Paste each function immediately after its matching marker and before the next marker.

**Indentation tip:** Paste the code flush with the left margin (column 1), select the pasted block, and press **Tab** to align it with the `BEGIN / END` markers. Use **Shift+Tab** to outdent when necessary.

### 3.1 Add vector similarity search with performance tracking

Find `BEGIN VECTOR SIMILARITY SEARCH FUNCTION` and add:

```python
def vector_similarity_search(
    container_name: str,
    query_embedding: list,
    top_n: int = 5
) -> dict:
    """
    Find documents most similar to the query using vector distance.
    This function performs a vector similarity search using the VectorDistance
    function and tracks the RU consumption and execution time. Results are
    ordered by distance (lowest = most similar).

    Args:
        container_name: Name of the container to search
        query_embedding: 256-dimensional query vector
        top_n: Number of results to return

    Returns:
        Dictionary containing results, ru_charge, and execution_time_ms
    """
    container = get_container(container_name)

    # Track execution time for performance comparison
    start_time = time.time()

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

    items = list(container.query_items(
        query=query,
        parameters=[
            {"name": "@topN", "value": top_n},
            {"name": "@queryVector", "value": query_embedding}
        ],
        enable_cross_partition_query=True
    ))

    end_time = time.time()
    execution_time_ms = (end_time - start_time) * 1000

    # This RU value is approximate for multi-page results.
    # For accurate production monitoring, use Azure Monitor.
    ru_charge = 0.0
    try:
        ru_charge = float(container.client_connection.last_response_headers.get(
            "x-ms-request-charge", 0
        ))
    except Exception:
        pass

    results = [
        {
            "chunk_id": item["id"],
            "document_id": item["documentId"],
            "content": item["content"],
            "metadata": item["metadata"],
            "similarity_score": item["similarityScore"]
        }
        for item in items
    ]

    return {
        "results": results,
        "ru_charge": ru_charge,
        "execution_time_ms": round(execution_time_ms, 2)
    }
```

This function executes the same kind of `VectorDistance` query against whichever container name it receives. It records elapsed time in milliseconds and attempts to read the RU charge from the Cosmos DB client response headers. RU values exposed this way can be approximate for paginated queries.

### 3.2 Add the function that compares all three index types

Find `BEGIN COMPARE INDEX PERFORMANCE FUNCTION` and add:

```python
def compare_index_performance(
    query_embedding: list,
    top_n: int = 5
) -> dict:
    """
    Run the same vector search against all three containers and compare performance.
    """
    comparison = {}

    # Test each container with the same query.
    for index_type, container_name in [
        ("flat", CONTAINER_FLAT),
        ("quantizedFlat", CONTAINER_QUANTIZED),
        ("diskANN", CONTAINER_DISKANN)
    ]:
        try:
            result = vector_similarity_search(container_name, query_embedding, top_n)
            comparison[index_type] = {
                "container": container_name,
                "results": result["results"],
                "ru_charge": result["ru_charge"],
                "execution_time_ms": result["execution_time_ms"],
                "result_count": len(result["results"]),
                "status": "success"
            }
        except Exception as e:
            comparison[index_type] = {
                "container": container_name,
                "results": [],
                "ru_charge": 0,
                "execution_time_ms": 0,
                "result_count": 0,
                "status": "error",
                "error": str(e)
            }

    return comparison
```

This function sends the same embedding and result-count limit to each container. It returns result counts, RU charges, execution times, and a status for each index type. If one query fails, its error is reported rather than stopping the entire comparison.

1. Save `index_functions.py`.
2. Review the complete file and make sure the functions are under their correct markers.
3. Preserve the starter file's imports, configuration constants, and existing `get_container()` helper. The functions use `time`, `CONTAINER_FLAT`, `CONTAINER_QUANTIZED`, and `CONTAINER_DISKANN`.

---

## Task 4 — Review the three container configurations

Open `client/setup_containers.py`. This lab asks you to **review** the container setup functions; do not replace the starter configuration.

Find and review these markers:

1. `BEGIN CREATE FLAT CONTAINER FUNCTION`
2. `BEGIN CREATE QUANTIZED CONTAINER FUNCTION`
3. `BEGIN CREATE DISKANN CONTAINER FUNCTION`

Each function uses the following common vector embedding policy:

| Setting | Value | Meaning |
|---|---|---|
| Embedding path | `/embedding` | JSON path where each vector is stored |
| Data type | `float32` | Representation of vector values |
| Dimensions | `256` | Must match the embedding vector length |
| Distance function | `cosine` | Metric used by the vector similarity query |
| Partition key | `/documentId` | Groups records by source document |
| Standard indexing | Include `/*`, exclude `/embedding/*` | Vector data uses its dedicated vector index |

The primary difference between the containers is the vector index type and its search trade-offs:

| Container | Index type | Search behavior | Typical use |
|---|---|---|---|
| `vectors-flat` | `flat` | Exact nearest-neighbor search | Small datasets where exact results matter; can consume more RUs as data grows |
| `vectors-quantized` | `quantizedFlat` | Search on compressed vector representations | Medium datasets or memory-constrained workloads |
| `vectors-diskann` | `diskANN` | Approximate nearest-neighbor search using a graph-based index | Large datasets and production workloads where low latency and RU efficiency matter |

Review how `vectorIndexes` uses the path `/embedding` with the respective index type. The vector embedding policy—including path, dimensions, data type, and distance function—is set when a container is created and cannot be changed afterward. The vector index type is part of the indexing policy and can be changed, but Cosmos DB must transform the index in the background. Creating test containers, loading representative data, and benchmarking them before choosing a production design is a practical approach.

---

## Task 5 — Complete Azure resource deployment

Return to the terminal where `python azdeploy.py` is running. After **Create Cosmos DB account** finishes, enter these menu options in order:

1. Enter **`2`** — **Create containers**. This creates the `vectors-flat`, `vectors-quantized`, and `vectors-diskann` containers.
2. Enter **`3`** — **Configure Entra ID access**. This assigns your user the required Cosmos DB data-plane access.
3. Enter **`4`** — **Check deployment status**. Confirm the account is ready and vector search is enabled.
4. Enter **`5`** — **Retrieve connection info**. This creates the environment-variable file.
5. Enter **`6`** — **Exit**.

From the project root, load the environment variables into the current PowerShell session:

```powershell
. .\.env.ps1
```

Keep this terminal open. If you open another terminal, load the environment file in that terminal as well. Verify the endpoint variable:

```powershell
$env:COSMOS_ENDPOINT
```

Do not share secrets or sensitive environment-variable values.

## Task 6 — Set up the Python environment

From the project root, switch to the `client` folder:

```powershell
Set-Location .\client
```

Create a virtual environment:

```powershell
python -m venv .venv
```

Activate it in Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

If execution policy blocks activation, allow it for the current PowerShell process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the dependencies:

```powershell
pip install -r requirements.txt
```

The dependencies include Flask, Azure Cosmos DB, and Azure Identity libraries.

If the current terminal does not already have the environment variables loaded, load them from the `client` directory:

```powershell
. ..\.env.ps1
```

The PowerShell prompt should display `(.venv)` when the virtual environment is active.

## Task 7 — Start the Flask application

Verify that the terminal is in `client`, the virtual environment is active, and environment variables are loaded. Start the web app:

```powershell
python app.py
```

Leave the app running and open:

[http://127.0.0.1:5000](http://127.0.0.1:5000)

---

## Task 8 — Load sample data into all three containers

The app uses `sample_vectors.json`, which contains **500 support tickets with pre-computed embeddings**. It loads identical data into all three containers for a fair comparison.

1. In **Container Status**, confirm all three containers initially show **0 documents**.
2. In **Load Sample Data**, select **Load Data to All Containers**.
3. Wait for loading to complete. Parallel processing usually takes around **30–45 seconds**.
4. Verify the success message reports the document count and RU costs for each container.

RU write costs may differ slightly because the containers use different index types.

## Task 9 — Compare vector-search performance

1. In **Vector Search Comparison**, select **I can't login to my account** from **Select Query**.
2. Keep the default **Top 5** result count.
3. Select **Compare Index Performance**.
4. Inspect the **Index Performance Comparison** table. Compare:
   - Number of results returned by each container.
   - RU cost for each search.
   - Execution time in milliseconds.
5. Scroll down to compare the returned results side by side.
6. Try additional queries such as `My payment was charged twice` and `Package hasn't arrived yet`, and compare the measurements again.

Interpret the observations carefully:

- For this small dataset, the three indexes should return similar results.
- RU consumption and latency can differ by index type.
- `diskANN` typically demonstrates better RU efficiency as vector datasets grow; one small test is not enough to establish performance for every workload.

## Task 10 — Compare filtered vector-search performance

This task combines a metadata filter with vector similarity ranking.

1. In **Filtered Vector Search Comparison**, select **Protect my account from hackers** from **Select Query**.
2. Select **account** from **Filter by Category**.
3. Select **Compare Filtered Search**.
4. Review the results:
   - Only records whose category is `account` should be returned.
   - Compare RU usage across the three index types.
   - The category filter is applied before vector ranking.
5. Repeat the same query with the **technical** category and compare the filtered results.

## Task 11 — Analyze and choose an index

Use your measurements and these general guidelines as a starting point:

| Scenario | Candidate index | Why |
|---|---|---|
| Small dataset (under 10,000 vectors) | `flat` | Exact results can justify the RU cost |
| Medium dataset with memory constraints | `quantizedFlat` | Compressed vectors can reduce memory use while maintaining useful search quality |
| Large production workload | `diskANN` | Approximate search can improve performance and RU efficiency at scale |

The best choice depends on measured latency, RU budget, data volume, and acceptable recall. Compare representative workloads before making a production decision.

When finished, return to the PowerShell terminal running Flask and press **Ctrl+C** to stop the application.

---

## Task 12 — Clean up Azure resources

Delete the dedicated resource group created for this lab. Replace `<rg-name>` with the exact name configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes every resource within it. Do not use a resource group that contains unrelated resources.

---

## Troubleshooting

### Flask app fails to start

- Confirm the virtual environment is active; the prompt should show `(.venv)`.
- From the `client` directory, run `pip install -r requirements.txt`.
- Run `python app.py` from the `client` directory.
- Ensure environment variables are loaded in this same terminal. From `client`, run `. ..\.env.ps1`.

### Authentication or access denied

- From the project root, run `python azdeploy.py` and select option **3** to configure Entra ID access.
- Verify your user has the **Contributor** role and the **Cosmos DB Built-in Data Contributor** role.
- Check that `$env:COSMOS_ENDPOINT` is populated.
- If permissions have just been assigned, allow time for them to take effect and retry.

### `setup_containers.py` fails

- Confirm the virtual environment is active.
- Ensure `COSMOS_ENDPOINT` and `COSMOS_DATABASE` are set in the current terminal.
- The setup script uses existing containers if they already exist.

### Vector search returns an error or no results

- Run `python azdeploy.py` from the project root and select option **2** to create or verify the three containers.
- Confirm the sample data has been loaded before running a search.
- Verify the embedding policy uses `/embedding`, `float32`, `cosine`, and 256 dimensions.
- Confirm that the account has the `EnableNoSQLVectorSearch` capability.

### Cosmos DB operations fail

- Run `python azdeploy.py` from the project root and select option **4** to check deployment status.
- Confirm the database was created and that the account is ready.
- Verify the environment variables are present in the active terminal.

### Environment variables are missing

- From the project root, run `python azdeploy.py` and select option **5** to retrieve connection information.
- From the project root, load it with `. .\.env.ps1`.
- From `client`, load it with `. ..\.env.ps1`.
- Verify the endpoint using `$env:COSMOS_ENDPOINT`.
- Environment variables loaded into one terminal do not automatically appear in a new terminal.

### Virtual environment activation fails

- From the `client` directory, use `.\.venv\Scripts\Activate.ps1`.
- If execution policy blocks activation, run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` and try again.
- If the activation script is missing, recreate the environment with `python -m venv .venv` from the `client` directory.

---

## Lab completion checklist

- [ ] Starter project extracted and `azdeploy.py` configured.
- [ ] Azure sign-in completed and `Microsoft.DocumentDB` registered.
- [ ] Cosmos DB account deployed with `EnableNoSQLVectorSearch`.
- [ ] Both functions implemented in `client/index_functions.py`.
- [ ] The `flat`, `quantizedFlat`, and `diskANN` container configurations reviewed.
- [ ] All three containers created and Entra ID access configured.
- [ ] Deployment status verified and environment variables loaded.
- [ ] Python virtual environment created and dependencies installed.
- [ ] Flask application started and 500 tickets loaded into each container.
- [ ] Standard vector searches compared for query results, RU cost, and latency.
- [ ] Filtered vector searches compared for `account` and `technical`.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- `flat` performs exact nearest-neighbor search and can be suitable for small datasets.
- `quantizedFlat` uses compressed vectors and can help when memory efficiency matters.
- `diskANN` uses approximate nearest-neighbor search and can be advantageous for large datasets.
- The same query and sample records should be used across containers to make comparisons meaningful.
- Measure both RU consumption and latency; choose the index based on the application's data volume, accuracy requirements, and cost constraints.
