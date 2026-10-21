# Lab 14 — Optimize Vector Search Performance in Azure Database for PostgreSQL

## Lab objective

Optimize vector search on Azure Database for PostgreSQL Flexible Server. You will create 100,000 products with 384-dimensional embeddings, measure a baseline query without vector indexes, build and compare IVFFlat and HNSW indexes, tune their search parameters, and add B-tree indexes to improve metadata-filtered vector queries.

**Estimated duration:** 30 minutes.

## Prerequisites

- An Azure subscription with permissions and quota to provision the required services.
- Visual Studio Code (VS Code).
- The latest Azure CLI.
- Python 3.12 or later.
- PostgreSQL command-line tools (`psql`).

**Command conventions:** Shell commands in this guide use Windows PowerShell and are each one line. SQL statements are also written one line at a time for execution at the `psql` prompt. Do not paste SQL statements into PowerShell; first connect to PostgreSQL with `psql`.

---

## Task 1 — Download and configure the starter project

1. Download the official starter ZIP file:

   [Download postgresql-optimize-vector-search-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/postgresql-optimize-vector-search-python.zip)

2. Move the ZIP file to a working folder and extract it.
3. Open VS Code. Select **File > Open Folder...** and open the extracted project folder.
4. Open `azdeploy.py`.
5. Update **only** the two values at the top of the script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name for this lab.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save the file. Do not change anything else in `azdeploy.py`.
7. In VS Code, select **Terminal > New Terminal**. Use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Verify the selected subscription:

```powershell
az account show --output table
```

If necessary, select the intended subscription:

```powershell
az account set --subscription "<subscription-id-or-name>"
```

Register the Azure Database for PostgreSQL resource provider:

```powershell
az provider register --namespace Microsoft.DBforPostgreSQL
```

Wait for registration to complete.

## Task 2 — Deploy PostgreSQL Flexible Server

From the project root, start the deployment script:

```powershell
python azdeploy.py
```

When the menu appears, enter **`1`** for **Create PostgreSQL server with Entra authentication**. This creates the server with Microsoft Entra-only authentication enabled. Deployment can take **5–10 minutes**.

**Keep this deployment terminal open.** You can review the vector-index concepts in Task 3 while the server is created.

---

## Task 3 — Understand the vector index types and tuning parameters

### IVFFlat

IVFFlat (Inverted File with Flat compression) groups vectors into clusters called *lists*. At query time, PostgreSQL searches a subset of clusters rather than scanning the full dataset.

- **`lists`** controls the number of clusters. For up to one million rows, a starting guideline is approximately `rows / 1000`. More lists can reduce the number of vectors scanned but increase index-build cost.
- **`ivfflat.probes`** controls how many clusters are searched per query. A low value is faster but can miss nearest neighbors; a higher value generally improves recall at the cost of latency.

### HNSW

HNSW (Hierarchical Navigable Small World) builds a multi-layer graph. Higher layers support fast navigation, while lower layers allow more precise searches.

- **`m`** controls the maximum connections per graph node. Higher values can improve recall but use more memory and take longer to build. Default: `16`.
- **`ef_construction`** controls the candidate list size during graph construction. Higher values can improve index quality but increase build time. Default: `64`.
- **`hnsw.ef_search`** controls the candidate list size at query time. Higher values can improve recall but increase query latency. Default: `40`.

### General comparison

| Consideration | IVFFlat | HNSW |
|---|---|---|
| Build time | Faster | Slower |
| Query speed | Fast | Often faster |
| Memory use | Lower | Higher |
| Recall | Good with tuning | Usually strong out of the box |
| Updates | May require index rebuilding/retraining considerations | Supports incremental updates |

You will measure the actual performance on the same 100,000-row dataset rather than relying only on these general expectations.

---

## Task 4 — Finish deployment and load connection details

After the **Create PostgreSQL server with Entra authentication** operation finishes, return to the deployment terminal and select the following options in order:

1. Enter **`2`** — **Configure vector extension allow-list**. This adds `vector` to the server's `azure.extensions` allow-list. The server restarts to apply the change; this may take **1–2 minutes**.
2. Enter **`3`** — **Check deployment status**. Confirm the server is ready.
3. Enter **`4`** — **Retrieve connection info and access token**. This creates the environment-variable files.
4. Enter **`5`** — **Exit**.

From the project root, load the variables into the current PowerShell session:

```powershell
. .\.env.ps1
```

Keep the terminal open. A new terminal does not automatically inherit these variables. The access token expires after approximately **one hour**. To reconnect later, run `python azdeploy.py`, select option **4** to generate a fresh token, and reload the variables using `. .\.env.ps1`.

---

## Task 5 — Connect to PostgreSQL and create the test dataset

From the project root, connect to the server using PowerShell environment variables:

```powershell
psql "host=$env:DB_HOST port=5432 dbname=$env:DB_NAME user=$env:DB_USER sslmode=require"
```

The deployment configures `PGPASSWORD` for authentication. At the `psql` prompt, perform the following database steps. Run each SQL statement separately.

**Important:** Keep this `psql` session open throughout Tasks 5–9. The `query_vectors` table created in Task 5 is temporary and only exists in this database session.

If results open in a pager, press **q** to return to the `psql` prompt.

### 5.1 Enable pgvector

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The `vector` extension supplies the `vector` data type and operators such as `<=>` for cosine distance. Azure Database for PostgreSQL includes pgvector support, but the extension must be explicitly enabled in each database where it is used.

### 5.2 Create the products table

```sql
CREATE TABLE products (id BIGSERIAL PRIMARY KEY, name TEXT NOT NULL, category_id INTEGER NOT NULL, price NUMERIC(10,2) NOT NULL, in_stock BOOLEAN DEFAULT true, embedding vector(384));
```

The `embedding` column stores 384-dimensional vectors. The other columns provide product name, category, price, and stock status for filtered-search exercises.

If the table already exists from an earlier attempt, do not create it again without checking its contents. For a fresh lab database, the command should succeed.

### 5.3 Generate 100,000 products with random vectors

```sql
INSERT INTO products (name, category_id, price, in_stock, embedding) SELECT 'Product ' || i, (random() * 20)::int + 1, (random() * 1000)::numeric(10,2), random() > 0.1, ('[' || array_to_string(ARRAY(SELECT (random() * 2 - 1)::float4 FROM generate_series(1, 384)), ',') || ']')::vector FROM generate_series(1, 100000) AS i;
```

This creates **100,000 rows**, each with a random 384-dimensional embedding. The categories range from 1 through 21, prices are randomized, and approximately 90% of rows are marked in stock. Because the vectors are random, this exercise measures index and query behavior rather than semantic quality.

### 5.4 Verify the dataset size

```sql
SELECT COUNT(*) FROM products;
```

Confirm that the count is **100000**.

### 5.5 Create a stable query vector

```sql
CREATE TEMP TABLE query_vectors AS SELECT ('[' || array_to_string(ARRAY(SELECT (random() * 2 - 1)::float4 FROM generate_series(1, 384)), ',') || ']')::vector AS embedding;
```

This temporary table contains one randomly generated query vector. Reusing it makes the query vector consistent throughout this `psql` session, allowing fair comparisons between index settings.

---

## Task 6 — Measure baseline performance without an index

Run this query at the `psql` prompt:

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Review the execution plan. Without a vector index, you should see a **Sequential Scan** and the query must consider the 100,000 rows to find the closest results.

1. Record the `Execution Time` reported at the bottom of the plan.
2. Run the same query two more times.
3. Use the later runs as the baseline because the first execution may be slower while caches are cold.
4. Record the execution time and scan type in your results table.

### Results log

Keep a record of the measurements. Your times and planner output will depend on server capacity and caching.

| Configuration | Key settings | Execution time | Scan/index shown | Notes |
|---|---|---:|---|---|
| Baseline | No vector index |  | Sequential Scan | Run three times |
| IVFFlat | `lists=100`, `probes=1` |  |  | Lower recall risk |
| IVFFlat | `lists=100`, `probes=50` |  |  | Higher recall, usually slower |
| HNSW | `m=16`, `ef_construction=64`, default `ef_search` |  |  | Record build time too |
| HNSW | `ef_search=20` |  |  | Faster, may miss neighbors |
| HNSW | `ef_search=100` |  |  | Higher recall, may be slower |

---

## Task 7 — Create and tune an IVFFlat index

### 7.1 Build the IVFFlat index

With 100,000 rows, the lab uses `lists = 100` as a starting point.

```sql
CREATE INDEX idx_products_embedding_ivfflat ON products USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

Note the time required to build the index.

### 7.2 Run the same query with IVFFlat

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Confirm the plan uses `idx_products_embedding_ivfflat` when the planner chooses the index. Record the execution time.

### 7.3 Test low probes

```sql
SET ivfflat.probes = 1;
```

Run the search again:

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

With `probes = 1`, the search checks only one cluster. It may be fast, but can miss some of the true nearest neighbors. Execution time alone does not measure recall.

### 7.4 Test higher probes

```sql
SET ivfflat.probes = 50;
```

Run the search again:

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Record the execution time. Searching more clusters generally improves recall while increasing latency.

---

## Task 8 — Create and tune an HNSW index

### 8.1 Remove the IVFFlat index

The exercise compares the index types independently. Drop the IVFFlat index first:

```sql
DROP INDEX idx_products_embedding_ivfflat;
```

### 8.2 Increase memory available for index creation

```sql
SET maintenance_work_mem = '256MB';
```

HNSW typically needs more memory during index construction than IVFFlat.

### 8.3 Build the HNSW index

```sql
CREATE INDEX idx_products_embedding_hnsw ON products USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

Record the build time. HNSW generally takes longer to build than IVFFlat, but can provide fast queries and strong recall.

### 8.4 Test the default HNSW query

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Record the execution time and confirm whether the plan uses `idx_products_embedding_hnsw`.

### 8.5 Test lower `ef_search`

```sql
SET hnsw.ef_search = 20;
```

Run the query:

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Lower `ef_search` usually reduces query work, but it may miss true nearest neighbors.

### 8.6 Test higher `ef_search`

```sql
SET hnsw.ef_search = 100;
```

Run the query:

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Record the execution time. The larger candidate list generally improves recall but may increase latency.

### 8.7 Compare the measurements

Your exact timings depend on the server tier, cache state, and planner choices. Common observations are:

- A sequential scan is usually slowest on a large dataset because it processes all 100,000 rows.
- IVFFlat with `probes = 1` can be a fast indexed option, but it might miss nearest neighbors.
- HNSW often offers better latency at comparable recall, with higher memory and build-time costs.
- Increasing `ivfflat.probes` or `hnsw.ef_search` usually improves recall at the cost of more query work and latency.

**Important:** `EXPLAIN ANALYZE` displays query-plan and timing information, not recall quality directly. To measure recall, compare results from indexed searches with a trusted exact-search result set.

---

## Task 9 — Optimize queries with metadata filters

The vector search workload often includes structured filters such as category, price, or stock status. B-tree indexes can help PostgreSQL evaluate these conditions efficiently.

### 9.1 Create a B-tree index on category

```sql
CREATE INDEX idx_products_category ON products (category_id);
```

### 9.2 Run a vector query filtered by category

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products WHERE category_id = 5 ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Inspect the plan and record the execution time. Depending on the selectivity and planner decisions, PostgreSQL may use the HNSW index for vector ranking and apply the category filter, or select a different plan.

### 9.3 Add a more selective category-and-price filter

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products WHERE category_id = 5 AND price BETWEEN 100 AND 200 ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Compare the execution time and plan against the category-only query. Multiple conditions can cause PostgreSQL to use a bitmap index scan, post-filter the vector results, or choose another strategy.

### 9.4 Create a composite B-tree index

```sql
CREATE INDEX idx_products_category_price ON products (category_id, price);
```

This index is designed for queries that filter by `category_id` and then by a price range.

### 9.5 Re-run the more selective query

```sql
EXPLAIN ANALYZE SELECT id, name, embedding <=> (SELECT embedding FROM query_vectors) AS distance FROM products WHERE category_id = 5 AND price BETWEEN 100 AND 200 ORDER BY embedding <=> (SELECT embedding FROM query_vectors) LIMIT 10;
```

Compare the plan and execution time. The composite index may improve filtering before or alongside vector search, but the optimizer's choice depends on data distribution and selectivity; an index's existence does not guarantee the planner will use it.

---

## Task 10 — Finish and clean up

If you want to inspect the current indexes before ending the database session, run:

```text
\di
```

Exit `psql`:

```text
\q
```

Delete the dedicated Azure resource group from a PowerShell terminal. Replace `<rg-name>` with the exact resource group name configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes every resource inside it. Use a dedicated resource group for this exercise. If you used an existing resource group containing unrelated resources, those resources will also be deleted.

---

## Troubleshooting

### Deployment fails

- Run `python azdeploy.py` again and select option **1**.
- If the script detects an existing PostgreSQL server, it warns that redeployment permanently deletes the server and its data. Confirm only if you intend to delete and recreate the server.
- After server creation, select option **2** to add `vector` to the extension allow-list.
- If Azure has not released the server name within five minutes, exit the deployment script, wait five minutes, and retry option **1**.

### `psql` connection fails

- Make sure deployment option **4** created `.env` and `.env.ps1`.
- From the project root, load variables using `. .\.env.ps1`.
- Verify readiness by running `python azdeploy.py` and selecting option **3**.
- Access tokens expire after approximately one hour. Select option **4** to generate a fresh token, then reload `. .\.env.ps1`.
- Verify that `DB_HOST`, `DB_NAME`, and `DB_USER` are present in the current PowerShell session.

### Access denied or authentication errors

- The Entra administrator is configured when option **1** creates the server. Verify it with option **3**.
- If the administrator is missing, the lab suggests redeploying with option **1**. Read the warning carefully because redeployment deletes the server and its data.
- Ensure `PGPASSWORD` and `DB_USER` are correct for the current authentication session. `DB_USER` should be the Azure account email used for the exercise.

### `CREATE EXTENSION` fails

- Confirm you selected deployment option **2** to add the `vector` extension to the server's `azure.extensions` allow-list.
- Allow 1–2 minutes for the server restart to finish.
- Reconnect if necessary and run `CREATE EXTENSION IF NOT EXISTS vector;` again.

### Index creation is slow or fails

- HNSW takes longer to build than IVFFlat. For 100,000 vectors, allow approximately **1–2 minutes**, depending on server capacity.
- Check CPU and memory metrics in Azure Monitor if the build times out.
- Consider reducing the dataset size for a smaller test environment, while noting that results may differ from the 100,000-row exercise.

### A vector index is not used by the query plan

- Confirm the query's ordering expression matches the index operator class, for example `embedding <=> query_vector` with `vector_cosine_ops`.
- Review the `EXPLAIN ANALYZE` output rather than assuming an index is used.
- Planner decisions depend on cost estimates, data distribution, and filter selectivity.
- Ensure that the index is still present; the IVFFlat index is intentionally dropped before the HNSW test.

### Filtered vector query is slow

- Compare the category-only and category-plus-price query plans.
- Confirm `idx_products_category` and `idx_products_category_price` were created successfully.
- The planner might choose a bitmap scan, post-filtering, or another strategy. An index does not guarantee a speedup for every filter distribution.

### Results open in a pager

When `psql` displays output in a pager, press **q** to return to the SQL prompt. Maximizing the terminal can make query plans easier to review.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure sign-in completed and `Microsoft.DBforPostgreSQL` registered.
- [ ] PostgreSQL Flexible Server deployed with Entra-only authentication.
- [ ] Vector extension allow-list configured and server readiness verified.
- [ ] Environment variables loaded and `psql` connection established.
- [ ] `vector` extension enabled and `products` table created with `vector(384)`.
- [ ] 100,000 test rows inserted and count verified.
- [ ] Temporary `query_vectors` table created in the same `psql` session.
- [ ] Baseline query run three times and timing recorded.
- [ ] IVFFlat index tested with `probes = 1` and `probes = 50`.
- [ ] HNSW index tested at default settings, `ef_search = 20`, and `ef_search = 100`.
- [ ] Category B-tree and composite category/price indexes created.
- [ ] Filtered vector queries compared using `EXPLAIN ANALYZE`.
- [ ] `psql` exited and Azure resources deleted when no longer needed.

## Key takeaways

- Measure a baseline before optimizing; `EXPLAIN ANALYZE` reveals scans, index usage, and execution time.
- IVFFlat trades search speed against recall through the `lists` and `ivfflat.probes` settings.
- HNSW trades memory and build time against query quality and latency through `m`, `ef_construction`, and `hnsw.ef_search`.
- B-tree indexes can accelerate metadata filtering, while vector indexes support nearest-neighbor ranking.
- Always measure the actual query plan and workload; index selection depends on dataset size, filter selectivity, memory, latency, and recall requirements.
