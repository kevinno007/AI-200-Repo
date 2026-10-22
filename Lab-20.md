# Lab 20 — Create an MCP Server with Azure Functions

## Objective

Create a local Azure Functions application that exposes two tools through the Model Context Protocol (MCP). Configure the MCP server, implement text-summarization and document-classification tool triggers, and test tool discovery and invocation from GitHub Copilot in agent mode.

**Estimated duration:** 25 minutes.

> **Preview-feature note:** The Azure Functions MCP extension is evolving. This lab follows the Microsoft Learn exercise's `mcpToolTrigger` and preview-bundle configuration. If your installed Functions Core Tools or extension rejects these settings, consult the [current Azure Functions MCP documentation](https://learn.microsoft.com/azure/azure-functions/functions-bindings-mcp).

## Prerequisites

- Visual Studio Code.
- Azure Functions extension for VS Code.
- Azure Functions Core Tools v4 or later (use a current v4 build).
- Python 3.9 or later.
- GitHub account with GitHub Copilot access.
- GitHub Copilot extension for VS Code.
- Optional: Ruff extension for formatting and linting Python.

**Command format:** Terminal commands are single-line Windows PowerShell commands. JSON configuration and Python source are multiline file contents, not shell commands.

---

## Task 1 — Create the Azure Functions project

1. Create a folder named `mcp-server-functions`.
2. In VS Code, select **File > Open Folder...** and open that folder.
3. Open the Command Palette with `Ctrl+Shift+P` or **View > Command Palette...**.
4. Run **Azure Functions: Create Function...** and select the following options:

| Prompt | Selection |
|---|---|
| Select the folder | The folder opened in VS Code |
| Select a project type | Python |
| Select a Python interpreter | **Skip virtual environment** |
| Select a template | HTTP trigger |
| Function name | Keep the default `http_trigger` |
| Authorization level | `ANONYMOUS` |

The extension scaffolds `function_app.py`, `host.json`, `local.settings.json`, `requirements.txt`, and `.vscode` debugging configuration. You will replace the HTTP-trigger code with MCP tools in Task 3.

## Task 2 — Configure the MCP server

### 2.1 Update `host.json`

Open `host.json`, replace its contents with this JSON, and save:

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle.Preview",
    "version": "[4.*, 5.0.0)"
  },
  "extensions": {
    "mcpToolTrigger": {
      "serverName": "document-tools",
      "serverVersion": "1.0.0",
      "serverInstructions": "Tools for document processing and classification"
    }
  }
}
```

The Preview bundle is used because this exercise uses the preview `mcpToolTrigger` binding. `serverName`, `serverVersion`, and `serverInstructions` identify the server and describe its purpose to MCP clients.

### 2.2 Update `local.settings.json`

Replace its contents with the following and save:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "AzureWebJobsSecretStorageType": "Files"
  }
}
```

If VS Code warns that a storage account is required when starting the local runtime, select **Skip for now** as described in the exercise.

### 2.3 Create `.vscode/mcp.json`

Create a file named `mcp.json` inside the existing `.vscode` folder. Add the following JSON and save:

```json
{
  "servers": {
    "document-tools-local": {
      "type": "sse",
      "url": "http://localhost:7071/runtime/webhooks/mcp/sse"
    }
  }
}
```

This registers the local server with VS Code. The server group will appear in Copilot as `document-tools-local`; the local Functions runtime serves the MCP endpoint on port `7071`.

---

## Task 3 — Define the MCP tool triggers

Open `function_app.py` and **replace the entire file** with the following code. This removes the scaffolded HTTP trigger and defines two MCP tool triggers:

```python
import azure.functions as func
import json
import logging

# Initialize the FunctionApp instance that registers all trigger functions
app = func.FunctionApp()

# Expose summarize_text to MCP clients with one string input.
@app.generic_trigger(
    arg_name="context",
    type="mcpToolTrigger",
    toolName="summarize_text",
    description="Summarize a block of text into key points",
    toolProperties='[{"propertyName": "text", "propertyType": "string", "description": "The text to summarize"}]'
)
def summarize_text(context: str) -> str:
    logging.info(f"summarize_text raw context: {context}")
    request = json.loads(context)
    logging.info(f"summarize_text parsed request: {request}")
    arguments = request.get("arguments", {})
    text = arguments.get("text", "")

    # Placeholder logic: this does not call an AI model.
    summary = f"Summary of {len(text.split())} words: {text[:100]}..."
    return json.dumps({"content": summary})


# Expose classify_document to MCP clients with text and categories inputs.
@app.generic_trigger(
    arg_name="context",
    type="mcpToolTrigger",
    toolName="classify_document",
    description="Classify a document into a category",
    toolProperties='[{"propertyName": "text", "propertyType": "string", "description": "The document text to classify"}, {"propertyName": "categories", "propertyType": "string", "description": "Comma-separated list of possible categories"}]'
)
def classify_document(context: str) -> str:
    logging.info(f"classify_document raw context: {context}")
    request = json.loads(context)
    logging.info(f"classify_document parsed request: {request}")
    arguments = request.get("arguments", {})
    text = arguments.get("text", "")
    categories = arguments.get("categories", "general")

    # Placeholder logic: choose the first supplied category.
    category_list = [c.strip() for c in categories.split(",")]
    selected_category = category_list[0] if category_list else "unknown"

    return json.dumps({
        "content": f"Classification: {selected_category}",
        "category": selected_category
    })
```

Save the file and review the implementation:

- `summarize_text` accepts a text input and returns the word count plus a preview of the first 100 characters.
- `classify_document` accepts text and a comma-separated category list, then selects the first category.
- Both functions parse the MCP request's `arguments` object and log request details.
- These are demonstration implementations, not real AI summarization or classification services.

The `toolName`, `description`, and `toolProperties` parameters define how the tools are presented to MCP clients and which input fields they accept.

---

## Task 4 — Configure the Python environment

Open **Terminal > New Terminal** in VS Code. Confirm the terminal is in the project directory.

Create a virtual environment:

```powershell
python -m venv .venv
```

Activate it using PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation due to the execution policy, allow it for the current process only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

Retry activation:

```powershell
.\.venv\Scripts\Activate.ps1
```

Install the project dependencies:

```powershell
pip install -r requirements.txt
```

Verify the installed packages if needed:

```powershell
pip list
```

Select the correct Python interpreter in VS Code:

1. Open the Command Palette with `Ctrl+Shift+P`.
2. Run **Python: Select Interpreter**.
3. Select the interpreter inside the project's `.venv` folder.

This ensures the debugger and Functions runtime use the correct Python environment.

---

## Task 5 — Start the local Functions runtime

1. Select the **Accounts** icon in VS Code and verify you are signed in to GitHub with an account that has GitHub Copilot access.
2. Press **F5** to start the Functions runtime with the debugger attached.
3. If a storage-account warning appears, select **Skip for now**.
4. Watch the terminal output for the registered functions. It should include both entries:

```text
Functions:
    classify_document: mcpToolTrigger
    summarize_text: mcpToolTrigger
```

If both functions are listed, the local server is running.

Alternatively, after stopping the debugger, the runtime can be started without debugging by running this single-line PowerShell command in the project folder:

```powershell
func start
```

The expected local SSE endpoint is:

`http://localhost:7071/runtime/webhooks/mcp/sse`

Some Functions Core Tools versions display health-check warnings. The course says these can be ignored if the runtime starts and lists both tool triggers.

---

## Task 6 — Confirm tool discovery in GitHub Copilot

1. Confirm VS Code detects `.vscode/mcp.json` and connects to the local MCP server.
2. Open **GitHub Copilot Chat**.
3. Select the **tools** icon at the bottom of the chat window.
4. Find the tool group named **`document-tools-local`**.
5. Confirm both tools appear with their descriptions:
   - `summarize_text`
   - `classify_document`

If a permission prompt appears the first time a tool is invoked, select **Allow in this Session**.

If the tools are not listed, keep the Functions runtime running and use the troubleshooting steps below.

---

## Task 7 — Test with explicit tool prompts

Explicitly naming the tool is the most reliable way to verify MCP invocation.

### 7.1 Test `classify_document`

Enter this prompt in Copilot Chat:

```text
Use the classify_document tool to classify this text: 'This agreement is entered into by Party A and Party B' with categories: contract, invoice, memo
```

Expected result: **contract**. The placeholder implementation selects the first category. Check the Functions terminal logs for the invocation.

### 7.2 Test `summarize_text`

Enter this prompt in Copilot Chat:

```text
Use the summarize_text tool to summarize this text: 'Azure Functions is a serverless compute service that lets you run event-triggered code without having to explicitly provision or manage infrastructure.'
```

Expected result: a string starting with `Summary of`, followed by the word count and a truncated preview of the input. Check the terminal logs to confirm that `summarize_text` ran.

If Copilot responds without invoking a tool, confirm the tool is enabled in the tool picker and try the explicit prompt again.

---

## Task 8 — Test automatic tool selection with natural language

Natural-language prompts rely on the model deciding whether to call a tool, so invocation is less predictable than when the tool is named explicitly.

### 8.1 Automatic classification

Enter this prompt in Copilot Chat:

```text
Is the following text an invoice, contract, or memo? 'Invoice B1234 for services rendered in January 2026. Total amount due: $5,000.'
```

Copilot should recognize that `classify_document` is relevant and invoke it. Confirm the classification and check the terminal logs. If it answers directly without calling a tool, try rephrasing or explicitly naming the tool.

### 8.2 Automatic summarization

Enter this prompt in Copilot Chat:

```text
Give me a brief summary of this text: 'Machine learning models require large datasets for training. The quality of the training data directly impacts model accuracy. Data preprocessing steps include cleaning, normalization, and feature extraction.'
```

Copilot should invoke `summarize_text`. Verify the invocation in the Functions terminal output.

### 8.3 Stop the runtime

Press **Shift+F5** to stop debugging and shut down the Functions runtime.

---

## Troubleshooting

### Azure Functions Core Tools does not start

Check the installed version:

```powershell
func --version
```

- Confirm Azure Functions Core Tools v4 or later is installed; current MCP extension versions may require a recent v4 build.
- Ensure port `7071` is not already used by another process.
- Confirm `local.settings.json` exists in the project root.
- Review terminal output for Python exceptions, missing dependencies, or extension-bundle errors.

### MCP tools do not appear in Copilot

- Check that `.vscode/mcp.json` is saved and the URL is `http://localhost:7071/runtime/webhooks/mcp/sse`.
- Confirm the Functions runtime is running and shows both trigger functions.
- Verify the server group is named `document-tools-local`.
- Restart VS Code if it does not detect the MCP configuration after saving.
- Ensure the signed-in GitHub account has Copilot access.

### Tool invocation returns errors

- Check the Functions terminal output for Python exceptions or stack traces.
- Verify `function_app.py` has the correct decorators, argument names, and indentation.
- Confirm the virtual environment is active and dependencies from `requirements.txt` are installed.
- Confirm the request contains a JSON envelope with an `arguments` object.

### Copilot answers without invoking a tool

- Check terminal logs to see whether a function ran.
- Use explicit prompts that name `classify_document` or `summarize_text`.
- Verify the tools are visible and enabled in Copilot's tool picker.
- Automatic tool selection is less deterministic than explicit invocation.

### Python environment problems

Check the installed dependencies:

```powershell
pip list
```

- Ensure VS Code uses the interpreter inside `.venv` via **Python: Select Interpreter**.
- If required, activate the environment again and install dependencies with `pip install -r requirements.txt`.

### MCP configuration is incompatible with the installed extension

This exercise uses the preview bundle and `mcpToolTrigger` configuration. The extension evolves; if current Core Tools reports that the binding or `host.json` settings are unrecognized, consult the [current Azure Functions MCP documentation](https://learn.microsoft.com/azure/azure-functions/functions-bindings-mcp) and align the project with the installed version.

---

## Lab completion checklist

- [ ] Project folder created and opened in VS Code.
- [ ] Python Azure Functions project scaffolded with HTTP trigger and anonymous authorization.
- [ ] `host.json` configured with the preview extension bundle and MCP server settings.
- [ ] `local.settings.json` configured.
- [ ] `.vscode/mcp.json` created with the local SSE endpoint.
- [ ] `function_app.py` replaced with `summarize_text` and `classify_document`.
- [ ] Virtual environment created, activated, and dependencies installed.
- [ ] VS Code interpreter set to `.venv`.
- [ ] Runtime starts and lists both MCP tool triggers.
- [ ] Both tools are visible under `document-tools-local` in Copilot.
- [ ] Explicit classification and summarization prompts tested successfully.
- [ ] Natural-language tool discovery tested and terminal logs checked.
- [ ] Runtime stopped with **Shift+F5**.

## Key takeaways

- Azure Functions MCP triggers expose functions as tools for MCP clients to discover and invoke.
- `toolName`, `description`, and `toolProperties` define each tool and its input schema.
- Explicitly naming a tool is the most reliable way to test the MCP connection.
- Natural-language discovery depends on the model's judgment and may require rephrasing.
- The sample tool logic is intentionally simple; a production server would call Azure AI services and use appropriate identity and security controls.
