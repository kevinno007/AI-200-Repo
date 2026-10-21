# Lab 15 — Perform Data Operations in Azure Managed Redis

## Lab objective

Create an Azure Managed Redis resource and complete a Python console application using `redis-py`. Connect with Microsoft Entra ID, store and retrieve Redis hashes, manage key expiration (TTL), and delete cache keys.

**Estimated duration:** 30 minutes.

## Prerequisites

- Azure subscription with sufficient permissions and quota.
- Visual Studio Code, Python 3.12 or later, and the latest Azure CLI.
- Azure CLI `redisenterprise` extension.
- Optional: Ruff extension for VS Code.

**Command format:** All terminal commands use Windows PowerShell and are single-line commands. The multiline Python blocks are code to paste into `main.py`, not shell commands.

---

## Task 1 — Download the starter project and deploy Azure Managed Redis

1. Download the official ZIP: [amr-data-operations-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/amr-data-operations-python.zip).
2. Extract it, open VS Code, and select **File > Open Folder...** to open the extracted project folder.
3. Open `azdeploy.py`. Change **only** the resource group name (`<your-resource-group-name>`) and Azure region (`<your-azure-region>`) at the top. Save the file.
4. Open **Terminal > New Terminal** in VS Code; use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Register the Azure Managed Redis resource provider:

```powershell
az provider register --namespace Microsoft.Cache
```

Install the Azure CLI extension:

```powershell
az extension add --name redisenterprise
```

Start the deployment script:

```powershell
python azdeploy.py
```

At the deployment menu, enter **`1`** for **Create Azure Managed Redis resource**. The script creates the resource group if needed and deploys Azure Managed Redis. Deployment usually takes **5–10 minutes**.

**Leave this terminal open.** Complete the code tasks while provisioning continues, and check the terminal periodically for deployment success or errors.

---

## Task 2 — Complete the client connection in `main.py`

Open `main.py` in VS Code. The file contains `BEGIN / END` comment markers. Paste each block after its matching marker and before the next marker. To align the code, paste flush with the left margin, select the block, and press **Tab**. Use **Shift+Tab** to outdent. Keep consistent Python spacing.

Find `# BEGIN CONNECTION CODE SECTION`. Add the following code within the existing connection function/`try` block; do not create a second connection function if the starter already provides one. Preserve the starter's exception handling.

```python
try:
    # Azure Managed Redis using Microsoft Entra ID authentication
    redis_host = os.getenv("REDIS_HOST")
    credential_provider = create_from_default_azure_credential(
        ("https://redis.azure.com/.default",),
    )
    r = redis.Redis(
        host=redis_host,
        port=10000,
        ssl=True,
        decode_responses=True,
        credential_provider=credential_provider,
        socket_timeout=30,
        socket_connect_timeout=30,
    )
    print(f"Connected to Redis at {redis_host}")
    input("\n Press Enter to continue...")
    return r
```

This connection reads `REDIS_HOST` from the environment, uses port `10000`, enables TLS, and authenticates via Microsoft Entra ID. The `redis-entraid` credential provider acquires and refreshes tokens.

## Task 3 — Add hash storage and retrieval

Find `# BEGIN STORE AND RETRIEVE CODE SECTION` and add:

```python
def store_hash_data(r, key, value) -> None:
    """Store hash data in Redis."""
    clear_screen()
    print(f"Storing hash data for key: {key}")
    result = r.hset(key, mapping=value)
    if result > 0:
        print(f"Data stored successfully under key '{key}' ({result} new fields added)")
    else:
        print(f"Data updated successfully under key '{key}' (all fields already existed)")
    input("\n Press Enter to continue...")

def retrieve_hash_data(r, key) -> None:
    """Retrieve hash data from Redis."""
    clear_screen()
    print(f"Retrieving hash data for key: {key}")
    retrieved_value = r.hgetall(key)
    if retrieved_value:
        print("\n Retrieved hash data:")
        for field, value in retrieved_value.items():
            print(f"  {field}: {value}")
    else:
        print(f"Key '{key}' does not exist.")
    input("\n Press Enter to continue...")
```

`HSET` stores multiple field/value pairs under one key; `HGETALL` retrieves all fields and values for that key.

## Task 4 — Add expiration and TTL functions

Find `# BEGIN EXPIRATION CODE SECTION` and add:

```python
def set_expiration(r, key) -> None:
    """Set an expiration time for a key."""
    clear_screen()
    print("Set expiration time for a key")
    expiration = int(input("Enter expiration time in seconds (default 3600): ") or 3600)
    result = r.expire(key, expiration)
    if result:
        print(f"Expiration time of {expiration} seconds set for key '{key}'")
    else:
        print(f"Key '{key}' does not exist. Expiration not set.")
    input("\n Press Enter to continue...")

def retrieve_expiration(r, key) -> None:
    """Retrieve the TTL of a key."""
    clear_screen()
    print(f"Retrieving the current TTL of {key}...")
    ttl = r.ttl(key)
    if ttl == -2:
        print(f"\n Key '{key}' does not exist.")
    elif ttl == -1:
        print(f"\n Key '{key}' has no expiration set (persists indefinitely).")
    else:
        print(f"\n Current TTL for '{key}': {ttl} seconds")
    input("\n Press Enter to continue...")
```

- `EXPIRE` sets a key's time-to-live in seconds. Pressing Enter at the prompt uses `3600` seconds (one hour).
- `TTL = -2` means the key does not exist.
- `TTL = -1` means the key exists without an expiration.
- A nonnegative TTL is the remaining lifetime in seconds.

## Task 5 — Add the delete function

Find `# BEGIN DELETE CODE SECTION` and add:

```python
def delete_key(r, key) -> None:
    """Delete a key and its associated value."""
    clear_screen()
    print(f"Deleting key: {key}...")
    result = r.delete(key)
    if result == 1:
        print(f"Key '{key}' deleted successfully.")
    else:
        print(f"Key '{key}' does not exist.")
    input("\n Press Enter to continue...")
```

Save `main.py`. Review the entire file and make sure the connection, hash, expiration, and delete blocks are in their intended sections. Do not alter the starter code outside the designated markers; it contains the menu, imports, mock data, and helper functions such as `clear_screen()`.

---

## Task 6 — Verify deployment and configure access

Return to the deployment terminal. When resource creation completes, press **Enter** to return to the menu. Choose these options in order:

1. Enter **`2`** — **Check deployment status**. Confirm the status is **Succeeded**. If not, wait a few minutes and retry option 2.
2. Enter **`3`** — **Create database and configure access**. This creates the database, assigns your account a Microsoft Entra ID data access policy, and creates `.env` and `.env.ps1` with `REDIS_HOST`.
3. Review the environment file and verify that `REDIS_HOST` has a value.
4. Enter **`4`** — **Exit**.

Access policy assignment may take a short time to propagate; wait briefly if the first connection fails.

## Task 7 — Configure the Python environment

From the project root, create a virtual environment:

```powershell
python -m venv .venv
```

Activate it:

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

Install dependencies:

```powershell
pip install -r requirements.txt
```

Verify the environment is active (`(.venv)` appears in the prompt) and packages are installed:

```powershell
pip list
```

## Task 8 — Load environment variables and start the app

From the project root, load the generated PowerShell environment variables:

```powershell
. .\.env.ps1
```

Keep the same terminal open. If you open a new terminal, reload the variables in that session. You can verify the endpoint variable without sharing credentials:

```powershell
$env:REDIS_HOST
```

Start the console app:

```powershell
python main.py
```

The menu should display:

1. Store hash data
2. Retrieve hash data
3. Set expiration
4. Retrieve expiration (TTL)
5. Delete key
6. Exit

The mock hash/key are defined near the beginning of `main()` in `main.py`. Use the key shown or specified by the starter code when following the prompts.

---

## Task 9 — Perform the Redis data operations

### 9.1 Store a hash

1. Choose **`1` — Store hash data**.
2. Follow the prompts and use the sample key/data defined in the starter.
3. Confirm that the app reports success, then press Enter to return to the menu.

`HSET` stores the field/value pairs under one key. The app reports how many new fields were added or that existing fields were updated.

### 9.2 Retrieve the hash

1. Choose **`2` — Retrieve hash data**.
2. Enter the same key used when storing the data.
3. Verify all fields and values are printed, then press Enter to return to the menu.

### 9.3 Check TTL before setting expiration

1. Choose **`4` — Retrieve expiration (TTL)**.
2. Use the same key.
3. Before setting an expiration, an existing key should show `TTL = -1`, meaning no expiration is set.

### 9.4 Set expiration

1. Choose **`3` — Set expiration**.
2. Use the same key and enter `60` seconds to test a short TTL, or press Enter to use `3600` seconds.
3. Confirm the app reports that the expiration was set.

The key must exist for `EXPIRE` to succeed.

### 9.5 Retrieve the remaining TTL

1. Choose **`4` — Retrieve expiration (TTL)** again.
2. Use the same key.
3. Confirm a nonnegative TTL is shown and is slightly lower than the number entered.

Run the TTL check promptly when testing 60 seconds. Once the key expires, it is deleted and TTL returns `-2`.

### 9.6 Delete and verify

1. If the key still exists, choose **`5` — Delete key** and provide the same key.
2. Confirm the deletion message.
3. Choose **`2` — Retrieve hash data** or **`4` — Retrieve expiration (TTL)** to verify the key no longer exists.
4. Choose **`6` — Exit** to close the app.

The options can also be explored in a different order. For example, immediately checking TTL after storing data demonstrates the `-1` case.

---

## Task 10 — Clean up Azure resources

Delete the dedicated resource group configured in `azdeploy.py`. Replace `<rg-name>` with the actual name:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting the resource group deletes every resource inside it. Use a dedicated resource group; unrelated resources inside an existing group will also be deleted.

---

## Troubleshooting

### Verify Azure Managed Redis deployment

- In the Azure portal, open the resource group and confirm the resource's **Provisioning State** is **Succeeded**.
- Confirm **Public network access** is enabled if connecting from your local computer.
- If status is not `Succeeded`, wait and run deployment menu option **2** again.

### Check code completeness and indentation

- Verify each block was added under the correct `BEGIN / END` marker in `main.py`.
- Use consistent spaces for Python indentation.
- Do not remove or modify code outside the designated sections.

### Verify environment variables

- Confirm `.env` and `.env.ps1` exist in the same project folder as `main.py`.
- Confirm `REDIS_HOST` is populated.
- Run `. .\.env.ps1` again after opening a new terminal.

### Check authentication and access

Verify Azure CLI sign-in:

```powershell
az account show
```

- Confirm deployment option **3** completed successfully so the current account has a data access policy.
- If access was just assigned, wait briefly and retry.

### Check Python dependencies

Confirm the virtual environment is active and packages were installed:

```powershell
pip list
```

If needed, reinstall them from the project folder:

```powershell
pip install -r requirements.txt
```

### Redis connection fails

- Confirm `$env:REDIS_HOST` is not empty.
- Ensure the connection uses TLS and port `10000`.
- Confirm deployment succeeded and the signed-in identity has a Redis data access policy.
- Wait briefly after access assignment, then restart `python main.py`.

### Unexpected key or TTL result

- Enter the exact same key that the sample code uses.
- `TTL = -2` means the key does not exist, perhaps because it expired or was deleted.
- `TTL = -1` means the key exists but has no expiration.
- A positive TTL counts down in seconds.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure sign-in completed and `Microsoft.Cache` registered.
- [ ] `redisenterprise` Azure CLI extension installed.
- [ ] Azure Managed Redis deployment succeeded.
- [ ] Entra-authenticated Redis connection added to `main.py`.
- [ ] Hash store/retrieve functions added.
- [ ] Expiration and TTL functions added.
- [ ] Delete function added.
- [ ] Database/access setup completed and `.env`/`.env.ps1` created.
- [ ] Python virtual environment activated and dependencies installed.
- [ ] Hash data stored and retrieved.
- [ ] TTL checked before and after setting expiration.
- [ ] Key deleted and nonexistence confirmed.
- [ ] Console app exited and Azure resources deleted.

## Key takeaways

- Redis hashes store multiple field/value pairs under one key; `HSET` writes them and `HGETALL` retrieves them.
- `EXPIRE` sets a key's time-to-live; `TTL` reports remaining seconds.
- `TTL = -1` means no expiration is set; `TTL = -2` means the key does not exist.
- `DEL` removes a key and its value.
- Azure Managed Redis supports Microsoft Entra authentication via `redis-entraid`, with TLS on port `10000`.
