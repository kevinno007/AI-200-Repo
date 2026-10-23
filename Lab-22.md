# Lab 22 — Retrieve Settings and Secrets from Azure App Configuration

## Lab objective

Deploy Azure App Configuration and an RBAC-enabled Azure Key Vault, then complete a Python Flask application that demonstrates three configuration-management patterns:

1. Load configuration using label stacking and resolve a Key Vault reference automatically.
2. List individual setting properties, including labels and metadata, without resolving secret values.
3. Trigger sentinel-based dynamic refresh so the app picks up a changed setting without restarting.

**Estimated duration:** 30 minutes.

## Prerequisites

- An Azure subscription with permissions and quota to create the required resources.
- Visual Studio Code (VS Code).
- Python 3.12 or later.
- The latest Azure CLI.
- Optional: Ruff extension for VS Code for formatting and linting Python code.

**Command convention:** All terminal commands in this guide use Windows PowerShell and are single-line commands. Python functions and JSON configuration are multiline file contents, not shell commands.

---

## Task 1 — Download the starter project and deploy Azure App Configuration

1. Download the official starter ZIP file:

   [Download app-config-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/app-config-python.zip)

2. Move the ZIP file to a working folder and extract it.
3. Open VS Code. Select **File > Open Folder...** and open the extracted project folder.
4. Open `azdeploy.py`.
5. Change **only** the two values at the top of the script:
   - Resource group name: replace `<your-resource-group-name>` with a dedicated resource group name.
   - Azure region: replace `<your-azure-region>` with a region available to your subscription.
6. Save the file. Do not modify other code in `azdeploy.py`.
7. Select **Terminal > New Terminal** in VS Code. Ensure the terminal is PowerShell and is in the project root.

Sign in to Azure:

```powershell
az login
```

Complete the browser prompts and select the correct Azure account and subscription.

Register the required resource providers:

```powershell
az provider register --namespace Microsoft.AppConfiguration
```

```powershell
az provider register --namespace Microsoft.KeyVault
```

Start the deployment script from the project root:

```powershell
python azdeploy.py
```

At the deployment menu, run these options **in order**:

1. Enter **`1`** — **Create App Configuration**. This creates the resource group if required and deploys an Azure App Configuration store.
2. Enter **`2`** — **Create Key Vault**. This creates an Azure Key Vault with RBAC authorization enabled.
3. Enter **`3`** — **Assign roles**. This assigns the **App Configuration Data Owner** role and **Key Vault Secrets Officer** role to your account so the app can manage settings and secrets using Microsoft Entra authentication.
4. Enter **`4`** — **Store settings**. This creates default unlabeled settings, `Production`-labeled overrides, a Key Vault secret, a Key Vault reference in App Configuration, and a sentinel setting for dynamic refresh.
5. Enter **`5`** — **Check deployment status**. Confirm the App Configuration store and Key Vault show **Succeeded**, role assignments are complete, and the settings have been stored. If provisioning or role assignment is not complete, wait and run option 5 again.
6. Enter **`6`** — **Retrieve connection info**. This creates the environment-variable file containing the App Configuration endpoint URL.
7. Enter **`7`** — **Exit**.

Load the environment variables in the current PowerShell session from the project root:

```powershell
. .\.env.ps1
```

Keep this terminal open. If you create a new terminal, load the environment variables again in that session.

---

## Task 2 — Implement loading settings with label stacking and Key Vault reference resolution

Open `client/appconfig_functions.py` in VS Code. The file has `BEGIN / END` comments marking the sections to complete.

**Indentation tip:** Paste each function flush with the left margin (column 1), select the pasted lines, and press **Tab** to align the code with the surrounding `BEGIN / END` markers. Use **Shift+Tab** to outdent if needed. Use spaces consistently. Preserve the imports and existing `get_provider()` and `get_client()` helpers.

Find `# BEGIN LOAD SETTINGS FUNCTION` and add:

```python
def load_settings():
    """Load all settings with label stacking and Key Vault reference resolution."""
    provider = get_provider()
    results = []

    # The provider resolves Key Vault references automatically and
    # applies label stacking: Production-labeled values override
    # unlabeled defaults for matching keys.
    known_keys = [
        "OpenAI:Endpoint",
        "OpenAI:DeploymentName",
        "OpenAI:ApiKey",
        "Pipeline:BatchSize",
        "Pipeline:RetryCount",
        "Sentinel"
    ]

    for key in known_keys:
        try:
            value = provider[key]
            is_secret = key == "OpenAI:ApiKey"
            display_value = value[:10] + "..." if is_secret and len(value) > 10 else value
            results.append({
                "key": key,
                "value": display_value,
                "type": "Key Vault reference" if is_secret else "configuration",
                "status": "loaded"
            })
        except KeyError:
            results.append({
                "key": key,
                "value": None,
                "type": "unknown",
                "status": "not found"
            })

    return results
```

### How label stacking works

The configuration provider merges two sets of settings in this order:

1. Settings with no label are loaded as the defaults. The null label selector is represented by `"\0"`.
2. Settings with the `Production` label are loaded second. Where a Production setting has the same key as a default, the Production value overrides it.

The provider is configured by the starter's `get_provider()` helper to resolve Key Vault references automatically, using the same credential. Therefore, `OpenAI:ApiKey` returns the referenced secret value rather than its reference URI. The function truncates the API key for display.

Save the file.

---

## Task 3 — Implement listing of setting properties

Find `# BEGIN LIST SETTINGS FUNCTION` and add:

```python
def list_setting_properties():
    """List all setting properties from the App Configuration store."""
    client = get_client()
    results = []

    # List every stored setting entry, including each label.
    # This displays the raw storage view rather than the merged provider view.
    for setting in client.list_configuration_settings():
        results.append({
            "key": setting.key,
            "label": setting.label or "(no label)",
            "content_type": setting.content_type or "—",
            "last_modified": str(setting.last_modified) if setting.last_modified else "—",
            "read_only": setting.read_only
        })

    return results
```

This function uses the management client to list each setting separately, including unlabeled defaults and Production-labeled overrides. It displays the key, label, content type, last-modified timestamp, and read-only flag without retrieving setting values.

Save your changes.

---

## Task 4 — Implement sentinel-based dynamic refresh

Find `# BEGIN REFRESH CONFIGURATION FUNCTION` and add:

```python
def refresh_configuration():
    """Demonstrate sentinel-based dynamic refresh of configuration settings."""
    provider = get_provider()
    client = get_client()

    # Capture the current value before making a change.
    tracked_keys = ["Pipeline:BatchSize"]
    before = {}
    for key in tracked_keys:
        try:
            before[key] = provider[key]
        except KeyError:
            before[key] = "—"

    # Simulate a setting change using a random Production override.
    import random
    new_batch = str(random.randint(100, 999))

    setting = ConfigurationSetting(
        key="Pipeline:BatchSize",
        value=new_batch,
        label="Production",
        content_type="text/plain"
    )
    client.set_configuration_setting(setting)

    # Change the sentinel so the provider knows settings have changed.
    # A timestamp makes the sentinel value different from its prior value.
    new_sentinel = str(int(time.time()))
    sentinel_setting = ConfigurationSetting(
        key="Sentinel",
        value=new_sentinel
    )
    client.set_configuration_setting(sentinel_setting)

    # Wait for the refresh interval, then ask the provider to reload.
    time.sleep(2)
    provider.refresh()

    # Capture the value after refresh.
    after = {}
    for key in tracked_keys:
        try:
            after[key] = provider[key]
        except KeyError:
            after[key] = "—"

    settings = []
    for key in tracked_keys:
        settings.append({
            "key": key,
            "before": before[key],
            "after": after[key],
            "changed": before[key] != after[key]
        })

    return {
        "settings": settings,
        "sentinel_value": new_sentinel,
        "new_batch_size": new_batch,
        "batch_size_updated": after["Pipeline:BatchSize"] == new_batch
    }
```

### What the sentinel does

A sentinel is a configuration key used as a change signal. The function:

1. Reads the current `Pipeline:BatchSize`.
2. Writes a new random `Pipeline:BatchSize` value under the `Production` label.
3. Changes the unlabeled `Sentinel` key to a new timestamp.
4. Waits briefly, then calls `provider.refresh()`.
5. Reads `Pipeline:BatchSize` again and compares the before/after values.

This demonstrates how the application can refresh configuration without restarting. The starter's provider configuration establishes the sentinel watch and refresh behavior.

Save `appconfig_functions.py`. Ensure all three functions are inserted under the correct markers. Do not remove or alter code outside the designated sections.

---

## Task 5 — Configure the Python environment

From the project root, change to the client directory:

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

If PowerShell blocks activation because of the execution policy, allow it for the current process only:

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

Verify the installed packages if needed:

```powershell
pip list
```

If this is a new terminal session, load the environment variables from the `client` directory:

```powershell
. ..\.env.ps1
```

The prompt should display `(.venv)` when the environment is active.

---

## Task 6 — Run the Flask application

From the `client` directory, start the app:

```powershell
python app.py
```

Open a browser and navigate to:

[http://localhost:5000](http://localhost:5000)

The web interface contains **Load Settings**, **List Setting Properties**, and **Refresh Configuration** actions.

### 6.1 Load settings

1. Select **Load Settings**.
2. Review the returned key, value, type, and status for each setting.
3. Confirm the results include:
   - `OpenAI:Endpoint`
   - `OpenAI:DeploymentName`
   - `OpenAI:ApiKey`
   - `Pipeline:BatchSize`
   - `Pipeline:RetryCount`
   - `Sentinel`
4. Verify that `OpenAI:ApiKey` is shown as a **Key Vault reference**, and that its displayed value is truncated.
5. Confirm the `Production` override for `Pipeline:BatchSize` takes precedence over the default, so the loaded value is **200**.

### 6.2 List setting properties

1. Select **List Setting Properties**.
2. Review each individual entry, including key, label, content type, last-modified timestamp, and read-only state.
3. Confirm that `Pipeline:BatchSize` appears twice:
   - Unlabeled default with value `10`.
   - `Production`-labeled override with value `200`.

This is the raw storage view. Unlike **Load Settings**, it does not merge labels into one final value.

### 6.3 Refresh configuration

1. Select **Refresh Configuration**.
2. Review the `Pipeline:BatchSize` value before and after the update.
3. Confirm the after-value matches the generated random value, and the result indicates the setting changed.
4. Check that a new sentinel value was written.

This verifies that the provider refreshed its configuration without restarting the Flask app.

When you finish testing, stop Flask in the PowerShell terminal by pressing **Ctrl+C**.

---

## Task 7 — Clean up Azure resources

Delete the dedicated resource group configured in `azdeploy.py`. Replace `<rg-name>` with the actual resource group name:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes every resource inside it. Use a dedicated group for this lab. If you used an existing group containing unrelated resources, those resources will also be deleted.

---

## Troubleshooting

### Verify the Azure deployment

- In the Azure portal, open your resource group and confirm the App Configuration store shows **Provisioning State: Succeeded**.
- Confirm the Key Vault also shows **Succeeded** and has RBAC authorization enabled.
- Run `python azdeploy.py` and select option **5** to check deployment status.
- If settings are missing, use option **4** to store the settings again.

### Check code placement and indentation

- Confirm all three functions are in `client/appconfig_functions.py` under the correct `BEGIN / END` markers.
- Use spaces consistently.
- Preserve the starter's imports and `get_provider()` and `get_client()` helpers.
- Do not alter unrelated code.

### Verify environment variables

- Confirm `.env` and `.env.ps1` exist in the project root and include `AZURE_APPCONFIG_ENDPOINT`.
- From the project root, load the variables with `. .\.env.ps1`.
- From `client`, load the variables with `. ..\.env.ps1`.
- Every new terminal session needs the variables loaded separately.

### Check authentication and roles

Confirm the current Azure CLI session:

```powershell
az account show
```

- Verify the **App Configuration Data Owner** and **Key Vault Secrets Officer** roles are assigned.
- If roles were assigned recently, wait for them to propagate and retry.

### Python environment or package errors

- Confirm that `(.venv)` appears in the terminal prompt.
- Run `pip install -r requirements.txt` from `client`.
- Check installed dependencies with `pip list`.
- If VS Code uses the wrong interpreter, run **Python: Select Interpreter** and choose the `.venv` interpreter.

### Key Vault reference is not resolved

- Confirm the Key Vault is configured for RBAC and the Key Vault Secrets Officer role has been assigned.
- Confirm the Key Vault reference was created when deployment option **4** stored the settings.
- Confirm `get_provider()` in the starter app uses Key Vault reference resolution.
- Do not expose full secret values in logs or UI output.

### Dynamic refresh does not show a change

- Confirm `Pipeline:BatchSize` is being updated with the `Production` label.
- Confirm the sentinel key changes after the configuration update.
- The refresh function waits 2 seconds before calling `provider.refresh()`. If necessary, run the refresh action again after checking the terminal output.
- Confirm the provider's sentinel refresh settings are preserved in the starter helper.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure login completed and both resource providers registered.
- [ ] App Configuration store deployed.
- [ ] RBAC-enabled Key Vault deployed.
- [ ] App Configuration Data Owner and Key Vault Secrets Officer roles assigned.
- [ ] Sample settings, Production overrides, Key Vault secret/reference, and sentinel stored.
- [ ] Deployment status verified and environment variables loaded.
- [ ] `load_settings()` implemented.
- [ ] `list_setting_properties()` implemented.
- [ ] `refresh_configuration()` implemented.
- [ ] Python virtual environment created and dependencies installed.
- [ ] Flask app started and all settings loaded.
- [ ] Production `Pipeline:BatchSize` value `200` observed overriding default `10`.
- [ ] All properties listed with their labels and metadata.
- [ ] Sentinel-based refresh tested and new batch size observed without restarting.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- Azure App Configuration centralizes non-secret application settings.
- Labels support environment-specific overrides; the later `Production` selector overrides unlabeled defaults for matching keys.
- Key Vault references let App Configuration settings point to secrets stored securely in Key Vault.
- Listing setting properties reveals metadata and separate labels without loading secret values.
- A sentinel key signals that configuration changed, allowing the application to refresh settings without a restart.
