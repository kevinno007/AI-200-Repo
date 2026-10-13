# Lab 3: Deploy a Containerized Backend API to Azure Container Apps

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Azure services:** Azure Container Registry (ACR), Azure Container Apps, Log Analytics  
**Shell:** Windows PowerShell only

## Lab objectives

In this lab, you will deploy a containerized backend API to Azure Container Apps (ACA). You will use a system-assigned managed identity to pull a private image from Azure Container Registry, store a sensitive value as an ACA secret, reference that secret from an environment variable, test the API endpoints, and review application logs.

By the end of the lab, you will have:
- Created an ACR registry and built a container image using ACR Tasks.
- Created an Azure Container Apps environment.
- Deployed a container app with external HTTPS ingress.
- Configured managed-identity authentication to ACR.
- Created an application secret and referenced it from an environment variable.
- Verified the health, configuration, and document-processing endpoints.
- Reviewed container logs and cleaned up the resources.

## Important billing note

Microsoft Learn notes that ACR task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan** for this exercise. Azure resources may incur charges until deleted.

## Prerequisites

Before starting, ensure you have:
- An Azure subscription with permissions and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- Python 3.12 or later installed.
- Permission to sign in to the intended Azure subscription.

Check that the tools are available from the VS Code PowerShell terminal:

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

   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aca-deploy-python.zip>

2. Move the ZIP file to your preferred working location and extract it into a folder.

3. Launch Visual Studio Code. Select **File > Open Folder...** and open the extracted project folder.

4. Open `azdeploy.py`. At the top of the file, change **only** the resource group name and Azure region values, then save the file. Do not change anything else in the script.

   ```python
   "<your-resource-group-name>"  # Resource Group name
   "<your-azure-region>"         # Azure region for the resources
   ```

   Choose a valid region available to your subscription. Record the resource group name; you will need it for cleanup.

5. In VS Code, select **Terminal > New Terminal**. Ensure the terminal is running **PowerShell** and the current directory is the project root—the folder containing `azdeploy.py` and the project files.

## Task 2: Sign in, install Azure CLI extensions, and register providers

1. Sign in to Azure and follow the prompts to choose the correct account and subscription.

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

## Task 3: Deploy the Azure resources

1. Ensure you are in the project root, then run the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. When prompted by the script, enter `1` to select **Create Azure Container Registry and build container image**. This creates ACR and uses ACR Tasks to build and push the container image.

3. Wait for the first operation to finish. Enter `2` to select **Create Container Apps environment**. The environment is required before you can deploy the container app.

4. Wait for the environment creation to finish. The script creates an environment-variable file for the remaining tasks.

5. Enter `4` to exit the deployment script.

6. Load the generated environment variables into the **current PowerShell session**.

   ```powershell
   . .\.env.ps1
   ```

7. Confirm that the key variables are populated.

   ```powershell
   $env:ACR_NAME
   ```

   ```powershell
   $env:ACR_SERVER
   ```

   ```powershell
   $env:CONTAINER_IMAGE
   ```

   ```powershell
   $env:CONTAINER_APP_NAME
   ```

   ```powershell
   $env:RESOURCE_GROUP
   ```

   ```powershell
   $env:ACA_ENVIRONMENT
   ```

   ```powershell
   $env:TARGET_PORT
   ```

   ```powershell
   $env:MODEL_NAME
   ```

   ```powershell
   $env:EMBEDDINGS_API_KEY
   ```

   Keep this terminal open. If you create a new terminal, load `.env.ps1` again. Do not share or publish the value of `EMBEDDINGS_API_KEY`.

## Task 4: Create the container app with managed-identity image pulls

The image is stored in a private registry. The container app must be configured to authenticate to ACR at creation time so its first revision can pull the image. The `--registry-identity system` option enables the system-assigned managed identity for registry access; with ACR, the CLI assigns the `AcrPull` role.

Run this command as a single line:

```powershell
az containerapp create --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --environment $env:ACA_ENVIRONMENT --image "$($env:ACR_SERVER)/$($env:CONTAINER_IMAGE)" --ingress external --target-port $env:TARGET_PORT --env-vars "MODEL_NAME=$($env:MODEL_NAME)" --registry-server $env:ACR_SERVER --registry-identity system
```

Wait for the command to complete successfully. The app is created with external ingress, so it can be reached through its generated HTTPS fully qualified domain name (FQDN).

## Task 5: Create a secret and reference it from an environment variable

Applications should not hard-code API keys in source code or ordinary environment-variable values. Azure Container Apps secrets provide a safer way to store sensitive configuration.

1. Create the `embeddings-api-key` secret from the environment variable generated by the starter script.

   ```powershell
   az containerapp secret set --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --secrets "embeddings-api-key=$($env:EMBEDDINGS_API_KEY)"
   ```

2. Reference the secret from the `EMBEDDINGS_API_KEY` environment variable.

   ```powershell
   az containerapp update --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --set-env-vars "EMBEDDINGS_API_KEY=secretref:embeddings-api-key"
   ```

   This update creates a new revision, restarting the app so the secret reference takes effect.

3. List revisions to confirm a new revision was created.

   ```powershell
   az containerapp revision list --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --output table
   ```

   **Expected result:** The latest revision name ends with a suffix such as `--0000002`, indicating a subsequent revision. Container Apps creates revisions when you change environment variables or secrets. Older inactive revisions may be pruned over time.

## Task 6: Retrieve the app URL and verify health

1. Retrieve the container app's FQDN and store it in a PowerShell variable.

   ```powershell
   $FQDN = az containerapp show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --query properties.configuration.ingress.fqdn --output tsv
   ```

2. Display the FQDN.

   ```powershell
   Write-Output $FQDN
   ```

3. Call the health endpoint.

   ```powershell
   Invoke-RestMethod -Uri "https://$FQDN/health"
   ```

   **Expected result:** The endpoint returns a response with `status` set to `healthy`.

4. If the request fails immediately after deployment, wait a minute and try again. The first startup may take longer while Container Apps pulls the image and starts the container.

## Task 7: Verify app configuration and secret reference

Call the root endpoint to verify that the app reports its configuration, including the configured model name and whether the API key is configured.

```powershell
Invoke-RestMethod -Uri "https://$FQDN/"
```

**Expected result:** The JSON response contains app information, the configured model name, and an indication that the API key is configured. The secret value itself should not be exposed in the response.

If the response indicates that the key is not configured, review Task 5 and the troubleshooting section.

## Task 8: Test document processing

1. Confirm that `document.txt` is present in the project root.

2. Send the document to the `/process` endpoint. The body is sent as plain text.

   ```powershell
   Invoke-RestMethod -Uri "https://$FQDN/process" -Method Post -ContentType "text/plain" -Body (Get-Content -Raw -Path "document.txt") | ConvertTo-Json -Depth 10
   ```

   **Expected result:** The API returns JSON containing mock document-analysis information.

3. Review the response for the result of the processing request. If the request fails, verify that the FQDN was retrieved correctly, that the app is healthy, and that `document.txt` exists in the current directory.

## Task 9: Review container logs

1. Display recent console output from the container app.

   ```powershell
   az containerapp logs show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP
   ```

2. Look for Gunicorn startup messages showing that workers started and are listening on port `8000`. You should also see HTTP request logs for calls such as `GET /health`, `GET /`, and `POST /process`.

3. To follow the logs while reproducing a request, run:

   ```powershell
   az containerapp logs show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --follow
   ```

4. While log following is active, call the health endpoint in another terminal or refresh it in a browser. Press **Ctrl+C** to stop following logs.

   **Note:** The command displays recent console output. Historical logs and advanced troubleshooting information are stored in the Log Analytics workspace associated with the Container Apps environment.

## Task 10: Troubleshooting

### Check deployment status with the script

1. From the project root, run the deployment script again.

   ```powershell
   python azdeploy.py
   ```

2. Select option `3` to check the status of ACR and the Container Apps environment. This helps verify that the base infrastructure is deployed and that the container image exists.

3. When finished, select option `4` to exit the script.

### Verify Azure authentication and environment variables

1. Check the signed-in subscription.

   ```powershell
   az account show --output table
   ```

2. Check the registry name.

   ```powershell
   $env:ACR_NAME
   ```

3. If environment variables are empty, return to the project root and load them again.

   ```powershell
   . .\.env.ps1
   ```

### Verify ACR deployment

1. In the [Azure portal](https://portal.azure.com), open the resource group and confirm that the Azure Container Registry exists and its provisioning state is **Succeeded**.

2. List registries accessible in the selected subscription.

   ```powershell
   az acr list --output table
   ```

### Troubleshoot ACR build failures

1. Confirm that you ran the deployment script from the project root.

2. List recent ACR task runs.

   ```powershell
   az acr task list-runs --registry $env:ACR_NAME --output table
   ```

3. Copy a run ID from the output and view its logs.

   ```powershell
   az acr task logs --registry $env:ACR_NAME --run-id "<run-id>"
   ```

   Replace `<run-id>` with a real run ID from the previous command.

### Troubleshoot container pull failures

If you see `ImagePullBackOff`, an unauthorized error, or HTTP 403:

1. Confirm that the container app has a system-assigned managed identity.

   ```powershell
   az containerapp identity show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP
   ```

2. Confirm that the registry exists and is accessible.

   ```powershell
   az acr show --name $env:ACR_NAME --resource-group $env:RESOURCE_GROUP --output table
   ```

3. Remember that role assignments can take a minute or two to propagate. If the app was just created, wait briefly and check again.

### Troubleshoot container startup and application errors

1. Follow the container logs.

   ```powershell
   az containerapp logs show --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --follow
   ```

2. If the app returns HTTP 502 or 503 shortly after deployment, wait a minute and retry.

3. Check revision status for provisioning errors.

   ```powershell
   az containerapp revision list --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --output table
   ```

### Troubleshoot secret configuration

1. List configured secrets. This lists secret metadata; do not share secret values.

   ```powershell
   az containerapp secret list --name $env:CONTAINER_APP_NAME --resource-group $env:RESOURCE_GROUP --output table
   ```

2. Call the root endpoint again to check whether the API key is configured.

   ```powershell
   Invoke-RestMethod -Uri "https://$FQDN/"
   ```

3. If the key is not configured, repeat the secret creation and environment-variable reference steps in Task 5. Do not paste the API key into screenshots, logs, or shared training materials.

## Task 11: Clean up Azure resources

**Warning:** Deleting a resource group deletes every resource inside it. If you used an existing resource group that contains other workloads, those resources will also be deleted. Verify the resource group name carefully before proceeding.

1. Display the resource group that will be deleted.

   ```powershell
   Write-Host "Resource group to delete: $env:RESOURCE_GROUP"
   ```

2. If it is the dedicated lab resource group and you are ready to remove all resources, run:

   ```powershell
   az group delete --name $env:RESOURCE_GROUP --no-wait --yes
   ```

   The command starts deletion in the background. Check the Azure portal if you need to confirm that deletion has completed.

## Lab completion checklist

- [ ] Downloaded and extracted the starter project.
- [ ] Updated only the resource group and region in `azdeploy.py`.
- [ ] Signed in to the correct Azure subscription.
- [ ] Installed the `containerapp` and `log-analytics` CLI extensions.
- [ ] Registered the `Microsoft.App`, `Microsoft.OperationalInsights`, and `Microsoft.ContainerRegistry` providers.
- [ ] Used deployment-script option `1` to create ACR and build the image.
- [ ] Used option `2` to create the Container Apps environment and option `4` to exit.
- [ ] Loaded `.env.ps1` and verified the required environment variables.
- [ ] Created the container app with external ingress and managed-identity ACR authentication.
- [ ] Created the `embeddings-api-key` secret and referenced it through `secretref`.
- [ ] Verified that a new revision was created.
- [ ] Retrieved the FQDN and tested `/health`, `/`, and `/process`.
- [ ] Reviewed container logs and used troubleshooting checks if required.
- [ ] Deleted the dedicated lab resource group if cleanup is required.

## Key takeaways

- Azure Container Apps can host containerized APIs with externally accessible HTTPS ingress.
- A system-assigned managed identity enables private ACR image pulls without storing registry credentials in the app.
- Container Apps secrets can be referenced by environment variables using `secretref`.
- Environment-variable and secret changes create new revisions.
- Health endpoints, API responses, revision status, and container logs provide practical ways to validate and troubleshoot deployments.
