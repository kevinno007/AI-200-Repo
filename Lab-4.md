# Lab 4: Diagnose and Fix a Failing Deployment in Azure Container Apps

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Azure services:** Azure Container Registry (ACR), Azure Container Apps, Log Analytics  
**Shell:** Windows PowerShell only

## Lab objectives

In this lab, you will deploy a mock AI document-processing API to Azure Container Apps, deliberately introduce two configuration problems, diagnose them using revisions and logs, and fix them.

You will:
- Deploy a container app and verify its default endpoint.
- Remove a required environment variable, observe the effect, diagnose the cause, and restore it.
- Configure an incorrect ingress target port, observe failed requests, diagnose the mismatch, and fix it.
- Query historical console logs in Log Analytics, including error-level messages.
- Delete the lab resources when finished.

## Important billing note

Microsoft Learn notes that Azure Container Registry task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan** for this exercise. Azure resources may incur charges until you delete them.

## Prerequisites

Before starting, ensure you have:
- An Azure subscription with permission and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- Python 3.12 or later installed.

Check the tools from the VS Code PowerShell terminal:

```powershell
az --version
```

```powershell
python --version
```

If either command is not recognized, install or repair the corresponding prerequisite before continuing.

---

## Task 1: Download and prepare the starter project

1. Download the starter ZIP file from the official project link:

   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aca-manage-python.zip>

2. Move the ZIP file to your preferred working location and extract it into a folder.

3. Open Visual Studio Code. Select **File > Open Folder...** and open the extracted project folder.

4. Open `azdeploy.py`. At the top of the file, change **only** the resource group name and Azure region values, then save the file. Do not change anything else in the script.

   ```python
   "<your-resource-group-name>"  # Resource Group name
   "<your-azure-region>"         # Azure region for the resources
   ```

   Choose a valid region available to your subscription. Record the resource group name because you will need it for cleanup.

5. In VS Code, select **Terminal > New Terminal**. Ensure that the terminal is running **PowerShell** and that the current directory is the project root—the folder containing `azdeploy.py`.

## Task 2: Sign in, install extensions, and register resource providers

1. Sign in to Azure and follow the prompts to select the correct account and subscription.

   ```powershell
   az login
   ```

2. Confirm the active subscription.

   ```powershell
   az account show --output table
   ```

3. If the wrong subscription is active, select the intended subscription.

   ```powershell
   az account set --subscription "<your-subscription-name-or-id>"
   ```

4. Install the Azure Container Apps CLI extension.

   ```powershell
   az extension add --name containerapp
   ```

5. Install the Log Analytics CLI extension.

   ```powershell
   az extension add --name log-analytics
   ```

6. Register the Azure Container Apps resource provider.

   ```powershell
   az provider register --namespace Microsoft.App
   ```

7. Register the Log Analytics resource provider.

   ```powershell
   az provider register --namespace Microsoft.OperationalInsights
   ```

8. Register the Azure Container Registry resource provider.

   ```powershell
   az provider register --namespace Microsoft.ContainerRegistry
   ```

## Task 3: Deploy the resources and verify the application

1. Make sure the terminal is in the project root and run the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. When prompted, enter `1` to select **Create Azure Container Registry and build container image**. Wait for ACR Tasks to build and push the image.

3. When the first operation finishes, enter `2` to select **Create Container Apps environment**. Wait for the environment creation to finish.

4. When that operation finishes, enter `3` to select **Deploy the container app and configure secrets**. Wait for the container app to be deployed.

5. When deployment completes, enter `5` to exit the script.

6. Load the generated environment variables into the current PowerShell session.

   ```powershell
   . .\.env.ps1
   ```

   Keep this terminal open. If you open another terminal, load `.env.ps1` again.

7. Retrieve the app's fully qualified domain name (FQDN).

   ```powershell
   $FQDN = az containerapp show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --query properties.configuration.ingress.fqdn -o tsv
   ```

8. Display the FQDN.

   ```powershell
   Write-Output $FQDN
   ```

9. Call the default endpoint to verify that the application is running.

   ```powershell
   Invoke-RestMethod -Uri "https://$FQDN/"
   ```

   **Expected result:** The endpoint returns JSON. Check that `model.name` is set to `gpt-5.4-mini`.

If the endpoint is not responding immediately, wait a minute and try again while the app starts.

## Task 4: Diagnose and fix a missing environment variable

The app depends on the `MODEL_NAME` environment variable. In this task, remove it deliberately, observe the change, inspect the app configuration, and restore it.

### Step 4.1: Introduce the problem

1. Remove the `MODEL_NAME` environment variable.

   ```powershell
   az containerapp update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --remove-env-vars MODEL_NAME
   ```

2. List the revisions.

   ```powershell
   az containerapp revision list -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP -o table
   ```

   **What to look for:** A new revision with a higher suffix (for example, `ai-api--0000002`) and `TrafficWeight` of `100`, indicating that the new revision receives all traffic.

3. Call the root endpoint again and inspect the model information.

   ```powershell
   (Invoke-RestMethod -Uri "https://$FQDN/").model
   ```

   **Expected result:** `model.name` shows the default value `not-configured` instead of `gpt-5.4-mini`.

### Step 4.2: Diagnose the cause

Inspect the container environment variables configured on the app.

```powershell
az containerapp show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --query "properties.template.containers[0].env" -o table
```

**Expected result:** `MODEL_NAME` is missing. The API response and configuration inspection together identify the cause.

### Step 4.3: Fix and verify

1. Add the environment variable back using the value loaded from `.env.ps1`.

   ```powershell
   az containerapp update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --set-env-vars "MODEL_NAME=$($env:MODEL_NAME)"
   ```

   **Important:** This change creates another revision. Wait for the update to finish.

2. Check the root endpoint again.

   ```powershell
   (Invoke-RestMethod -Uri "https://$FQDN/").model
   ```

   **Expected result:** The configured model name is restored to `gpt-5.4-mini`.

3. Optionally, verify the latest revision is active.

   ```powershell
   az containerapp revision list -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP -o table
   ```

You have now reproduced, diagnosed, and fixed an environment-variable configuration issue.

## Task 5: Diagnose and fix an ingress target-port issue

The API listens on port `8000`. Container Apps ingress must route requests to that port. In this task, set the wrong target port, observe the failure, diagnose the mismatch, and restore the correct port.

### Step 5.1: Introduce the problem

1. Change the ingress target port to `3000`.

   ```powershell
   az containerapp ingress update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --target-port 3000
   ```

2. Call the health endpoint.

   ```powershell
   Invoke-RestMethod -Uri "https://$FQDN/health"
   ```

   **Expected result:** The request fails or times out because ingress routes traffic to port `3000`, while the application listens on port `8000`. This failure is intentional for the exercise.

### Step 5.2: Diagnose the cause

1. Inspect the ingress configuration.

   ```powershell
   az containerapp show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --query "properties.configuration.ingress" -o yaml
   ```

   **Expected result:** The `targetPort` is set to `3000`.

2. Review the container logs.

   ```powershell
   az containerapp logs show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP
   ```

   Look for Gunicorn startup messages indicating that the app is listening on port `8000`. The application is running, but ingress is targeting the wrong port.

### Step 5.3: Fix and verify

1. Restore the ingress target port to `8000`.

   ```powershell
   az containerapp ingress update -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --target-port 8000
   ```

2. Call the health endpoint again.

   ```powershell
   Invoke-RestMethod -Uri "https://$FQDN/health"
   ```

   **Expected result:** The endpoint returns `status` as `healthy`.

You have now reproduced, diagnosed, and fixed an ingress configuration issue.

## Task 6: Query Log Analytics for historical troubleshooting

The `az containerapp logs show` command displays recent console output. Historical logs are stored in the Log Analytics workspace associated with the Container Apps environment. Log ingestion can take a few minutes.

### Step 6.1: Retrieve the workspace ID

1. Get the Log Analytics workspace customer ID from the Container Apps environment.

   ```powershell
   $WORKSPACE_ID = az containerapp env show -n $env:ACA_ENVIRONMENT -g $env:RESOURCE_GROUP --query properties.appLogsConfiguration.logAnalyticsConfiguration.customerId -o tsv
   ```

2. Display the workspace ID.

   ```powershell
   Write-Output "Workspace ID: $WORKSPACE_ID"
   ```

### Step 6.2: Query the latest console logs

Run the following Kusto query to display the 20 most recent console log entries, including timestamp and message.

```powershell
az monitor log-analytics query -w $WORKSPACE_ID --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$($env:CONTAINER_APP_NAME)' | project TimeGenerated, Log_s | order by TimeGenerated desc | take 20" -o table
```

**Expected result:** A table of recent log entries. If the query returns no rows, wait a few minutes and run it again.

### Step 6.3: Query error-level messages

Filter the console logs for messages containing the word `error`.

```powershell
az monitor log-analytics query -w $WORKSPACE_ID --analytics-query "ContainerAppConsoleLogs_CL | where ContainerAppName_s == '$($env:CONTAINER_APP_NAME)' and Log_s contains 'error' | order by TimeGenerated desc | take 20" -o table
```

These queries help you investigate past issues even after container restarts or revision changes. Log Analytics data may take **2–5 minutes** to appear.

## Task 7: Troubleshooting reference

### Container app is not responding

1. Check whether a revision is active.

   ```powershell
   az containerapp revision list -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP -o table
   ```

2. Inspect ingress configuration and verify the target port is `8000`.

   ```powershell
   az containerapp show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP --query "properties.configuration.ingress" -o yaml
   ```

3. Check recent console logs.

   ```powershell
   az containerapp logs show -n $env:CONTAINER_APP_NAME -g $env:RESOURCE_GROUP
   ```

### Logs are not appearing in Log Analytics

- `az containerapp logs show` displays recent console output; use Log Analytics for historical data.
- Wait 2–5 minutes for log ingestion, then retry the query.
- Confirm the workspace ID was retrieved from the correct Container Apps environment.

### Environment variable changes do not appear to take effect

- Container Apps creates a new revision when environment variables change. Check the revision list and confirm that the intended revision is active.
- Use `--set-env-vars` carefully: avoid replacing unrelated environment variables. Do not use `--replace-env-vars` unless you intend to replace the complete environment-variable set.

## Task 8: Clean up Azure resources

**Warning:** Deleting a resource group deletes every resource inside it. If you used an existing resource group containing other workloads, those resources will also be deleted.

1. Display the resource group configured for this lab.

   ```powershell
   Write-Output "Resource group to delete: $env:RESOURCE_GROUP"
   ```

2. Confirm that the displayed name is the dedicated lab resource group. Then delete it and its resources.

   ```powershell
   az group delete --name $env:RESOURCE_GROUP --no-wait --yes
   ```

The command starts deletion in the background. You can verify completion in the Azure portal.

## Lab completion checklist

- [ ] Downloaded and extracted `aca-manage-python.zip`.
- [ ] Updated only the resource group and region in `azdeploy.py`.
- [ ] Signed in to the correct Azure subscription.
- [ ] Installed the `containerapp` and `log-analytics` CLI extensions.
- [ ] Registered `Microsoft.App`, `Microsoft.OperationalInsights`, and `Microsoft.ContainerRegistry`.
- [ ] Used deployment-script options `1`, `2`, `3`, then `5`.
- [ ] Loaded `.env.ps1` and verified the default endpoint returns JSON with model name `gpt-5.4-mini`.
- [ ] Removed `MODEL_NAME`, observed `not-configured`, inspected the environment configuration, and restored the variable.
- [ ] Set ingress to port `3000`, observed the failed health request, diagnosed the mismatch, and restored port `8000`.
- [ ] Verified `/health` returns `healthy`.
- [ ] Queried recent and error-level console logs in Log Analytics.
- [ ] Deleted the dedicated lab resource group if cleanup is required.

## Key takeaways

- Container Apps revisions help you identify configuration changes and the revision receiving traffic.
- Environment variables are part of the container app configuration; a missing variable can cause unexpected API behavior.
- Ingress `targetPort` must match the port on which the application listens.
- Console logs help with immediate diagnosis; Log Analytics supports historical troubleshooting.
- Make configuration changes deliberately and verify each fix through the API as a consumer would.
