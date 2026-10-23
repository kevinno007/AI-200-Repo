# Lab 21 — Manage Secrets with Azure Key Vault

## Lab objective

Deploy an RBAC-enabled Azure Key Vault with sample secrets, then complete a Python Flask application that retrieves secrets and metadata, lists secret properties without exposing values, creates a new secret version to simulate credential rotation, and implements a time-based cache to reduce Key Vault API calls.

**Estimated duration:** 20 minutes.

## Prerequisites

- Azure subscription with permissions and quota for the required resources.
- Visual Studio Code, Python 3.12 or later, and the latest Azure CLI.
- Optional: Ruff extension for VS Code.

**Command convention:** All terminal commands below are single-line Windows PowerShell commands. Python functions are multiline code to paste into `client/keyvault_functions.py`, not terminal commands.

---

## Task 1 — Download the starter project and deploy Azure Key Vault

1. Download the official ZIP: [key-vault-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/key-vault-python.zip).
2. Extract it, open VS Code, and select **File > Open Folder...** to open the extracted project folder.
3. Open `azdeploy.py`. Change **only** the resource group name (`<your-resource-group-name>`) and Azure region (`<your-azure-region>`) at the top of the script. Save it; do not change anything else.
4. Open **Terminal > New Terminal** in VS Code and use PowerShell in the project root.

Sign in to Azure:

```powershell
az login
```

Register the Key Vault provider:

```powershell
az provider register --namespace Microsoft.KeyVault
```

Start the deployment script:

```powershell
python azdeploy.py
```

At the menu, run these options in order:

1. Enter **`1`** — **Create Key Vault**. Creates the resource group if necessary and deploys Key Vault with **RBAC authorization enabled**.
2. Enter **`2`** — **Assign role**. Assigns **Key Vault Secrets Officer** to your account so you can read, create, update, and delete secrets using Microsoft Entra authentication.
3. Enter **`3`** — **Store secrets**. Stores two sample secrets: `openai-api-key` and `cosmosdb-connection-string`. Both have tags for environment/service identification.
4. Enter **`4`** — **Check deployment status**. Verify the vault status is `Succeeded`, the role is assigned, and both secrets are stored. Wait and check again if provisioning is still in progress.
5. Enter **`5`** — **Retrieve connection info**. Creates the environment-variable file with the Key Vault URL.
6. Enter **`6`** — **Exit**.

Load the environment variables in the current PowerShell session from the project root:

```powershell
. .\.env.ps1
```

Keep this terminal open. Variables do not automatically carry over to a new terminal; load them again in each new session.

---

## Task 2 — Implement secret retrieval

Open `client/keyvault_functions.py`. Add each function under its corresponding `BEGIN ... FUNCTION` marker and before the next marker. Preserve existing imports and the `get_client()` helper.

**Indentation tip:** Paste the code flush with the left margin (column 1), select the pasted block, and press **Tab** to align it with the existing markers. Use **Shift+Tab** to outdent if needed. Use spaces consistently.

Find `# BEGIN RETRIEVE SECRETS FUNCTION` and add:

```python
def retrieve_secrets():
    """Retrieve secrets and display their metadata."""
    client = get_client()
    results = []

    secret_names = ["openai-api-key", "cosmosdb-connection-string"]

    for name in secret_names:
        try:
            secret = client.get_secret(name)
            results.append({
                "name": secret.name,
                "value": secret.value[:20] + "..." if len(secret.value) > 20 else secret.value,
                "version": secret.properties.version,
                "content_type": secret.properties.content_type,
                "created_on": str(secret.properties.created_on),
                "tags": secret.properties.tags or {},
                "status": "retrieved"
            })
        except ResourceNotFoundError:
            results.append({
                "name": name,
                "value": None,
                "version": None,
                "content_type": None,
                "created_on": None,
                "tags": {},
                "status": "not found"
            })
        except HttpResponseError as e:
            results.append({
                "name": name,
                "value": None,
                "version": None,
                "content_type": None,
                "created_on": None,
                "tags": {},
                "status": f"error: {e.message}"
            })

    return results
```

The function retrieves secret values and metadata, including version, content type, creation date, and tags. It truncates values in the UI so full credentials are not exposed and handles missing secrets separately from authorization/network errors.

## Task 3 — List secret properties without reading values

Find `# BEGIN LIST SECRETS FUNCTION` and add:

```python
def list_secret_properties():
    """List all secret properties without retrieving values."""
    client = get_client()
    results = []

    # This lists metadata only, following the principle of least privilege.
    for prop in client.list_properties_of_secrets():
        results.append({
            "name": prop.name,
            "enabled": prop.enabled,
            "content_type": prop.content_type,
            "created_on": str(prop.created_on),
            "updated_on": str(prop.updated_on)
        })

    return results
```

This operation is useful for inventory and audit tasks. It returns names, enabled state, content type, creation time, and update time, without retrieving secret values.

## Task 4 — Create a new secret version (credential rotation)

Find `# BEGIN CREATE SECRET VERSION FUNCTION` and add:

```python
def create_secret_version(secret_name, new_value):
    """Create a new version of a secret and verify the update."""
    client = get_client()

    try:
        current = client.get_secret(secret_name)
        old_version = current.properties.version
        old_value = current.value[:20] + "..." if len(current.value) > 20 else current.value
    except ResourceNotFoundError:
        old_version = None
        old_value = None

    # set_secret creates a new version and preserves the prior version.
    client.set_secret(
        secret_name,
        new_value,
        content_type="text/plain",
        tags={"environment": "development", "rotated": "true"}
    )

    # Without a version parameter, get_secret returns the latest version.
    confirmed = client.get_secret(secret_name)

    return {
        "name": secret_name,
        "old_version": old_version,
        "old_value": old_value,
        "new_version": confirmed.properties.version,
        "new_value": confirmed.value[:20] + "..." if len(confirmed.value) > 20 else confirmed.value,
        "created_on": str(confirmed.properties.created_on),
        "tags": confirmed.properties.tags or {}
    }
```

Calling `set_secret()` with an existing name creates another version; it does not overwrite the previous version. The function compares the old and new version identifiers and attaches rotation tags to the new version.

## Task 5 — Implement cached secret retrieval

Find `# BEGIN CACHED RETRIEVAL FUNCTION` and add:

```python
def cached_retrieval():
    """Demonstrate time-based caching to reduce Key Vault API calls."""
    client = get_client()
    cache = {}
    cache_ttl = 30
    vault_calls = 0
    access_log = []
    secret_names = ["openai-api-key", "cosmosdb-connection-string"]

    for i in range(5):
        for name in secret_names:
            cached = cache.get(name)
            now = time.monotonic()

            if cached and (now - cached["timestamp"]) < cache_ttl:
                access_log.append({
                    "round": i + 1,
                    "secret": name,
                    "result": "cache hit",
                    "value": cached["value"]
                })
            else:
                secret = client.get_secret(name)
                vault_calls += 1
                truncated = secret.value[:20] + "..." if len(secret.value) > 20 else secret.value
                cache[name] = {"value": truncated, "timestamp": now}
                access_log.append({
                    "round": i + 1,
                    "secret": name,
                    "result": "cache miss",
                    "value": truncated
                })

    return {
        "access_log": access_log,
        "vault_calls": vault_calls,
        "total_accesses": len(access_log),
        "cache_ttl_seconds": cache_ttl
    }
```

The cache is an in-memory dictionary with a **30-second TTL** measured using `time.monotonic()`. The exercise performs five rounds for two secrets: normally the first access to each secret is a cache miss and the remaining accesses are cache hits. Expected summary: **10 accesses and 2 Key Vault API calls**, provided the TTL does not expire during the test.

Save `keyvault_functions.py`. Confirm all four functions are under the correct markers. Preserve the existing imports, especially `time`, `ResourceNotFoundError`, `HttpResponseError`, and the `get_client()` helper. Do not alter code outside the designated sections.

---

## Task 6 — Configure the Python environment

From the project root, enter the client folder:

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

If PowerShell blocks activation, allow it for the current process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install dependencies:

```powershell
pip install -r requirements.txt
```

Verify installed packages if needed:

```powershell
pip list
```

If this is a new terminal, load the environment variables from the `client` directory:

```powershell
. ..\.env.ps1
```

The prompt should include `(.venv)` when the environment is active.

## Task 7 — Run the Flask app and test the secret operations

From the `client` directory, run:

```powershell
python app.py
```

Open [http://localhost:5000](http://localhost:5000) in a browser.

### 7.1 Retrieve secrets

1. Select **Retrieve Secrets** in the left panel.
2. Verify `openai-api-key` and `cosmosdb-connection-string` both show status `retrieved`.
3. Inspect each secret's truncated value, version ID, content type, creation time, and tags.

Do not copy or expose full secret values.

### 7.2 List secret properties

1. Select **List Secret Properties**.
2. Verify the list displays each secret's name, enabled state, content type, creation time, and update time.
3. Confirm the operation does not display secret values.

### 7.3 Create a new API key version

1. Select **Create New Version**.
2. The app creates a new version of `openai-api-key` with a randomly generated value to simulate credential rotation.
3. Compare the old and new version IDs and truncated values. Check the rotation tags on the new version.

### 7.4 Verify rotation

1. Select **Retrieve Secrets** again.
2. Check that `openai-api-key` shows the new version and value preview.
3. Remember that the old version is preserved in Key Vault, while an unversioned `get_secret()` returns the newest version.

### 7.5 Run cached retrieval

1. Select **Run Cached Retrieval**.
2. Review all five rounds of accesses to both secrets.
3. Confirm the access log shows cache hits and misses.
4. Under normal timing, verify the summary shows **10 accesses** and **2 Key Vault API calls**; the cache TTL is 30 seconds.

Stop the Flask app with **Ctrl+C** in the terminal running it.

---

## Task 8 — Clean up Azure resources

Replace `<rg-name>` with the dedicated resource group name configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes every resource inside it. Use a dedicated resource group; unrelated resources in an existing group will also be deleted.

---

## Troubleshooting

### Verify Key Vault deployment

- In the Azure portal, open the resource group and confirm Key Vault **Provisioning State** is `Succeeded`.
- Confirm RBAC authorization is enabled, not legacy access-policy mode.
- Run `python azdeploy.py` and select option **4** to verify the secrets were stored.

### Secrets are missing

- Run `python azdeploy.py` and select option **3** to store the sample secrets again.
- Verify the exact names: `openai-api-key` and `cosmosdb-connection-string`.

### Key Vault access is denied

Check the Azure CLI sign-in:

```powershell
az account show
```

- Ensure option **2** assigned the **Key Vault Secrets Officer** role to the signed-in account.
- Role assignments can take time to propagate; wait briefly and retry.
- Make sure the app uses the same signed-in Azure identity.

### Code placement or indentation errors

- Verify each function is in `client/keyvault_functions.py` under the correct `BEGIN / END` marker.
- Use spaces consistently and preserve existing imports and `get_client()`.
- Do not modify code outside the designated sections.

### Environment variables are missing

- Confirm `.env` and `.env.ps1` exist in the project root and contain `KEY_VAULT_URL`.
- From the project root, run `. .\.env.ps1`.
- From `client`, run `. ..\.env.ps1`.
- A new terminal needs the environment variables loaded again.

### Python environment or dependency errors

- Confirm `(.venv)` appears in the terminal prompt.
- Install packages from `client` with `pip install -r requirements.txt`.
- Check packages with `pip list` and select the `.venv` interpreter in VS Code via **Python: Select Interpreter**.

### Cache makes more than two Key Vault calls

- The cache TTL is 30 seconds. If processing takes longer than that, an entry may expire and trigger another API request.
- Review the hit/miss log and repeat the test promptly.

### Secret values are exposed

- The sample UI truncates secret values. Do not add logging or result output that prints full API keys or connection strings.
- In production, do not store secrets in source control or write their values to application logs.

---

## Lab completion checklist

- [ ] Starter project extracted and `azdeploy.py` configured.
- [ ] Azure sign-in completed and `Microsoft.KeyVault` registered.
- [ ] RBAC-enabled Key Vault deployed.
- [ ] Key Vault Secrets Officer role assigned.
- [ ] Sample secrets stored and deployment status verified.
- [ ] Environment variables loaded from `.env.ps1`.
- [ ] `retrieve_secrets()` implemented.
- [ ] `list_secret_properties()` implemented.
- [ ] `create_secret_version()` implemented.
- [ ] `cached_retrieval()` implemented.
- [ ] Python virtual environment activated and dependencies installed.
- [ ] Both secrets retrieved and metadata inspected.
- [ ] Secret properties listed without exposing values.
- [ ] New API-key version created and verified.
- [ ] Cached retrieval tested with 10 accesses and normally 2 Key Vault calls.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- Azure Key Vault securely stores API keys, connection strings, certificates, and other sensitive values.
- RBAC centrally controls who can manage and retrieve secrets.
- Listing secret properties supports inventory and auditing without exposing values.
- Calling `set_secret()` with an existing name creates a new version and preserves prior versions.
- A time-based cache reduces repeated Key Vault API calls, but cached secrets remain in memory until expiration.
