# Lab 23 — Instrument an App with OpenTelemetry and Azure Monitor

## Lab objective
Deploy Application Insights and complete a Python Flask app that instruments a document-processing pipeline with OpenTelemetry. Configure Azure Monitor OpenTelemetry, create parent/child spans, add searchable attributes, and use Application Insights transaction search and KQL to diagnose a simulated performance bottleneck.

**Estimated duration:** 25 minutes.

## Prerequisites
- Azure subscription with permissions and quota for the required resources.
- Visual Studio Code.
- Python 3.12 or later.
- Latest Azure CLI.
- Optional: Ruff extension for VS Code.

**Command convention:** All terminal commands below are single-line Windows PowerShell commands. Python code and KQL are multiline content to paste into the indicated editor, not shell commands.

---

## Task 1 — Download the starter project and deploy Application Insights

1. Download the official starter ZIP: [instrument-app-python.zip](https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/instrument-app-python.zip).
2. Extract it, then open the extracted folder in VS Code.
3. Open `azdeploy.py`. Change only the resource group name (`<your-resource-group-name>`) and Azure region (`<your-azure-region>`) at the top. Save the file.
4. Open **Terminal > New Terminal** in VS Code.

Sign in to Azure:

```powershell
az login
```

Register the required providers:

```powershell
az provider register --namespace Microsoft.Insights
```

```powershell
az provider register --namespace Microsoft.OperationalInsights
```

Install the Application Insights CLI extension:

```powershell
az extension add --name application-insights
```

Run the deployment script:

```powershell
python azdeploy.py
```

At the menu, select these options in order:

1. Enter **`1`** — **Create Application Insights**. The script creates the resource group if needed and deploys Application Insights.
2. Enter **`2`** — **Assign role**. This assigns **Monitoring Metrics Publisher** to your account so the app can publish telemetry using Microsoft Entra authentication.
3. Enter **`3`** — **Check deployment status**. Verify Application Insights shows `Succeeded` and the role is assigned. If provisioning is still in progress, wait and check again.
4. Enter **`4`** — **Retrieve connection info**. The script creates `.env` and `.env.ps1` with `APPLICATIONINSIGHTS_CONNECTION_STRING` and `OTEL_SERVICE_NAME`.
5. Enter **`5`** — **Exit**.

Load environment variables from the project root:

```powershell
. .\.env.ps1
```

Keep this terminal open. Load the variables again in any new terminal session.

---

## Task 2 — Configure Azure Monitor OpenTelemetry

Open `client/telemetry_functions.py`. Add the following code under `# BEGIN CONFIGURE TELEMETRY FUNCTION`.

**Indentation tip:** Paste code flush with the left margin, select the pasted lines, and press **Tab** to align with the existing `BEGIN / END` markers. Use **Shift+Tab** to outdent if needed. Preserve existing imports and helpers.

```python
def configure_telemetry():
    """Configure the Azure Monitor OpenTelemetry Distro."""
    connection_string = os.environ.get("APPLICATIONINSIGHTS_CONNECTION_STRING")

    if not connection_string:
        raise ValueError(
            "APPLICATIONINSIGHTS_CONNECTION_STRING environment variable must be set"
        )

    from azure.monitor.opentelemetry import configure_azure_monitor

    credential = DefaultAzureCredential(
        exclude_managed_identity_credential=True
    )

    configure_azure_monitor(
        connection_string=connection_string,
        credential=credential,
    )
```

Why it matters:
- `APPLICATIONINSIGHTS_CONNECTION_STRING` identifies the Application Insights resource.
- `DefaultAzureCredential` uses Microsoft Entra authentication.
- `exclude_managed_identity_credential=True` avoids repeated attempts to contact Azure Instance Metadata Service during local development.
- `configure_azure_monitor()` configures the exporter and automatic Flask request instrumentation.
- `OTEL_SERVICE_NAME` sets the service name shown in the Application Map.

Save the file.

---

## Task 3 — Implement the parent span

Under `# BEGIN PROCESS DOCUMENTS FUNCTION`, add:

```python
def process_documents(doc_count):
    """Process a batch of documents through the pipeline with tracing."""
    tracer = get_tracer()
    results = []

    with tracer.start_as_current_span("process-documents") as parent_span:
        parent_span.set_attribute("document.count", doc_count)
        parent_span.set_attribute("pipeline.name", "document-processing")

        for i in range(1, doc_count + 1):
            doc_id = f"DOC-{i:04d}"
            validate_result = validate_document(doc_id)
            enrich_result = enrich_document(doc_id)
            store_result = store_document(doc_id)
            results.append({
                "doc_id": doc_id,
                "validate": validate_result,
                "enrich": enrich_result,
                "store": store_result
            })

        parent_span.set_attribute("document.processed", len(results))

    return results
```

The `process-documents` span is the parent for the batch. It records the requested document count and the number processed. Each stage function creates a child span.

Save the file.

---

## Task 4 — Instrument the pipeline stages

Under `# BEGIN PIPELINE STAGE FUNCTIONS`, add all three functions:

```python
def validate_document(doc_id):
    """Validate a document and record a traced span."""
    tracer = get_tracer()
    with tracer.start_as_current_span("validate-document") as span:
        span.set_attribute("document.id", doc_id)
        span.set_attribute("document.stage", "validate")
        time.sleep(random.uniform(0.05, 0.15))
        is_valid = True
        span.set_attribute("document.valid", is_valid)
        span.set_status(StatusCode.OK)
    return {"status": "valid", "duration_ms": round(random.uniform(50, 150))}


def enrich_document(doc_id):
    """Enrich a document with metadata and record a traced span."""
    tracer = get_tracer()
    with tracer.start_as_current_span("enrich-document") as span:
        span.set_attribute("document.id", doc_id)
        span.set_attribute("document.stage", "enrich")
        if doc_id in ("DOC-0003", "DOC-0005"):
            delay = random.uniform(1.5, 3.0)
            span.set_attribute("enrichment.slow", True)
        else:
            delay = random.uniform(0.05, 0.2)
            span.set_attribute("enrichment.slow", False)
        time.sleep(delay)
        span.set_attribute("enrichment.duration_s", round(delay, 3))
        span.set_status(StatusCode.OK)
    return {
        "status": "enriched",
        "duration_ms": round(delay * 1000),
        "slow": doc_id in ("DOC-0003", "DOC-0005")
    }


def store_document(doc_id):
    """Store a document and record a traced span."""
    tracer = get_tracer()
    with tracer.start_as_current_span("store-document") as span:
        span.set_attribute("document.id", doc_id)
        span.set_attribute("document.stage", "store")
        span.set_attribute("storage.type", "blob")
        time.sleep(random.uniform(0.05, 0.2))
        span.set_status(StatusCode.OK)
    return {"status": "stored", "duration_ms": round(random.uniform(50, 200))}
```

Each function creates a span with `start_as_current_span()`. Since `process_documents()` is active, the stage spans automatically become children of the parent span.

The custom attributes are searchable in Application Insights:
- `document.id`: document identifier.
- `document.stage`: validate, enrich, or store.
- `document.valid`: validation result.
- `enrichment.slow`: identifies slow enrichment.
- `enrichment.duration_s`: simulated enrichment duration.
- `storage.type`: storage operation type.

**Expected behavior:** `DOC-0003` and `DOC-0005` experience 1.5–3 seconds of enrichment latency. Other documents take substantially less time.

Save `telemetry_functions.py`. Ensure all code is under the correct markers and preserve existing imports and helpers.

---

## Task 5 — Configure the Python environment

From the project root, navigate to `client`:

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

Verify packages if needed:

```powershell
pip list
```

If this is a new terminal session, load environment variables from `client`:

```powershell
. ..\.env.ps1
```

The prompt should show `(.venv)` when the environment is active.

---

## Task 6 — Run the app and generate telemetry

From `client`, start Flask:

```powershell
python app.py
```

Open [http://localhost:5000](http://localhost:5000).

### 6.1 Check telemetry status
1. Select **Check Telemetry Status**.
2. Verify telemetry is active and `service.name` is `document-pipeline-app`.

### 6.2 Process documents
1. Select **Process Documents**.
2. Verify five documents appear in the results.
3. Confirm `DOC-0003` and `DOC-0005` show much higher enrichment durations and a **SLOW** tag.
4. Select **Process Documents** two more times to generate more traces.
5. Leave the app running while inspecting Application Insights.

Wait **2–3 minutes** for telemetry to arrive.

---

## Task 7 — Inspect a transaction in Application Insights

1. Open the [Azure portal](https://portal.azure.com).
2. Find the Application Insights resource in your lab resource group.
3. Under **Investigate**, select **Search**.
4. Find a `POST /process-documents` entry and open it.
5. Inspect the end-to-end transaction hierarchy:
   - Root HTTP request span.
   - `process-documents` parent span.
   - Child spans for `validate-document`, `enrich-document`, and `store-document`.

Find `enrich-document` spans lasting 1.5 seconds or more. Select one associated with `DOC-0003` or `DOC-0005` and verify attributes including `document.id`, `document.stage`, and `enrichment.slow = True`.

---

## Task 8 — Query telemetry with KQL

In the Application Insights resource, select **Logs** under **Monitoring**. Close any query-template dialog and select **KQL mode**.

### 8.1 Inspect custom spans

Run this query:

```kusto
dependencies
| where timestamp > ago(1h)
| project timestamp, name, duration,
    documentId = customDimensions["document.id"],
    stage = customDimensions["document.stage"],
    slow = customDimensions["enrichment.slow"]
| order by timestamp desc
```

Review rows for `validate-document`, `enrich-document`, and `store-document`. The `slow` column should be `True` for the slow enrichment spans associated with `DOC-0003` and `DOC-0005`.

### 8.2 Compare slow and fast enrichment

Run this query:

```kusto
dependencies
| where timestamp > ago(1h) and name == "enrich-document"
| extend slow = tostring(customDimensions["enrichment.slow"])
| summarize avgDuration = round(avg(duration), 0) by slow
```

Expected results:
- `slow = True`: average duration of **1,500 ms or more**.
- `slow = False`: average duration of **under 200 ms**.

This confirms enrichment is the bottleneck. Stop Flask with **Ctrl+C** when finished.

---

## Task 9 — Clean up Azure resources

Replace `<rg-name>` with the resource group name configured in `azdeploy.py`:

```powershell
az group delete --name "<rg-name>" --no-wait --yes
```

**Caution:** Deleting a resource group deletes every resource inside it. Use a dedicated resource group.

---

## Troubleshooting

### Application Insights deployment
- In the Azure portal, verify the resource shows **Provisioning State: Succeeded**.
- Run `python azdeploy.py` and select option **3** to check status.

### Connection string and environment variables
- Confirm `.env` and `.env.ps1` exist in the project root and contain `APPLICATIONINSIGHTS_CONNECTION_STRING`.
- Confirm `OTEL_SERVICE_NAME` is `document-pipeline-app`.
- From the root, run `. .\.env.ps1`; from `client`, run `. ..\.env.ps1`.
- Each new terminal session needs environment variables loaded separately.

### Authentication and role
Verify the Azure CLI session:

```powershell
az account show
```

- Ensure **Monitoring Metrics Publisher** is assigned to your account.
- If the role was recently assigned, wait for propagation and retry.
- If necessary, rerun `python azdeploy.py` and choose option **2**.

### Code and Python environment
- Confirm all functions are in `client/telemetry_functions.py` under the correct `BEGIN / END` markers.
- Preserve existing imports and helpers; use consistent indentation.
- Ensure `(.venv)` is active and run `pip install -r requirements.txt`.
- If VS Code uses the wrong interpreter, run **Python: Select Interpreter** and choose `.venv`.

### Telemetry is missing
- Wait **2–5 minutes** after generating telemetry, then refresh the portal.
- Compare the connection string with the value on the Application Insights **Overview** page.
- Check the VS Code terminal for export errors.
- If KQL returns no results, widen the query time range from `ago(1h)` to `ago(4h)`.

---

## Lab completion checklist

- [ ] Starter ZIP extracted and `azdeploy.py` configured.
- [ ] Azure login completed and `Microsoft.Insights` and `Microsoft.OperationalInsights` registered.
- [ ] Application Insights CLI extension installed.
- [ ] Application Insights deployed and Monitoring Metrics Publisher role assigned.
- [ ] `.env` and `.env.ps1` created and environment variables loaded.
- [ ] `configure_telemetry()` implemented.
- [ ] `process_documents()` implemented with parent-span attributes.
- [ ] Three pipeline stages implemented with child spans.
- [ ] Python virtual environment activated and dependencies installed.
- [ ] Telemetry status verified and five documents processed three times.
- [ ] End-to-end transaction inspected in Application Insights.
- [ ] KQL used to inspect custom span attributes.
- [ ] Slow and fast enrichment durations compared.
- [ ] Flask stopped and Azure resources deleted.

## Key takeaways

- OpenTelemetry standardizes traces, metrics, and logs.
- Azure Monitor OpenTelemetry Distro integrates Python apps with Application Insights and automatically instruments Flask requests.
- Parent/child spans reveal the structure of a multi-stage operation.
- Custom span attributes make telemetry searchable by document ID and pipeline stage.
- Application Insights transaction search and KQL help isolate slow spans and quantify performance differences.
