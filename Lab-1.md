# Lab 1: Build and Run a Container Image with Azure Container Registry Tasks

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Technology:** Azure Container Registry (ACR), ACR Tasks, Azure CLI, Python, Visual Studio Code  
**Command shell:** Windows PowerShell only

## Lab objective

In this lab, you will use Azure Container Registry Tasks to build a container image in Azure, verify the image and its tags, run a command inside the image, create a second image version, inspect build history, and lock a production image. You do **not** need Docker installed locally because the image build runs in Azure.

## Important billing note

The Microsoft Learn lab states that ACR task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan** for this exercise. Azure resources may incur charges until you delete them.

## Prerequisites

Before starting, make sure you have:

- An Azure subscription with permission and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- Python 3.12 or later installed.
- An internet connection.
- Access to sign in to the intended Azure subscription.

Check that the Azure CLI and Python are available from the VS Code PowerShell terminal:

```powershell
az --version
```

```powershell
python --version
```

If either command is not recognized, install or repair the corresponding prerequisite before continuing.

---

## Task 1: Download and prepare the starter project

1. Open the starter ZIP download link in your browser:  
   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/acr-tasks-python.zip>

2. Save the ZIP file, move it to your preferred working location, and extract it into a folder.

3. Open Visual Studio Code.

4. Select **File > Open Folder...** and open the extracted project folder.

5. In VS Code, open `azdeploy.py`.

6. At the top of `azdeploy.py`, update **only** the resource group name and Azure region values to values suitable for your subscription. Save the file. Do not modify the rest of the script.

   ```python
   "<your-resource-group-name>"  # Resource Group name
   "<your-azure-region>"         # Azure region for the resources
   ```

   Choose a resource group name that you can identify easily, and use a valid Azure region available to your subscription. Remember the resource group name because you will need it during cleanup.

7. In VS Code, select **Terminal > New Terminal**. Confirm that the terminal is running **PowerShell** and that its current directory is the project root—the folder containing `azdeploy.py` and the `api` folder.

## Task 2: Sign in and deploy Azure resources

1. Sign in to Azure. Follow the prompts to select the correct account and subscription.

   ```powershell
   az login
   ```

2. Confirm the active subscription and account.

   ```powershell
   az account show --output table
   ```

   If the wrong subscription is active, select the intended subscription before continuing:

   ```powershell
   az account set --subscription "<your-subscription-name-or-id>"
   ```

3. Register the Azure Container Registry resource provider.

   ```powershell
   az provider register --namespace Microsoft.ContainerRegistry
   ```

4. From the project root, run the deployment script. It deploys the Azure Container Registry and creates environment-variable files used in the remaining tasks.

   ```powershell
   python azdeploy.py
   ```

5. Wait for the script to finish. If deployment fails, read the error output and resolve the issue before proceeding.

6. Load the environment variables into the **current PowerShell session**.

   ```powershell
   . .\.env.ps1
   ```

7. Verify that the registry name was loaded.

   ```powershell
   $env:ACR_NAME
   ```

   The command should display the ACR name. If it is blank, confirm that `.env.ps1` exists in the project root and run the dot-source command again. Keep this terminal open; a new terminal session may require you to load the environment variables again.

## Task 3: Build the first image with ACR Tasks

1. Confirm that you are still in the project root and that the `api` folder is present.

2. Build the image and push it to ACR with the tag `v1.0.0`.

   ```powershell
   az acr build --registry $env:ACR_NAME --image inference-api:v1.0.0 ./api
   ```

3. Watch the output. ACR Tasks should upload the source context, queue and run the cloud build, display build-layer output, push the resulting image to your registry, and report the image digest and task status.

**Expected result:** The build completes successfully and the image `inference-api:v1.0.0` is pushed to your ACR.

## Task 4: Verify the image in ACR

1. List the repositories in your registry.

   ```powershell
   az acr repository list --name $env:ACR_NAME --output table
   ```

   **Expected result:** The list includes `inference-api`.

2. List the tags for the `inference-api` repository.

   ```powershell
   az acr repository show-tags --name $env:ACR_NAME --repository inference-api --output table
   ```

   **Expected result:** The output includes `v1.0.0`.

3. View manifest metadata, including the image digest.

   ```powershell
   az acr manifest list-metadata --registry $env:ACR_NAME --name inference-api --output table
   ```

   Note the digest. A SHA-256 digest identifies the image content independently of the tag assigned to it.

## Task 5: Run a validation command inside the image

Use `az acr run` to execute a Python command inside the image and verify that the Flask application can be imported.

```powershell
az acr run --registry $env:ACR_NAME --cmd "$env:ACR_NAME.azurecr.io/inference-api:v1.0.0 python -c 'from app import app'" /dev/null
```

**Expected result:** ACR pulls the image and runs the command. The task should finish with a successful run status. This confirms that the container starts sufficiently to import the Flask application without errors.

If the command fails, review the task output for the underlying error before continuing.

## Task 6: Build and verify a second image version

1. Build the same application with a new tag, `v1.1.0`.

   ```powershell
   az acr build --registry $env:ACR_NAME --image inference-api:v1.1.0 ./api
   ```

2. List the tags again.

   ```powershell
   az acr repository show-tags --name $env:ACR_NAME --repository inference-api --output table
   ```

   **Expected result:** Both `v1.0.0` and `v1.1.0` appear. This demonstrates how a registry can retain multiple versions of an application image.

## Task 7: Review build history and protect the production image

1. List recent ACR task runs.

   ```powershell
   az acr task list-runs --registry $env:ACR_NAME --output table
   ```

   Review the run IDs, statuses, trigger types, and durations. The run history can help you track builds and troubleshoot failures.

2. Lock the `v1.0.0` image by disabling writes to it.

   ```powershell
   az acr repository update --name $env:ACR_NAME --image inference-api:v1.0.0 --write-enabled false
   ```

   This protects the image from accidental modification or deletion. Do this only after you have completed the earlier validation steps.

3. Verify the image configuration.

   ```powershell
   az acr repository show --name $env:ACR_NAME --image inference-api:v1.0.0
   ```

   **Expected result:** The image details show `writeEnabled` as `false`. The image is now write-locked. Keep in mind that a locked image must be made writable again before you can modify or delete it.

## Task 8: Troubleshooting

### Azure sign-in or subscription problems

1. Confirm the signed-in account and active subscription.

   ```powershell
   az account show --output table
   ```

2. If needed, sign in again and select the correct subscription.

   ```powershell
   az login
   ```

   ```powershell
   az account set --subscription "<your-subscription-name-or-id>"
   ```

### Environment variable is empty

1. Check the registry name.

   ```powershell
   $env:ACR_NAME
   ```

2. If it is empty, return to the project root and load the generated environment file into the current terminal session.

   ```powershell
   . .\.env.ps1
   ```

### Verify ACR deployment

1. In the [Azure portal](https://portal.azure.com), open the resource group you configured and confirm that the container registry exists and its provisioning state is **Succeeded**.

2. List the registries accessible to the current subscription.

   ```powershell
   az acr list --output table
   ```

### Build failures

- Read the ACR build output for errors.
- Confirm that the `api` folder and its Dockerfile are present and that you are running the command from the project root.
- Review recent task runs.

  ```powershell
  az acr task list-runs --registry $env:ACR_NAME --output table
  ```

- Confirm that the registry name is populated and that the registry is accessible in the selected subscription.
- If ACR Tasks cannot run under your subscription's free credits, use a Pay-As-You-Go subscription or another paid plan, as noted at the beginning of the lab.

## Task 9: Clean up Azure resources

**Important:** Deleting a resource group deletes **all resources inside that resource group**. If you used an existing resource group that contains other workloads, those resources will also be deleted. Verify the resource group name carefully before running the command.

Replace `your-resource-group-name` with the exact resource group name you entered in `azdeploy.py`.

```powershell
az group delete --name "your-resource-group-name" --no-wait --yes
```

The command starts the resource-group deletion without waiting for it to finish. You can check the deletion status in the Azure portal.

## Lab completion checklist

- [ ] Downloaded and extracted the starter project.
- [ ] Updated only the resource group and region values in `azdeploy.py`.
- [ ] Signed in to the correct Azure subscription.
- [ ] Registered the `Microsoft.ContainerRegistry` provider.
- [ ] Deployed ACR and loaded `.env.ps1`.
- [ ] Built and verified `inference-api:v1.0.0`.
- [ ] Successfully ran the Flask import validation inside the image.
- [ ] Built `inference-api:v1.1.0` and verified both tags.
- [ ] Reviewed ACR task history.
- [ ] Locked `v1.0.0` and verified `writeEnabled` is `false`.
- [ ] Deleted the lab resource group, if cleanup is required.

## Key takeaways

- **ACR Tasks** builds container images in Azure, so a local Docker installation is not required for this workflow.
- ACR repositories can store multiple image versions through tags.
- Image digests identify image content independently of tags.
- ACR task run history supports build monitoring and troubleshooting.
- Write-locking a validated image helps protect a production version from accidental changes.
