# Lab 13 — Implement Vector Search with Azure Database for PostgreSQL

## Lab objective

Build a product-similarity search application using Azure Database for PostgreSQL Flexible Server and the `pgvector` extension. You will deploy PostgreSQL with Microsoft Entra authentication, complete a Flask application, enable `pgvector`, create a table for products and vector embeddings, add an HNSW index, load sample products, and use cosine distance to find similar products.

**Estimated duration:** 30 minutes.

## Prerequisites

- An Azure subscription with permissions and quota to create the required resources.
- Visual Studio Code (VS Code).
- The latest Azure CLI.
- Python 3.12 or later.
- PostgreSQL command-line tools (`psql`).
- Optional: the Ruff extension for VS Code for formatting and linting Python code.

**Command convention:** All terminal commands are single-line Windows PowerShell commands. SQL statements and `psql` meta-commands are also provided one line at a time. The Flask route implementations are multiline Python code to paste into `client/app.py`; they are not shell commands.

---

## Task 1 — Download and configure the starter project

1. Download the official starter ZIP file:

   [Download postgresql-vector-search-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/postgresql-vector-search-python.zip)

2. Move the ZIP file to your working folder and extract it.
3. Open VS Code. Select **File > Open Folder...** and open the extracted project folder.
4. Open `azdeploy.py`.
5. Change **only** the two values at the top of the script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name for this lab.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save `azdeploy.py`. Do not change anything else in the deployment script.
7. Select **Terminal > New Terminal** in VS Code. Use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Follow the browser prompts and select the correct Azure account and subscription.

Verify the active subscription:

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

Wait for provider registration before continuing.

## Task 2 — Deploy the PostgreSQL server

From the project root, run:

```powershell
python azdeploy.py
```

When the deployment menu appears, enter **`1`** to choose **Create PostgreSQL server with Entra authentication**. This deploys Azure Database for PostgreSQL Flexible Server with Microsoft Entra-only authentication enabled. Deployment can take **5–10 minutes**.

**Important:** Leave the deployment terminal open while the server is being created. Complete the Flask route code in Task 3 while deployment continues.

---

## Task 3 — Complete the Flask application code

Open `client/app.py` in VS Code. Add each route immediately after its corresponding `BEGIN ... SECTION` comment and before the next marker. Do not replace the other starter code.

**Indentation tip:** Paste each block flush with the left margin (column 1), select all pasted lines, and press **Tab** to align the block with the existing `BEGIN / END` markers. Use **Shift+Tab** to outdent if needed.

### 3.1 Add the route that loads sample products

Find `BEGIN LOAD DATA SECTION` and add:

```python
@app.route("/load-data", methods=["POST"])
def load_data():
    """Load sample products into the database."""
    try:
        products = load_json_file("sample_products.json")

        with get_connection() as conn:
            with conn.cursor() as cur:
                for product in products:
                    # Check if product already exists.
                    cur.execute("SELECT id FROM products WHERE name = %s", (product["name"],))
                    if cur.fetchone():
                        continue

                    # Format the embedding as a pgvector bracketed array.
                    embedding_str = "[" + ",".join(str(x) for x in product["embedding"]) + "]"
                    cur.execute("""
                        INSERT INTO products (name, category, description, price, embedding)
                        VALUES (%s, %s, %s, %s, %s)
                    """, (
                        product["name"],
                        product["category"],
                        product["description"],
                        product["price"],
                        embedding_str
                    ))
                # Commit all inserts in one transaction.
                conn.commit()

        flash(f"Successfully loaded {len(products)} sample products!", "success")
    except Exception as e:
        flash(f"Error loading data: {str(e)}", "error")

    return redirect(url_for("index"))
```

This route loads records from `sample_products.json` and saves each product's pre-computed embedding. It skips products whose names are already in the database.

### 3.2 Add the route that searches for similar products

Find `BEGIN SEARCH SECTION` and add:

```python
@app.route("/search", methods=["POST"])
def search():
    """Find products similar to the selected product using vector similarity."""
    product_id = request.form.get("product_id")
    if not product_id:
        flash("Please select a product", "error")
        return redirect(url_for("index"))

    try:
        with get_connection() as conn:
            with conn.cursor() as cur:
                # Get the selected product and its embedding.
                cur.execute("""
                    SELECT id, name, category, description, price, embedding
                    FROM products WHERE id = %s
                """, (product_id,))
                row = cur.fetchone()

                if not row:
                    flash("Product not found", "error")
                    return redirect(url_for("index"))

                searched_product = {
                    "id": row[0], "name": row[1], "category": row[2],
                    "description": row[3], "price": row[4]
                }
                embedding = row[5]

                # Use cosine distance. Lower distance means more similar.
                cur.execute("""
                    SELECT id, name, category, description, price, embedding <=> %s AS distance
                    FROM products
                    WHERE id != %s
                    ORDER BY distance
                    LIMIT 5
                """, (embedding, product_id))
                results = [
                    {"id": r[0], "name": r[1], "category": r[2], "description": r[3], "price": r[4], "distance": r[5]}
                    for r in cur.fetchall()
                ]

        products = get_products()
        new_products = get_new_products()
        return render_template("index.html", products=products, new_products=new_products, results=results, searched_product=searched_product)

    except Exception as e:
        flash(f"Error searching: {str(e)}", "error")
        return redirect(url_for("index"))
```

The `<=>` operator is pgvector's cosine-distance operator. A lower distance means closer vectors; `0` indicates identical vectors, while values near `2` indicate opposite directions. The route excludes the selected product itself and returns the five closest products.

### 3.3 Add the route that inserts a new product

Find `BEGIN ADD PRODUCT SECTION` and add:

```python
@app.route("/add-product", methods=["POST"])
def add_product():
    """Add a new product from the new_products.json file."""
    product_index = request.form.get("product_index")

    if product_index is None or product_index == "":
        flash("Please select a product to add", "error")
        return redirect(url_for("index"))

    try:
        new_products = load_json_file("new_products.json")
        product = new_products[int(product_index)]

        with get_connection() as conn:
            with conn.cursor() as cur:
                # Check if product already exists.
                cur.execute("SELECT id FROM products WHERE name = %s", (product["name"],))
                if cur.fetchone():
                    flash(f"Product '{product['name']}' already exists", "error")
                    return redirect(url_for("index"))

                # Format the embedding as a pgvector array.
                embedding_str = "[" + ",".join(str(x) for x in product["embedding"]) + "]"
                cur.execute("""
                    INSERT INTO products (name, category, description, price, embedding)
                    VALUES (%s, %s, %s, %s, %s)
                """, (
                    product["name"],
                    product["category"],
                    product["description"],
                    product["price"],
                    embedding_str
                ))
                conn.commit()

        flash(f"Successfully added '{product['name']}'!", "success")
    except Exception as e:
        flash(f"Error adding data: {str(e)}", "error")
    return redirect(url_for("index"))
```

This route inserts a product selected from `new_products.json`. It prevents duplicate product names and stores the product's embedding with its other details.

### 3.4 Review and save the application

1. Save `client/app.py`.
2. Review the full file. The starter application already contains the imports, connection helpers, product-query helpers, and routes' surrounding infrastructure.
3. Confirm the three route blocks are in the correct locations and indentation matches the surrounding code.
4. Note that the application uses the existing `get_connection()` function to connect to PostgreSQL using Microsoft Entra authentication.

---

## Task 4 — Finish deployment and configure the pgvector extension

Wait until the deployment terminal reports that PostgreSQL deployment is complete. Then use the deployment menu in this order:

1. Enter **`2`** — **Configure vector extension allow-list**. This adds `vector` to the server's `azure.extensions` allow-list. The server restarts to apply the setting; this can take **1–2 minutes**.
2. Enter **`3`** — **Check deployment status**. Verify the server is ready.
3. Enter **`4`** — **Retrieve connection info and access token**. This creates the environment-variable files.
4. Enter **`5`** — **Exit**.

From the project root, load the variables into the current PowerShell session:

```powershell
. .\.env.ps1
```

Keep this terminal open. Environment variables loaded in one terminal do not automatically appear in another terminal.

Connect to PostgreSQL using the connection variables:

```powershell
psql "host=$env:DB_HOST port=5432 dbname=$env:DB_NAME user=$env:DB_USER sslmode=require"
```

If the connection fails, review the Troubleshooting section before proceeding.

---

## Task 5 — Enable pgvector and create the products schema

Run the following SQL statements individually at the `psql` prompt.

### 5.1 Enable the pgvector extension

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The extension adds the `vector` data type and similarity-search operators. It must be enabled in the database before creating or querying vector columns.

### 5.2 Create the products table

The table stores the product's name, category, description, price, and embedding. **Important correction:** The official page describes the sample embeddings as **384-dimensional**, but its displayed table statement says `vector(330)`. Use `vector(384)` so the column matches the documented sample embedding size.

```sql
CREATE TABLE products (id SERIAL PRIMARY KEY, name TEXT NOT NULL, category TEXT, description TEXT, price NUMERIC(10, 2), embedding vector(384));
```

If the `products` table already exists because you previously ran the lab, do not create it again. Confirm its structure before proceeding.

### 5.3 Create an HNSW vector index

```sql
CREATE INDEX products_embedding_idx ON products USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
```

This index supports approximate nearest-neighbor search using cosine distance.

- `m = 16` controls the number of connections created per layer in the HNSW graph.
- `ef_construction = 64` affects the quality and build cost of the graph during index creation.
- `vector_cosine_ops` matches the cosine-distance operator used by the application's `<=>` query.

### 5.4 Verify the table and index

At the `psql` prompt, inspect the table:

```text
\d products
```

Confirm the columns include `id`, `name`, `category`, `description`, `price`, and `embedding`, with the embedding column using `vector(384)`. Verify the HNSW index is listed.

Exit `psql`:

```text
\q
```

---

## Task 6 — Set up the Python environment

From the project root, move to the client directory:

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

If PowerShell blocks script execution, allow activation for the current process only:

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

The requirements include Flask, Psycopg for PostgreSQL connectivity, and Azure Identity for Microsoft Entra authentication.

Ensure the environment variables are loaded in the terminal that will run Flask. If needed, from the `client` directory run:

```powershell
. ..\.env.ps1
```

The PowerShell prompt should show `(.venv)` once the virtual environment is active.

---

## Task 7 — Start the Flask application

From the `client` directory, with the virtual environment active and environment variables loaded, run:

```powershell
python app.py
```

Leave the app running. Open a browser and navigate to:

[http://127.0.0.1:5000](http://127.0.0.1:5000)

You should see the **Vector Search Demo** page with an empty product list on the first run.

---

## Task 8 — Load sample products

1. In the Flask page, select **Load Sample Products**.
2. Wait for the operation to complete.
3. Verify the success message appears and the product list includes examples such as:
   - Wireless Bluetooth Headphones
   - Gaming Laptop
   - Running Shoes

The sample JSON contains **10 products** with pre-computed embeddings. The route inserts them into the PostgreSQL table.

If products do not appear, check the Flask terminal for errors and confirm `client/sample_products.json` exists.

## Task 9 — Perform product-similarity searches

1. In **Find Similar Products**, select **Wireless Bluetooth Headphones**.
2. Select **Find Similar**.
3. Review the returned products and distance scores. **Noise Cancelling Earbuds** should appear near the top because both products are audio devices.
4. Try selecting other products. Observe how the closest results change with the selected product's category and description.

The query uses the stored vector and the `<=>` cosine-distance operator. Results are sorted by distance in ascending order, so smaller distances appear first.

## Task 10 — Add new products and observe how results change

1. In **Add New Product**, select **Espresso Machine**.
2. Select **Add Product** and wait for the success message.
3. In **Find Similar Products**, select **Coffee Maker**.
4. Select **Find Similar**. Notice that **Espresso Machine** now appears among the similar products with a low distance score because both products are home coffee appliances.
5. Return to **Add New Product**, add **Wireless Gaming Mouse**, and observe how it appears in searches for related products.
6. Add **Fitness Tracker Band** and test searches for related products.

These steps show how inserting a new product and its embedding changes the search space without having to rebuild the entire application.

After testing, stop Flask by returning to the PowerShell terminal running the app and pressing **Ctrl+C**.

---

## Task 11 — Clean up Azure resources

Delete the dedicated resource group created for this lab. Replace `<rg-name>` with the resource group name configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes all resources inside it. Use a dedicated resource group for the lab. If you used an existing group containing unrelated resources, those resources will also be deleted.

---

## Troubleshooting

### Deployment fails

- Run `python azdeploy.py` again and select option **1**.
- If the script detects an existing PostgreSQL server, it warns that redeployment permanently deletes the server and its data. Confirm only if you intend to delete and recreate the server.
- After a new deployment, choose option **2** to add `vector` to the server's extension allow-list.
- If Azure has not released the PostgreSQL server name within five minutes, exit the deployment script, wait five minutes, and retry option **1**.

### `psql` connection fails

- Ensure the deployment script created both `.env` and `.env.ps1` by choosing option **4**.
- From the project root, load variables with `. .\.env.ps1`.
- Confirm the server is ready by running `python azdeploy.py` and selecting option **3**.
- The access token expires after approximately one hour. Select option **4** to retrieve a fresh token and reload `. .\.env.ps1`.

### Access denied or authentication errors

- The Microsoft Entra administrator is configured when option **1** creates the server. Verify configuration with option **3**.
- If the administrator is missing, the lab advises redeploying with option **1**. Read the warning carefully because redeployment deletes the server and its data.
- Verify `PGPASSWORD` is set correctly in the current PowerShell session.
- Confirm `DB_USER` is the email address associated with the Azure account used for authentication.

### Flask application fails to start

- Ensure the virtual environment is active; the terminal prompt should show `(.venv)`.
- From `client`, run `pip install -r requirements.txt`.
- Verify all three route functions were added to `app.py` under the correct markers.
- Ensure environment variables are loaded in the same terminal that runs Flask.

### Database connection errors in Flask

- Connect with `psql` and run `\d products` to verify the table exists.
- Ensure `CREATE EXTENSION IF NOT EXISTS vector;` completed successfully.
- Check that the app's environment variables point to the same database where you created the table.
- Ensure the embedding column dimensions match the sample JSON vectors.

### Table or vector-dimension errors

- Confirm the table schema uses `embedding vector(384)`, matching the documented sample embedding size.
- The page's original SQL snippet contains `vector(330)`, which conflicts with its prose stating 384 dimensions. Do not use 330 for this sample dataset.
- If you already created a `vector(330)` table and have not loaded data, recreate the table with the corrected schema. Only drop a table if you are certain its data can be deleted.

### No products appear after loading

- Check the Flask terminal for error messages.
- Confirm `client/sample_products.json` exists.
- Verify the `products` table has the expected schema and that the Flask app is connected to the same database.
- Try the **Load Sample Products** action again after fixing the underlying error; the handler skips products whose names already exist.

### Virtual environment activation fails

- From the `client` directory, activate with `.\.venv\Scripts\Activate.ps1`.
- If PowerShell blocks activation, run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass` and retry.
- If the activation script is missing, recreate the environment using `python -m venv .venv` from `client`.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure login completed and `Microsoft.DBforPostgreSQL` registered.
- [ ] PostgreSQL Flexible Server deployed with Entra-only authentication.
- [ ] All three Flask route functions added to `client/app.py`.
- [ ] Extension allow-list configured and deployment status verified.
- [ ] Environment variables and access token loaded into PowerShell.
- [ ] Connected to PostgreSQL with `psql`.
- [ ] `vector` extension enabled.
- [ ] `products` table created with `embedding vector(384)`.
- [ ] HNSW index created and verified using `\d products`.
- [ ] Python virtual environment activated and dependencies installed.
- [ ] Flask application started and 10 sample products loaded.
- [ ] Similar-product search tested with multiple products.
- [ ] Espresso Machine, Wireless Gaming Mouse, and Fitness Tracker Band added and search results rechecked.
- [ ] Flask stopped and Azure resources deleted when no longer needed.

## Key takeaways

- `pgvector` adds vector storage and similarity-search operators to PostgreSQL.
- The `<=>` operator measures cosine distance; smaller distances indicate more similar vectors.
- The vector column's dimensions must match the embedding array length.
- An HNSW index supports efficient approximate nearest-neighbor search using the configured distance operator class.
- Pre-computed embeddings let the application find semantically similar products, even if their product names do not share exact keywords.
