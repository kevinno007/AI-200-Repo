# Lab 2: Deploy a Container to Azure App Service

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Services:** Azure Container Registry (ACR), Azure Container Apps? No — Azure App Service for Linux containers, managed identity, Microsoft Entra role-based access control  
**Shell:** Windows PowerShell only

## Lab objectives

In this lab, you will deploy a containerized document-processing API to Azure App Service. The app pulls its image from a private Azure Container Registry (ACR) using a system-assigned managed identity and the `AcrPull` role. You will configure runtime settings and logging, test the API, inspect logs and settings, and clean up the resources.

By the end of the lab, you will have:
- Deployed an ACR registry and built a container image with ACR Tasks.
- Created a Linux App Service plan and a web app for containers.
- Configured managed identity authentication to a private ACR without registry credentials.
- Enabled persistent storage, Always On, and container logging.
- Tested document processing and confirmed that processed documents are listed.
- Inspected container logs, the SCM (Kudu) site, and application settings.

## Important billing note

The Microsoft Learn lab notes that ACR task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan** for this exercise. Azure resources may incur charges until you delete them.

## Prerequisites

Before starting, ensure you have:
- An Azure subscription with permissions and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- Python 3.12 or later installed.
- Access to sign in to the intended Azure subscription.

Check the tools in the VS Code PowerShell terminal:

```powershell
az --version
```

```powershell
python --version
```

If either command is not recognized, install or repair the corresponding prerequisite before continuing.

---

## Task 1: Download and prepare the starter project

1. Download the starter ZIP file from:  
   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/app-svc-container-python.zip>

2. Move the ZIP file to your preferred working location and extract it into a folder.

3. Open Visual Studio Code, select **File > Open Folder...**, and open the extracted project folder.

4. Open `azdeploy.py`. At the top of the file, change **only** the resource group name and Azure region values, then save the file. Do not change anything else in the script.

   ```python
   "<your-resource-group-name>"  # Resource Group name
   "<your-azure-region>"         # Azure region for the resources
   ```

   Use a valid region available to your subscription. Record the resource group name; you will need it for cleanup.

5. In VS Code, select **Terminal > New Terminal**. Confirm that the terminal is **PowerShell** and that the current directory is the project root—the folder containing `azdeploy.py` and the project files.

## Task 2: Sign in and register resource providers

1. Sign in to Azure and follow the prompts to choose the intended account and subscription.

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

4. Register the Azure Container Registry provider.

   ```powershell
   az provider register --namespace Microsoft.ContainerRegistry
   ```

5. Register the Azure App Service provider.

   ```powershell
   az provider register --namespace Microsoft.Web
   ```

## Task 3: Deploy ACR, build the image, and create an App Service plan

1. Ensure the terminal is in the project root, then start the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. When prompted, enter `1` to select **Create Azure Container Registry and build container image**. Wait for the operation to complete. This creates ACR with its admin account disabled and uses ACR Tasks to build and push the `docprocessor:v1` image.

3. When the previous operation finishes, enter `2` to select **Create App Service Plan**. Wait for the plan creation to complete.

4. When the previous operation finishes, enter `4` to exit the deployment script.

5. Load the generated environment variables into the current PowerShell session.

   ```powershell
   . .\.env.ps1
   ```

6. Confirm that the registry name and other variables are populated.

   ```powershell
   $env:ACR_NAME
   ```

   ```powershell
   $env:RESOURCE_GROUP
   ```

   ```powershell
   $env:APP_PLAN
   ```

   ```powershell
   $env:APP_NAME
   ```

   **Important:** Keep this terminal open. If you open a new terminal, load `.env.ps1` again. If any variable is empty, confirm that the file exists in the project root and dot-source it again.

## Task 4: Create the web app and enable managed identity

1. Create the Linux container web app using the image built in ACR.

   ```powershell
   az webapp create --resource-group $env:RESOURCE_GROUP --plan $env:APP_PLAN --name $env:APP_NAME --container-image-name "$($env:ACR_NAME).azurecr.io/docprocessor:v1"
   ```

2. Enable a system-assigned managed identity for the web app.

   ```powershell
   az webapp identity assign --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME
   ```

   A system-assigned managed identity gives the web app an Azure-managed identity that can be granted access to other Azure resources. This avoids storing registry credentials in app settings.

## Task 5: Grant ACR pull permission to the web app

The ACR registry is private. The web app needs permission to pull its image. Grant the `AcrPull` role to the web app's managed identity, scoped to the registry.

1. Retrieve the web app's managed identity principal ID.

   ```powershell
   $PRINCIPAL_ID = az webapp identity show --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --query principalId --output tsv
   ```

2. Retrieve the ACR resource ID.

   ```powershell
   $ACR_ID = az acr show --resource-group $env:RESOURCE_GROUP --name $env:ACR_NAME --query id --output tsv
   ```

3. Assign the `AcrPull` role at the registry scope.

   ```powershell
   az role assignment create --assignee $PRINCIPAL_ID --scope $ACR_ID --role AcrPull
   ```

   **Important:** Role assignments can take a minute or two to propagate. If the app cannot pull the image immediately, wait briefly before troubleshooting.

4. Configure App Service to use the system-assigned identity for ACR authentication.

   ```powershell
   az webapp config set --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --acr-use-identity true --acr-identity '[system]'
   ```

5. Set the image name and registry URL explicitly.

   ```powershell
   az webapp config container set --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --container-image-name "$($env:ACR_NAME).azurecr.io/docprocessor:v1" --container-registry-url "https://$($env:ACR_NAME).azurecr.io"
   ```

   **Expected behavior:** This command may display a failed credential lookup. It is expected because the ACR admin account is disabled and the app uses managed identity. **Do not enable the ACR admin account or add registry credentials.**

## Task 6: Configure runtime settings and container logging

Run each command separately.

1. Set the container port to `80`.

   ```powershell
   az webapp config appsettings set --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --settings WEBSITES_PORT=80
   ```

2. Enable App Service storage for persistent processed-document data.

   ```powershell
   az webapp config appsettings set --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
   ```

3. Enable Always On to help reduce cold-start latency.

   ```powershell
   az webapp config set --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --always-on true
   ```

4. Enable container logging to the filesystem.

   ```powershell
   az webapp log config --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --docker-container-logging filesystem
   ```

## Task 7: Verify the deployment

1. Retrieve the app's default host name.

   ```powershell
   $APP_URL = az webapp show --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --query defaultHostName --output tsv
   ```

2. Print the application URL.

   ```powershell
   Write-Host "Application URL: https://$APP_URL"
   ```

3. Open the printed HTTPS URL in a browser. Leave the browser open because you will use the application again in this lab. The app should return a response indicating that it is running.

   **Note:** The first request may take longer while App Service pulls the image and starts the container. If the app briefly returns HTTP 502 or 503, wait a minute and try again.

## Task 8: Test the document-processing API

Run these commands from the project root, where `document.txt` is located.

1. Submit `document.txt` to the `/process` endpoint. The command reads the file as raw text and sends it as `text/plain`.

   ```powershell
   $body = Get-Content -Raw -Path "document.txt"; Invoke-RestMethod -Method Post -Uri "https://$APP_URL/process" -ContentType "text/plain" -Body $body | ConvertTo-Json -Depth 10
   ```

   The API returns mock analysis results, including extracted entities, key phrases, and sentiment analysis. Check the response for an indication that the result was saved to persistent storage.

2. List the processed documents.

   ```powershell
   Invoke-RestMethod -Uri "https://$APP_URL/documents" | ConvertTo-Json -Depth 10
   ```

   **Expected result:** The document you just processed appears in the list. This verifies that the processing endpoint works and that persistent storage is configured.

## Task 9: Stream container logs

1. Start a real-time log stream.

   ```powershell
   az webapp log tail --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME
   ```

2. While the stream is running, refresh the application in the browser or send another API request. Observe the log entries.

3. Press **Ctrl+C** to stop streaming.

If no logs appear, confirm that container logging was enabled in Task 6 and review the troubleshooting section below.

## Task 10: Inspect the diagnostic console (SCM/Kudu)

1. Print the SCM (Kudu) URL.

   ```powershell
   Write-Host "Kudu URL: https://$($env:APP_NAME).scm.azurewebsites.net"
   ```

2. Open the printed URL in a browser and sign in if prompted.

3. In the left navigation pane:
   - Open **Environment** to view environment variables and verify app settings.
   - Expand **SSH**, then select **Kudu** to open a browser-based shell.
   - Open **File Manager**, then navigate to `/home/LogFiles/` to inspect log files.
   - Optionally, expand **Logs** and select **Log Stream** to view logs in the browser.
   - You can also use the **Application** entry under SSH to connect to the app container.

   **Note:** The SCM site is separate from the application container. It does not provide a complete view of the container's filesystem or running processes.

## Task 11: Verify application settings

1. List the web app's application settings.

   ```powershell
   az webapp config appsettings list --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --output table
   ```

2. Confirm that `WEBSITES_PORT` is set to `80` and `WEBSITES_ENABLE_APP_SERVICE_STORAGE` is set to `true`, along with any system-provided settings.

3. Verify that managed-identity authentication for ACR is enabled.

   ```powershell
   az webapp config show --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --query acrUseManagedIdentityCreds --output tsv
   ```

   **Expected result:** `true`.

## Task 12: Troubleshooting

### Azure authentication or subscription issues

1. Check the current account and subscription.

   ```powershell
   az account show --output table
   ```

2. If needed, sign in again.

   ```powershell
   az login
   ```

3. Select the correct subscription.

   ```powershell
   az account set --subscription "<your-subscription-name-or-id>"
   ```

### Environment variables are empty

1. Check the registry name.

   ```powershell
   $env:ACR_NAME
   ```

2. Reload the environment variables from the project root.

   ```powershell
   . .\.env.ps1
   ```

### Verify ACR deployment

1. In the [Azure portal](https://portal.azure.com), open your resource group and confirm that the ACR resource exists and its provisioning state is **Succeeded**.

2. List registries available in the selected subscription.

   ```powershell
   az acr list --output table
   ```

### ACR build failures

1. Confirm you ran `python azdeploy.py` from the project root and that the project files are present.

2. List recent ACR task runs.

   ```powershell
   az acr task list-runs --registry $env:ACR_NAME --output table
   ```

3. Copy a run ID from the output and view that run's logs.

   ```powershell
   az acr task logs --registry $env:ACR_NAME --run-id "<run-id>"
   ```

   Replace `<run-id>` with an actual run ID from the previous command.

### “No credential was provided to access Azure Container Registry”

This message may appear during container configuration. It is expected because the ACR admin account is disabled by design and the app uses managed identity. Do **not** enable the admin account or add registry credentials.

Verify the managed-identity configuration:

```powershell
az webapp config show --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --query acrUseManagedIdentityCreds --output tsv
```

The command should return `true`.

### Image pull failures (`ImagePullBackOff`, unauthorized, or HTTP 403)

- Confirm that the web app's system-assigned identity is enabled.

  ```powershell
  az webapp identity show --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME
  ```

- Confirm that the `AcrPull` role assignment is scoped to the ACR resource.

  ```powershell
  az role assignment list --assignee $PRINCIPAL_ID --scope $ACR_ID --output table
  ```

- Remember that role assignments can take a minute or two to propagate.
- Re-run the container configuration command from Task 5 if the image name or registry URL is incorrect.

### Container startup or application errors

1. Confirm container logging is enabled and stream the logs.

   ```powershell
   az webapp log tail --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME
   ```

2. If the app returns HTTP 502 or 503 just after deployment, wait a minute and try again. The first startup may take longer while App Service pulls and starts the image.

### Persistent storage does not appear to work

1. Check that the storage setting exists and is `true`.

   ```powershell
   az webapp config appsettings list --resource-group $env:RESOURCE_GROUP --name $env:APP_NAME --query "[?name=='WEBSITES_ENABLE_APP_SERVICE_STORAGE'].{Name:name,Value:value}" --output table
   ```

2. Submit `document.txt` to `/process` again (Task 8), then query `/documents` and confirm the processed document appears.

## Task 13: Clean up Azure resources

**Warning:** Deleting a resource group deletes every resource in it. If you selected an existing resource group that contains other workloads, those resources will also be deleted. Confirm the resource group name before running the command.

1. Verify the resource group name.

   ```powershell
   Write-Host "Resource group to delete: $env:RESOURCE_GROUP"
   ```

2. If the displayed name is the dedicated lab resource group and you are ready to remove all its resources, run:

   ```powershell
   az group delete --name $env:RESOURCE_GROUP --no-wait --yes
   ```

   The command starts deletion in the background. You can check the resource group in the Azure portal to confirm that cleanup has completed.

## Lab completion checklist

- [ ] Downloaded and extracted the starter project.
- [ ] Updated only the resource group and region in `azdeploy.py`.
- [ ] Signed in to the correct subscription and registered `Microsoft.ContainerRegistry` and `Microsoft.Web`.
- [ ] Used menu option `1` to create ACR and build the image.
- [ ] Used menu option `2` to create the App Service plan, then option `4` to exit.
- [ ] Loaded `.env.ps1` and verified the environment variables.
- [ ] Created the web app and enabled its system-assigned managed identity.
- [ ] Assigned `AcrPull` at the ACR scope and configured managed-identity image pulls.
- [ ] Configured port `80`, persistent storage, Always On, and container logging.
- [ ] Opened the application URL and tested `/process` and `/documents`.
- [ ] Streamed logs, inspected Kudu, and checked app settings.
- [ ] Verified `acrUseManagedIdentityCreds` is `true`.
- [ ] Deleted the dedicated lab resource group if cleanup is required.

## Key takeaways

- Azure App Service can host a Linux container image stored in a private ACR.
- A system-assigned managed identity plus the `AcrPull` role enables registry access without stored registry credentials.
- App Service app settings control container port, persistent storage, and runtime behavior.
- Container logs and the SCM/Kudu site help diagnose deployment and runtime issues.
- Always verify the resource group before deleting it.
