# Lab 17 — Implement Semantic Search in Azure Managed Redis

## Lab objective

Build a product semantic-search application using Azure Managed Redis and RediSearch. You will deploy Azure Managed Redis, authenticate with Microsoft Entra ID, create an HNSW vector index, store product embeddings and metadata in Redis hashes, and run K-nearest-neighbor (KNN) similarity searches from a Flask web interface.

**Estimated duration:** 30 minutes.

## Prerequisites

- An Azure subscription with permissions and quota to provision the required resources.
- Visual Studio Code (VS Code).
- Python 3.12 or later.
- Latest Azure CLI.
- Azure CLI `redisenterprise` extension version **2.75.0 or later**. The setup steps below install or upgrade it.
- Optional: Ruff extension for VS Code to format and lint Python code.

**Command convention:** All terminal commands in this guide use Windows PowerShell and are single-line commands. The multiline Python implementations are code to paste into `client/vector_functions.py`, not terminal commands.

---

## Task 1 — Download the starter project and deploy Azure Managed Redis

1. Download the official starter ZIP file:

   [Download amr-vector-query-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/amr-vector-query-python.zip)

2. Move the ZIP file to your working folder and extract it.
3. Open VS Code, select **File > Open Folder...**, and open the extracted project folder.
4. Open `azdeploy.py`.
5. Change **only** the two values at the top of the script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name for this lab.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save the file. Do not change any other settings in `azdeploy.py`.
7. In VS Code, select **Terminal > New Terminal**. Use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and select the correct Azure account and subscription.

Register the Azure Managed Redis resource provider:

```powershell
az provider register --namespace Microsoft.Cache
```

Install or upgrade the Azure CLI extension. Version 2.75.0 or later is required to configure Microsoft Entra ID access:

```powershell
az extension add --upgrade --name redisenterprise
```

Start the deployment script from the project root:

```powershell
python azdeploy.py
```

When the menu appears, enter **`1`** for **Create Azure Managed Redis resource**. The script creates the resource group if it does not exist and deploys Azure Managed Redis. Provisioning takes approximately **5–10 minutes**.

**Leave the deployment script running.** Continue to the implementation tasks while Azure provisions the resource. Check the terminal periodically for deployment errors.

---

## Task 2 — Implement the Redis client connection

Open `client/vector_functions.py` in VS Code. The file contains `BEGIN / END` comment markers for the missing code sections.

**Indentation tip:** Paste each code block flush with the left margin (column 1), select the pasted lines, and press **Tab** to align the block with the markers. Use **Shift+Tab** to outdent if needed. Use consistent Python indentation.

Find `# BEGIN CONNECTION CODE SECTION` and add:

```python
def get_client() -> redis.Redis:
    """Create a Redis client for Azure Managed Redis using Microsoft Entra ID."""
    redis_host = os.environ.get("REDIS_HOST")

    if not redis_host:
        raise ValueError("REDIS_HOST environment variable must be set")

    credential_provider = create_from_default_azure_credential(
        ("https://redis.azure.com/.default",),
    )

    return redis.Redis(
        host=redis_host,
        port=10000,
        ssl=True,
        decode_responses=False,
        credential_provider=credential_provider,
        socket_timeout=30,
        socket_connect_timeout=30,
    )
```

Important details:

- The client reads the endpoint from the `REDIS_HOST` environment variable.
- Azure Managed Redis uses TLS over port `10000`.
- `create_from_default_azure_credential()` uses `DefaultAzureCredential` to obtain Microsoft Entra tokens and refresh them automatically.
- `decode_responses=False` keeps values as bytes. This matters because vector embeddings are stored as serialized `float32` bytes.
- The code raises a clear error if `REDIS_HOST` is missing.

Save the file. Do not modify `client/app.py`; the starter Flask app already calls the functions you will implement.

---

## Task 3 — Add code to create the vector index

The RediSearch index defines text fields for product metadata and a vector field named `embedding`. The vector index uses **HNSW**, cosine distance, and **8 dimensions**, matching the sample embeddings.

Find `# BEGIN CREATE VECTOR INDEX CODE SECTION` and add this method within the existing vector-store class:

```python
def _create_vector_index(self):
    """Create a RediSearch index for product semantic search."""
    try:
        schema = (
            TextField("name"),
            TextField("category"),
            TextField("product_id"),
            VectorField(
                "embedding",
                "HNSW",
                {
                    "TYPE": "FLOAT32",
                    "DIM": VECTOR_DIM,
                    "DISTANCE_METRIC": "COSINE",
                },
            ),
        )

        definition = IndexDefinition(
            prefix=["product:"],
            index_type=IndexType.HASH,
        )

        self.r.ft(VECTOR_INDEX_NAME).create_index(
            fields=schema,
            definition=definition,
        )
    except redis.ResponseError as e:
        if "already exists" not in str(e):
            raise Exception(f"Error creating vector index: {e}")
    except Exception as e:
        raise Exception(f"Error creating vector index: {e}")
```

The index configuration means:

- `TextField` indexes product metadata fields such as name, category, and product ID.
- `VectorField` defines `embedding` as an HNSW vector field.
- `TYPE: FLOAT32` matches the serialized NumPy embedding values.
- `DIM: VECTOR_DIM` uses the starter's configured embedding size of 8.
- `DISTANCE_METRIC: COSINE` ranks products using cosine distance.
- `prefix=["product:"]` limits indexed hashes to keys beginning with `product:`.
- `IndexType.HASH` tells RediSearch that products are stored as Redis hashes.

Save the file.

---

## Task 4 — Implement product vector storage

Find `# BEGIN STORE PRODUCT CODE SECTION` and add:

```python
def store_product(
    self,
    vector_key: str,
    vector: list[float],
    metadata: dict[str, str] | None = None,
) -> tuple[bool, str]:
    """Store or update a product hash containing embedding and metadata."""
    try:
        embedding = np.array(vector, dtype=np.float32)
        data: dict[str, Any] = {"embedding": embedding.tobytes()}

        if metadata:
            for key, value in metadata.items():
                data[key] = str(value)

        result = self.r.hset(vector_key, mapping=data)
        if result > 0:
            return True, f"Product stored successfully under key '{vector_key}'"
        return True, f"Product updated successfully under key '{vector_key}'"
    except Exception as e:
        return False, f"Error storing product: {e}"
```

This function:

1. Converts the input embedding to a NumPy `float32` array.
2. Converts the array to bytes and stores it in the `embedding` hash field.
3. Stores metadata fields such as `product_id`, `name`, and `category` as strings.
4. Uses `HSET` to create the product hash or update an existing hash.

The product key should use the `product:` prefix, for example `product:001`.

Save the file.

---

## Task 5 — Implement KNN similarity search

Find `# BEGIN SEARCH SIMILAR PRODUCTS CODE SECTION` and add:

```python
def search_similar_products(
    self,
    query_vector: list[float],
    top_k: int = 3,
) -> tuple[bool, list[dict[str, Any]] | str]:
    """Run a KNN vector query against product embeddings."""
    try:
        query_bytes = np.array(query_vector, dtype=np.float32).tobytes()

        knn_query = (
            Query(f"*=>[KNN {top_k} @embedding $query_vec AS score]")
            .return_fields("name", "category", "product_id", "score")
            .sort_by("score")
            .dialect(2)
        )

        results = self.r.ft(VECTOR_INDEX_NAME).search(
            knn_query,
            query_params={"query_vec": query_bytes},
        )

        if results.total == 0:
            return False, "No products found in Redis. Load sample products first."

        similarities: list[dict[str, Any]] = []
        for doc in results.docs:
            similarities.append(
                {
                    "key": doc.id,
                    "similarity": float(doc.score),
                    "product_id": doc.product_id.decode() if isinstance(doc.product_id, bytes) else doc.product_id,
                    "name": doc.name.decode() if isinstance(doc.name, bytes) else doc.name,
                    "category": doc.category.decode() if isinstance(doc.category, bytes) else doc.category,
                }
            )

        return True, similarities
    except Exception as e:
        return False, f"Error searching products: {e}"
```

The query uses RediSearch's KNN syntax:

- `KNN {top_k}` specifies how many nearest neighbors to return.
- `@embedding` is the vector field in the index.
- `$query_vec` is the query vector passed as bytes.
- `AS score` names the distance score.
- `.sort_by("score")` orders results by the returned score.

The query and stored vectors must both contain **8 numeric values** because that is the index dimension configured by the starter data. The function converts byte-valued result fields to strings for display.

Save `vector_functions.py` and verify the code is in the correct sections. Keep all existing imports, class code, constants, and helper functions.

---

## Task 6 — Finish deployment, create the database, and load environment variables

Return to the terminal running `python azdeploy.py`. After Azure Managed Redis has been created successfully, press **Enter** to return to the menu.

Run these options in order:

1. Enter **`2`** — **Create database and configure access**. This creates a database with the RediSearch module, assigns a Microsoft Entra ID data access policy to your account, and creates `.env` and `.env.ps1` containing the `REDIS_HOST` endpoint.
2. Enter **`3`** — **Check deployment status**. Confirm the cluster and database are ready.
3. Enter **`4`** — **Exit**.

From the project root, load the generated PowerShell environment variables:

```powershell
. .\.env.ps1
```

Keep this terminal session open. Environment variables loaded into one terminal do not automatically appear in a new terminal.

---

## Task 7 — Configure the Python environment

From the project root, change to the `client` directory:

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

If PowerShell blocks activation because script execution is disabled, allow it for the current process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Then retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the application dependencies:

```powershell
pip install -r requirements.txt
```

Check the installed packages if needed:

```powershell
pip list
```

If this terminal was opened after the environment variables were loaded in another terminal, load them from the `client` directory:

```powershell
. ..\.env.ps1
```

The prompt should display `(.venv)` when the virtual environment is active.

---

## Task 8 — Start the Flask application

From the `client` directory, with the virtual environment active and environment variables loaded, run:

```powershell
python app.py
```

Leave the app running and open a browser at:

[http://localhost:5000](http://localhost:5000)

The page includes data operations, similarity search, storing products, and removing products.

---

## Task 9 — Load products and perform the first similarity search

1. In **Data Operations**, select **Load Sample Products**.
2. Select **List All Products** and confirm that product keys appear in **Operation Results**.
3. In **Similarity Search**, enter `product:001`.
4. Leave `top_k` set to `5` and select **Find Similar**.
5. Review the product matches and distance scores in **Operation Results**.

The application stores sample embeddings and metadata in Redis, then queries the vector index to return the nearest neighbors.

If no search results appear, load the sample products first and verify the key exists using **List All Products**.

---

## Task 10 — Store a new product and search again

In **Store Product**, enter these exact values.

**Product key**

```text
product:011
```

**Embedding**

```text
[0.53, 0.63, 0.58, 0.37, 0.68, 0.47, 0.73, 0.57]
```

**Metadata fields**

```text
product_id=011
name=Gym Bag
category=Sports
```

1. Select **Store Product**.
2. In **Similarity Search**, enter `product:009`.
3. Select **Find Similar**.
4. Review the returned neighbors. Confirm the ordering reflects the newly stored `product:011` vector and metadata where relevant.

This demonstrates how adding a new product vector changes the results returned by similarity search.

---

## Task 11 — Remove the new product and verify

1. In **Remove Product**, enter `product:011`.
2. Select **Remove**.
3. Select **List All Products** again.
4. Confirm `product:011` no longer appears.

This verifies that the newly added product can be removed from the Redis dataset.

When finished, return to the PowerShell terminal running Flask and press **Ctrl+C** to stop the application.

---

## Task 12 — Clean up Azure resources

Delete the dedicated resource group configured in `azdeploy.py`. Replace `<rg-name>` with the exact resource group name:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting the resource group deletes every resource contained within it. Use a dedicated group for this exercise; unrelated resources in an existing resource group will also be deleted.

---

## Troubleshooting

### Verify Azure Managed Redis deployment

- In the Azure portal, open the resource group and confirm the Redis resource has **Provisioning State: Succeeded**.
- Run the deployment script's **Check deployment status** option and confirm the cluster and database are ready before starting the app.
- If deployment status is not `Succeeded`, wait and check it again.

### Resolve deployment failures

- Temporary regional capacity limits can cause deployment failures.
- Exit the script and change the Azure region variable near the top of `azdeploy.py` to another available region, such as `eastus2`, `australiaeast`, or `canadacentral`.
- Save the file, rerun `python azdeploy.py`, and choose option **1**. The script automatically deletes the failed resource before another attempt.

### Check authentication and access

Verify Azure CLI sign-in:

```powershell
az account show
```

- Ensure deployment option **2** completed successfully so your account has a Redis data access policy.
- If the app reports an authentication error just after the policy assignment, wait briefly and retry.

### Check code completeness and indentation

- Confirm every block is in the correct section of `client/vector_functions.py`.
- Use consistent spaces for indentation.
- Do not edit `client/app.py`; the starter Flask application is already wired to call these functions.
- Preserve the class methods' indentation and the existing imports/constants.

### Verify environment variables

- Confirm `.env` and `.env.ps1` exist in the project root and contain `REDIS_HOST`.
- From the project root, run `. .\.env.ps1`.
- From the `client` directory, run `. ..\.env.ps1`.
- Each new terminal session needs the environment variables loaded separately.

### Check Python environment and dependencies

- Ensure the virtual environment is active before running Flask.
- Install dependencies from `client` using `pip install -r requirements.txt`.
- Run `pip list` to verify installation.

### No search results or missing products

- Load sample products before starting similarity searches.
- Select **List All Products** and confirm keys such as `product:001` appear.
- Ensure stored embeddings and query embeddings contain exactly 8 numeric values, matching `VECTOR_DIM`.
- Ensure the index creation step completed successfully.

### Resource capacity unavailable

If Azure reports the chosen SKU cannot be deployed in the selected region, change the region variable as described above and retry. Do not repeatedly retry the same unavailable region.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure sign-in completed and `Microsoft.Cache` registered.
- [ ] `redisenterprise` extension installed or upgraded to version 2.75.0 or later.
- [ ] Azure Managed Redis resource deployed successfully.
- [ ] `get_client()` implemented with Microsoft Entra ID, TLS, and port 10000.
- [ ] HNSW RediSearch vector index implemented.
- [ ] Product hash storage function implemented.
- [ ] KNN similarity search function implemented.
- [ ] Database and RediSearch module configured; `.env` and `.env.ps1` created.
- [ ] Python virtual environment activated and dependencies installed.
- [ ] Sample products loaded and listed.
- [ ] Similarity search tested using `product:001` with `top_k = 5`.
- [ ] `product:011` stored with the exact Gym Bag vector and metadata.
- [ ] Search repeated for `product:009`.
- [ ] `product:011` removed and absence verified.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- Azure Managed Redis can store product metadata and vector embeddings in Redis hashes.
- RediSearch supports an HNSW index and KNN search with cosine distance.
- The vector index dimension must match both stored and query embeddings; this lab uses 8 dimensions.
- Embeddings are converted to `float32` bytes before being stored in Redis.
- Microsoft Entra ID authentication avoids handling a Redis access key in application code.
- Adding or removing product hashes changes which products are available to similarity search.
