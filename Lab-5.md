# Lab 5: Configure Autoscaling Using KEDA Triggers

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Azure services:** Azure Container Registry (ACR), Azure Container Apps, Log Analytics  
**Shell:** Windows PowerShell only

## Lab objectives

In this lab, you will deploy a mock agent API to Azure Container Apps (ACA), configure HTTP concurrency autoscaling with KEDA, generate concurrent requests, observe replica changes, and then manage scaling settings through a YAML configuration file.

By the end of the lab, you will have:
- Created ACR and built a container image using ACR Tasks.
- Deployed an Azure Container Apps environment and container app.
- Configured an HTTP concurrency scale rule.
- Generated load with a local Flask dashboard and observed replica scaling.
- Exported the app configuration to YAML, changed the scale settings, and applied the update.
- Verified the final scaling configuration and cleaned up Azure resources.

## Important billing note

The Microsoft Learn lab states that ACR task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan** for this exercise. Azure resources may incur charges until deleted.

## Prerequisites

Before starting, ensure you have:
- An Azure subscription with permission and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- Python 3.12 or later installed.
- Optionally, the YAML extension for Visual Studio Code.

Check that Azure CLI and Python are available in the VS Code PowerShell terminal:

```powershell
az --version
```

```powershell
python --version
```

If either command is not recognized, install or repair the corresponding prerequisite before continuing.

---

## Task 1: Download and prepare the starter project

1. Download the starter ZIP file:

   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aca-scale-python.zip>

2. Move the ZIP file to your preferred working location and extract it into a folder.

3. Open Visual Studio Code. Select **File > Open Folder...** and open the extracted project folder.

4. Open `azdeploy.py`. At the top of the file, change **only** the resource group name and Azure region values, then save the file. Do not change anything else in the script.

   ```python
   "<your-resource-group-name>"  # Resource Group name
   "<your-azure-region>"         # Azure region for the resources
   ```

   Use a valid Azure region available to your subscription. Record the resource group name because you will need it for cleanup.

5. In VS Code, select **Terminal > New Terminal**. Confirm that the terminal is running **PowerShell** and that the current directory is the project root—the folder containing `azdeploy.py`.

## Task 2: Sign in, install CLI extensions, and register providers

1. Sign in to Azure and follow the prompts to select the intended account and subscription.

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

## Task 3: Deploy Azure resources and verify the app

1. Make sure the terminal is in the project root, then run the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. When prompted, enter `1` to select **Create Azure Container Registry and build container image**. Wait for ACR Tasks to build and push the image.

3. When the first operation finishes, enter `2` to select **Create Container Apps environment**. Wait for the environment creation to finish.

4. When the previous operation finishes, enter `3` to select **Create Container App**. Wait for the container app to be deployed.

5. When deployment completes, enter `5` to exit the script.

6. Load the generated environment variables into the current PowerShell session.

   ```powershell
   . .\.env.ps1
   ```

7. Verify that the app endpoint is available.

   ```powershell
   Invoke-RestMethod "$env:CONTAINER_APP_URL/"
   ```

   **Expected result:** The endpoint returns JSON indicating that the app is running.

   Keep this terminal open. If you close it and create a new terminal, load `.env.ps1` again.

## Task 4: Configure HTTP concurrency autoscaling

Azure Container Apps uses KEDA-based scaling rules. This lab configures scaling based on concurrent HTTP requests, which can act as a proxy for requests in progress.

**Note:** Applying configuration updates, including scaling changes, creates a new revision.

1. Configure the HTTP scale rule with a minimum of zero replicas, a maximum of ten replicas, and a concurrency threshold of ten requests.

   ```powershell
   az containerapp update --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --min-replicas 0 --max-replicas 10 --scale-rule-name http-scaling --scale-rule-type http --scale-rule-http-concurrency 10
   ```

2. Verify the scaling configuration.

   ```powershell
   az containerapp show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --query "properties.template.scale" --output json
   ```

   **Expected result:** The output includes the `http-scaling` rule, `minReplicas` set to `0`, and `maxReplicas` set to `10`.

## Task 5: Start the local load-generator dashboard

The starter project includes a local Flask dashboard that generates concurrent requests and displays Container Apps revisions and replica counts.

1. Change to the `client` directory.

   ```powershell
   Set-Location .\client
   ```

2. Create a Python virtual environment for the dashboard.

   ```powershell
   python -m venv .venv
   ```

3. Activate the virtual environment in PowerShell.

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

   If PowerShell blocks activation because of the execution policy, use the following command for the current terminal session, then activate the environment again:

   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```

4. Install the dashboard dependencies.

   ```powershell
   python -m pip install -r requirements.txt
   ```

5. Start the dashboard.

   ```powershell
   python app.py
   ```

6. Open a browser and navigate to:

   <http://127.0.0.1:5000>

7. In the dashboard's left pane, select **Refresh Revisions & Replicas**.

   **Expected result:** You should see one or zero replicas. The app may initially have one running replica. After the workload becomes idle, scaling to zero can take approximately five minutes because the default cooldown period is 300 seconds.

## Task 6: Generate load and observe scale-out

1. In the dashboard's **Load Generator** section, select **Start** to send requests to the container app.

2. Select **Refresh Revisions & Replicas** every 5–10 seconds.

3. Observe the replica count. With sufficient overlapping requests, the number of replicas should increase in response to demand.

4. If the load generator stops, run it again to generate additional traffic and observe replica changes.

5. When you have observed scaling, close the dashboard browser tab and return to the terminal running the Flask app.

6. Press **Ctrl+C** to stop the local dashboard.

7. Return to the project root before proceeding with YAML configuration.

   ```powershell
   Set-Location ..
   ```

   **Important:** The local dashboard must be stopped before you move on. The environment variables loaded in the original PowerShell session remain available.

## Task 7: Configure scaling using YAML

Using YAML makes the scale configuration repeatable and easier to manage when an app has multiple scaling rules.

### Step 7.1: Export the current configuration

1. From the project root, export the container app configuration to `app-config.yaml`.

   ```powershell
   az containerapp show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --output yaml > app-config.yaml
   ```

2. Open `app-config.yaml` in Visual Studio Code.

   ```powershell
   code app-config.yaml
   ```

### Step 7.2: Edit the scale section

In `app-config.yaml`, locate `properties` → `template` → `scale`. Update the scale settings to the following values, preserving the YAML indentation and retaining the existing HTTP scale rule configuration:

```yaml
scale:
  cooldownPeriod: 200
  maxReplicas: 5
  minReplicas: 1
  pollingInterval: 30
```

- `cooldownPeriod: 200` reduces the cooldown period to 200 seconds.
- `maxReplicas: 5` limits the app to five replicas.
- `minReplicas: 1` keeps at least one replica running.
- `pollingInterval: 30` checks the scaling condition every 30 seconds.

Save the file. Do not remove unrelated configuration or the HTTP scale rule. If the exported YAML contains read-only properties such as `id`, `systemData`, or `type` that cause an update error, remove those properties as appropriate before retrying.

### Step 7.3: Apply the YAML configuration

1. Apply the edited YAML to the container app.

   ```powershell
   az containerapp update --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --yaml app-config.yaml
   ```

2. Verify the updated scale settings.

   ```powershell
   az containerapp show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --query "properties.template.scale" --output json
   ```

   **Expected result:** The configuration shows `cooldownPeriod` of `200`, `maxReplicas` of `5`, and `minReplicas` of `1`. The HTTP scaling rule should remain configured.

## Task 8: Troubleshooting

### The app does not scale out under load

1. Verify the configured scale rules.

   ```powershell
   az containerapp show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --query "properties.template.scale" --output json
   ```

2. Ensure that you are generating concurrent requests with the dashboard. Requests must overlap to increase the concurrency count.

3. Increase the dashboard's `delayMs` setting to approximately `500`–`1500` milliseconds so requests overlap and concurrency accumulates.

4. Inspect recent system logs for scaling events.

   ```powershell
   az containerapp logs show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --type system --tail 50
   ```

### The dashboard does not start or cannot list revisions and replicas

- Ensure that the Python virtual environment is activated; the terminal prompt should show `(.venv)`.
- Confirm that dependencies were installed from the `client` directory.

  ```powershell
  python -m pip install -r requirements.txt
  ```

- Confirm that Azure CLI is installed and that you signed in with `az login`.
- Confirm that the Container Apps CLI extension is installed.

  ```powershell
  az extension add --name containerapp
  ```

- If you opened a new PowerShell terminal, return to the project root and load the generated environment variables again.

  ```powershell
  . .\.env.ps1
  ```

### PowerShell virtual environment activation fails

From the `client` directory, use:

```powershell
.\.venv\Scripts\Activate.ps1
```

If script execution is blocked, allow activation for the current PowerShell process and retry:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```

If the activation script is missing, recreate the virtual environment and reinstall the requirements.

### The YAML update fails

- Check YAML indentation and confirm the `scale` section is under `properties` → `template`.
- Confirm that the scale settings are valid and that the existing HTTP scale rule has not been accidentally removed.
- Remove read-only properties such as `id`, `systemData`, and `type` if Azure CLI reports that they cannot be updated.
- Apply the YAML again and re-run the verification command.

## Task 9: Clean up Azure resources

**Warning:** Deleting a resource group deletes every resource inside it. If you selected an existing resource group containing other workloads, those resources will also be deleted.

1. Display the resource group name before deleting it.

   ```powershell
   Write-Output "Resource group to delete: $env:RESOURCE_GROUP"
   ```

2. Confirm that it is the dedicated lab resource group. Then delete it and all resources inside it.

   ```powershell
   az group delete --name $env:RESOURCE_GROUP --no-wait --yes
   ```

The command starts deletion in the background. You can verify completion in the Azure portal.

## Lab completion checklist

- [ ] Downloaded and extracted `aca-scale-python.zip`.
- [ ] Updated only the resource group and region in `azdeploy.py`.
- [ ] Signed in to Azure, installed the `containerapp` and `log-analytics` extensions, and registered the required providers.
- [ ] Used deployment-script options `1`, `2`, `3`, then `5`.
- [ ] Loaded `.env.ps1` and verified the app endpoint.
- [ ] Configured the `http-scaling` rule with minimum replicas `0`, maximum replicas `10`, and HTTP concurrency `10`.
- [ ] Started the local dashboard and generated load.
- [ ] Refreshed revisions and replicas every 5–10 seconds and observed scaling behavior.
- [ ] Exported the app configuration to `app-config.yaml`.
- [ ] Updated `cooldownPeriod` to `200`, `maxReplicas` to `5`, and `minReplicas` to `1`.
- [ ] Applied the YAML and verified the final scale settings.
- [ ] Deleted the dedicated lab resource group if cleanup is required.

## Key takeaways

- KEDA-based HTTP concurrency scaling helps Container Apps scale out when requests overlap and demand increases.
- A minimum replica count of zero can reduce idle costs, but scaling back to zero may be delayed by the cooldown period.
- Revisions are created when container app configuration changes.
- YAML provides a repeatable way to manage scaling parameters.
- Load generation, revision inspection, replica monitoring, and system logs help validate and troubleshoot autoscaling.
