# Lab 24 — Query Logs with KQL in Azure Application Insights

## Overview

In this lab, you will deploy an Azure Application Insights resource, generate sample OpenTelemetry telemetry, analyze application health and performance with Kusto Query Language (KQL), and create an Azure Monitor action group and scheduled-query alert rule.

**Estimated duration:** 20–30 minutes, plus time for Azure resource provisioning and telemetry ingestion.

## Learning objectives

By the end of this lab, you will be able to:

- Deploy Application Insights and configure the required Azure CLI extensions.
- Generate sample request, dependency, and exception telemetry from a Python application.
- Query failed requests, request volume, response-time percentiles, exceptions, and dependency latency using KQL.
- Correlate exceptions with requests using `operation_Id`.
- Create an email action group and a log-search alert for a high request-failure count.
- Verify alert configuration and clean up the resources created for the exercise.

## Prerequisites

- An Azure subscription with permission to create resources and role assignments.
- Visual Studio Code.
- Python 3.12 or later.
- The latest Azure CLI installed and available in the VS Code terminal.
- An email address that can receive the action-group test/alert notifications.

> **Important:** Use a dedicated resource group for this lab. The cleanup step deletes the whole resource group and every resource inside it. Do not use a shared or production resource group.

## Part 1 — Download and configure the starter project

1. Download the starter project ZIP file:

   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/analyze-logs-python.zip>

2. Extract the ZIP into a working folder. In Visual Studio Code, select **File > Open Folder** and open the extracted project folder.

3. Open `azdeploy.py`. At the top of the file, update only the resource group name and Azure region to values valid for your subscription. Do not modify other parts of the script.

   - Resource group: choose a new, dedicated name, such as `rg-ai200-lab24-<unique-suffix>`.
   - Azure region: use an Azure region available to your subscription, such as `eastus` or `centralindia`.

4. In VS Code, select **Terminal > New Terminal** and ensure the terminal profile is **PowerShell**.

5. Sign in to Azure and select the subscription you will use:

   ```powershell
   az login
   ```

   If more than one subscription is available, list them and set the intended subscription:

   ```powershell
   az account list --output table
   az account set --subscription "<subscription-id-or-name>"
   az account show --output table
   ```

6. Register the required resource providers:

   ```powershell
   az provider register --namespace Microsoft.Insights
   az provider register --namespace Microsoft.OperationalInsights
   ```

   Provider registration can take a few minutes. You can check the status with:

   ```powershell
   az provider show --namespace Microsoft.Insights --query registrationState --output tsv
   az provider show --namespace Microsoft.OperationalInsights --query registrationState --output tsv
   ```

7. Install the Azure CLI extensions used by the lab. Run each command on one line:

   ```powershell
   az extension add --name application-insights
   az extension add --name scheduled-query
   ```

8. Start the deployment helper from the project root:

   ```powershell
   python azdeploy.py
   ```

9. When the menu appears, run these options in order:

   - Enter **1 — Create Application Insights**. The script creates the resource group if it does not exist, and creates the Application Insights resource.
   - Enter **2 — Assign role**. This assigns the **Monitoring Metrics Publisher** role so the application can publish telemetry using Microsoft Entra authentication.
   - Enter **3 — Check deployment status**. Confirm the Application Insights resource reports **Succeeded** and the role assignment is complete. If provisioning is still underway, wait and check again.
   - Enter **4 — Retrieve connection info**. This generates the environment files containing values used by the Python app and Azure CLI commands, including the connection string, resource group, Application Insights name/resource ID, and alert email.
   - Enter **5 — Exit**.

10. In the same PowerShell terminal, load the environment variables from the generated PowerShell file:

    ```powershell
    . .\.env.ps1
    ```

    Keep this terminal open. If you open another terminal, run the command again there. Confirm the essential variables are populated:

    ```powershell
    Write-Output "RESOURCE_GROUP=$env:RESOURCE_GROUP"
    Write-Output "APPINSIGHTS_NAME=$env:APPINSIGHTS_NAME"
    Write-Output "APPINSIGHTS_RESOURCE_ID=$env:APPINSIGHTS_RESOURCE_ID"
    Write-Output "ALERT_EMAIL=$env:ALERT_EMAIL"
    ```

    If a value is blank, repeat **option 4 — Retrieve connection info** in `azdeploy.py` and reload `.env.ps1`.

## Part 2 — Generate sample telemetry

The included application uses OpenTelemetry to generate sample telemetry that appears in Application Insights' `requests`, `dependencies`, and `exceptions` tables.

1. From the project root, move into the client folder:

   ```powershell
   Set-Location .\client
   ```

2. Create a Python virtual environment:

   ```powershell
   python -m venv .venv
   ```

3. Activate it in PowerShell:

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

   If script execution is blocked by your machine's policy, follow your organization's approved PowerShell policy rather than changing a system-wide policy without authorization.

4. Install the required Python packages:

   ```powershell
   python -m pip install -r requirements.txt
   ```

5. Run the telemetry generator:

   ```powershell
   python app.py
   ```

   Wait for the script to finish and review its summary. Each run is expected to generate **15 request spans, 12 dependency spans, and 5 exception spans**.

6. Run the generator **two more times**, waiting for each run to finish:

   ```powershell
   python app.py
   ```

   ```powershell
   python app.py
   ```

   Across three successful runs, expect approximately 45 request spans, 36 dependency spans, and 15 exception spans.

7. Wait **2–3 minutes** for telemetry ingestion before querying the data. Application Insights batches telemetry, so data does not necessarily appear immediately.

## Part 3 — Open the Logs blade

1. Open the [Azure portal](https://portal.azure.com/).
2. Navigate to the lab resource group and open the Application Insights resource created by the deployment script.
3. Under **Monitoring**, select **Logs**.
4. Close any query template dialog that appears.
5. Ensure the query editor is in **KQL** mode. Paste each query below into the portal's Logs editor and select **Run**. These are KQL queries for the portal, not PowerShell terminal commands.
6. Set the time range to cover the period when you ran the generator. Start with **Last 1 hour**; widen it if necessary.

## Part 4 — Query failed requests

Run this query to count failed requests by service and HTTP result code:

```kusto
requests
| where success == false
| summarize failedCount = count() by cloud_RoleName, resultCode
| order by failedCount desc
```

**How it works**

- `requests` contains incoming request telemetry.
- `where success == false` filters to failed requests.
- `summarize ... count()` counts failures grouped by service and result code.
- `order by failedCount desc` shows the most frequent failure combinations first.

**Expected result:** Sample rows should include services such as `api-gateway`, `doc-processor`, and `auth-service`, with failure codes such as `500` and `429`. Exact counts can vary by run and telemetry ingestion.

**Trainer checkpoint:** Ask participants which service has the greatest number of failures and whether HTTP 500 and 429 indicate the same kind of problem.

## Part 5 — Query request volume and performance

Run this query to examine request volume and response-time distribution over one-hour intervals:

```kusto
requests
| summarize requestCount = count(),
    avgDuration = avg(duration),
    p95Duration = percentile(duration, 95)
    by bin(timestamp, 1h), cloud_RoleName
| order by timestamp desc
```

**How it works**

- `bin(timestamp, 1h)` groups requests into hourly buckets.
- `requestCount` counts requests in each bucket.
- `avgDuration` calculates the average request duration.
- `p95Duration` is the 95th-percentile duration: 95% of observed requests completed within this duration, while the slowest 5% took at least as long.

**Interpretation:** Compare `avgDuration` with `p95Duration`. A much higher p95 value can indicate tail-latency issues: most requests may be fast, while a smaller group is significantly slower. The `duration` values in Application Insights are typically represented as time durations; check the displayed units when interpreting results.

## Part 6 — Join exceptions with requests

Run this query to correlate exceptions with the requests that triggered them:

```kusto
exceptions
| join kind=inner (requests | project requestName = name, operation_Id, cloud_RoleName) on operation_Id
| summarize exceptionCount = count() by requestName, exceptionType = type, cloud_RoleName
| order by exceptionCount desc
| take 15
```

**How it works**

- The query starts with the `exceptions` table.
- The inner join matches each exception to a request using the shared `operation_Id`.
- The subquery renames the request `name` field to `requestName` to avoid a name collision after the join.
- `summarize` counts exceptions by request name, exception type, and service.
- `take 15` limits the displayed results to 15 rows after sorting.

**Interpretation:** Identify which request operations produce the most exceptions and which exception types occur. The inner join only returns exceptions that match a request with the same `operation_Id`.

## Part 7 — Analyze dependency latency

Run this query to calculate latency statistics for outbound dependencies such as HTTP services or databases:

```kusto
dependencies
| summarize callCount = count(),
    avgDuration = avg(duration),
    p50 = percentile(duration, 50),
    p95 = percentile(duration, 95),
    p99 = percentile(duration, 99)
    by target, type
| order by p95 desc
```

**How it works**

- `dependencies` records outbound calls made by the application.
- `target` identifies the destination and `type` describes the dependency category.
- `p50`, `p95`, and `p99` represent the 50th, 95th, and 99th percentiles.
- Sorting by `p95` puts targets with the highest 95th-percentile latency at the top.

**Interpretation:** A large gap between p50 and p95 suggests that a notable portion of calls is slower than typical. A high p99 may reveal rare but severe latency outliers.

## Part 8 — Create an email action group

Return to the VS Code terminal and ensure you are still using the PowerShell session in which `.env.ps1` was loaded. The command below creates an action group named `pipeline-alerts-ag` with an email action and saves the action group's resource ID in the `$ACTION_GROUP_ID` PowerShell variable.

```powershell
$ACTION_GROUP_ID = az monitor action-group create --resource-group $env:RESOURCE_GROUP --name pipeline-alerts-ag --short-name PipeAlert --action email oncall-email $env:ALERT_EMAIL --query id --output tsv
```

Check that the command returned a non-empty resource ID:

```powershell
Write-Output $ACTION_GROUP_ID
```

If the variable is blank or the command failed, verify that `$env:RESOURCE_GROUP` and `$env:ALERT_EMAIL` are populated, that you are signed in to the correct subscription, and that the deployment script's connection-info step completed.

## Part 9 — Create a scheduled-query alert rule

Create a log-search alert that evaluates failed requests over a five-minute window and fires when more than 10 failures are detected. The alert uses severity **1 (Error)** and triggers the email action group.

Run this complete command on **one PowerShell line**:

```powershell
az monitor scheduled-query create --resource-group $env:RESOURCE_GROUP --name high-failure-rate-alert --scopes $env:APPINSIGHTS_RESOURCE_ID --action-groups $ACTION_GROUP_ID --condition "count 'FailedRequests' > 10" --condition-query 'FailedRequests="requests | where success == false"' --evaluation-frequency 5m --window-size 5m --severity 1 --description "Alert when more than 10 requests fail in a 5-minute window"
```

> **Parameter note:** Use `--evaluation-frequency 5m` as one value. Do not split it into `5 m`.

The alert query defines `FailedRequests` as the failed-request rows. Azure Monitor evaluates the query on its configured frequency and window; when the configured threshold is exceeded, the rule can trigger the action group and send an email. Alert evaluation and email delivery are asynchronous, so a notification may not arrive immediately.

## Part 10 — Verify the alert configuration

1. Display the alert rule's configuration:

   ```powershell
   az monitor scheduled-query show --resource-group $env:RESOURCE_GROUP --name high-failure-rate-alert --query "{Name:name, Severity:severity, Enabled:enabled, Window:windowSize, Frequency:evaluationFrequency, Threshold:condition.failingPeriods}" --output table
   ```

2. Confirm the rule is enabled and that its severity, evaluation frequency, and window match the values configured above.

3. List scheduled-query alert rules in the lab resource group:

   ```powershell
   az monitor scheduled-query list --resource-group $env:RESOURCE_GROUP --output table
   ```

   You should see `high-failure-rate-alert`. Because the sample app generates failures, Azure Monitor begins evaluating the rule on its configured five-minute cycle. Whether an email is sent depends on the number of failed requests in an evaluation window and the alert's evaluation behavior.

## Part 11 — Troubleshooting

### Application Insights was not deployed successfully

- Open the resource group in the Azure portal and confirm the Application Insights resource exists and provisioning succeeded.
- Run `python azdeploy.py` from the project root and choose **3 — Check deployment status**.
- Confirm the active Azure subscription with `az account show --output table`.

### Environment variables are missing

- Confirm `.env.ps1` was generated in the project root.
- Run `. .\.env.ps1` in the same PowerShell session in which you intend to run the Azure CLI commands.
- Check that `APPLICATIONINSIGHTS_CONNECTION_STRING`, `RESOURCE_GROUP`, `APPINSIGHTS_NAME`, `APPINSIGHTS_RESOURCE_ID`, and `ALERT_EMAIL` have values. Do not share or publish the connection string.
- If values are missing, run **4 — Retrieve connection info** in `azdeploy.py` again and reload the file.

### Azure authentication or role assignment fails

- Sign in again with `az login` and set the correct subscription using `az account set --subscription "<subscription-id-or-name>"`.
- Run **2 — Assign role** in the deployment script and wait for role assignment to propagate.
- Confirm that your account has the required permissions to create resources and role assignments.

### Python environment or package errors

- Confirm the virtual environment is active in the PowerShell terminal.
- From the `client` folder, run `python -m pip install -r requirements.txt` again.
- Check installed packages with `python -m pip list`.
- Review the terminal output from `python app.py` for export or authentication errors.

### KQL queries return no results

- Wait two to five minutes after running the telemetry generator and select **Run** again.
- Confirm you opened the correct Application Insights resource.
- Widen the Logs time range, for example from **Last 1 hour** to **Last 4 hours**.
- Confirm the generator completed successfully and inspect terminal output for errors.

### Alert creation fails

- Confirm the scheduled-query extension is installed: `az extension add --name scheduled-query`.
- Check that `$env:APPINSIGHTS_RESOURCE_ID`, `$env:RESOURCE_GROUP`, `$env:ALERT_EMAIL`, and `$ACTION_GROUP_ID` are populated.
- Confirm the Application Insights resource ID belongs to the active subscription and resource group.
- If Azure CLI reports that dynamic installation of a preview extension is required, review the message and your organization's policy before using its suggested configuration.

## Part 12 — Clean up Azure resources

When you have finished reviewing the query results and alert configuration, delete the dedicated resource group created for this lab. This removes the Application Insights resource and the action group/alert resources in that group.

> **Destructive operation:** Verify the resource group value before executing the command. Do not run it against a shared or production resource group.

```powershell
Write-Output "Resource group to delete: $env:RESOURCE_GROUP"
```

After verifying that it is the dedicated lab resource group, run:

```powershell
az group delete --name $env:RESOURCE_GROUP --no-wait --yes
```

The deletion runs asynchronously in Azure. If the environment is being reused for another lab, wait until deletion is complete before recreating resources with the same names.

## Completion checklist

- [ ] Deployed Application Insights and verified the resource status.
- [ ] Assigned Monitoring Metrics Publisher and loaded the PowerShell environment variables.
- [ ] Ran the telemetry generator three times successfully.
- [ ] Queried failed requests and request performance in KQL.
- [ ] Joined exceptions with requests using `operation_Id`.
- [ ] Analyzed dependency latency using p50, p95, and p99 percentiles.
- [ ] Created the `pipeline-alerts-ag` action group.
- [ ] Created and verified `high-failure-rate-alert`.
- [ ] Deleted only the dedicated resource group created for the lab.

## Reference

Official Microsoft Learn exercise: <https://microsoftlearning.github.io/mslearn-azure-ai/instructions/instrument-observe/02-query-logs-kql.html>
