# Lab 7: Configure an App on Azure Kubernetes Service

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Azure services:** Azure Container Registry (ACR), Azure Kubernetes Service (AKS)  
**Tools:** Azure CLI, `kubectl`, Python, Visual Studio Code  
**Shell:** Windows PowerShell only

## Lab objectives

In this lab, you will deploy a containerized API to AKS and configure it using Kubernetes resources:

- **ConfigMap** for non-sensitive application settings.
- **Secret** for sensitive configuration values.
- **PersistentVolumeClaim (PVC)** for persistent API log storage.
- **Deployment** for running the API container.
- **LoadBalancer Service** for exposing the API outside the cluster.

You will then use a Python client to test the API, inspect configuration and secret status, retrieve mock products, and view the API log summary.

## Important notes

- Microsoft Learn estimates **10–15 minutes** for Azure resource deployment and approximately **30 minutes** for the complete lab.
- The source lab notes that ACR task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan**.
- The PVC and persistent log storage in this exercise are for demonstration. For production logging, use a centralized service such as Azure Monitor or Application Insights. If persistent log storage is used, implement log rotation to avoid filling the volume and causing container failures or pod evictions.
- Azure resources may incur charges until you delete the lab resource group.

## Prerequisites

Before starting, ensure you have:
- An Azure subscription with permission and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- `kubectl` installed.
- Python 3.12 or later installed.
- Optional: the YAML extension for Visual Studio Code.

Check the tools from the VS Code PowerShell terminal:

```powershell
az --version
```

```powershell
kubectl version --client
```

```powershell
python --version
```

If a command is not recognized, install or repair the corresponding prerequisite before continuing.

---

## Task 1: Download and prepare the starter project

1. Download the starter ZIP file:

   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aks-configure-python.zip>

2. Move the ZIP file to your preferred working location and extract it into a folder.

3. Open Visual Studio Code. Select **File > Open Folder...** and open the extracted project folder.

4. Open `azdeploy.py`. At the top of the script, update only the resource group name and Azure region, then save the file. Do not change anything else in the script.

   ```python
   rg = "<your-resource-group-name>"  # Resource Group name
   location = "<your-azure-region>"   # Azure region for the resources
   ```

   Record the resource group name because you will need it during cleanup.

5. In VS Code, select **Terminal > New Terminal**. Ensure the terminal is running **PowerShell** and that the current directory is the project root—the folder containing `azdeploy.py`.

## Task 2: Sign in and register Azure resource providers

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

4. Register the AKS provider.

   ```powershell
   az provider register --namespace Microsoft.ContainerService
   ```

5. Register the ACR provider.

   ```powershell
   az provider register --namespace Microsoft.ContainerRegistry
   ```

6. Register the compute provider.

   ```powershell
   az provider register --namespace Microsoft.Compute
   ```

7. Register the network provider.

   ```powershell
   az provider register --namespace Microsoft.Network
   ```

8. Register the storage provider.

   ```powershell
   az provider register --namespace Microsoft.Storage
   ```

These providers support AKS, ACR, and AKS dependencies. Registering them explicitly can avoid occasional cluster-creation failures on new subscriptions.

## Task 3: Deploy the Azure resources

1. From the project root, start the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. Follow the menu in this exact order. Wait for each operation to finish before selecting the next option.

   1. Enter `1` — **Create Azure Container Registry (ACR)**. When complete, copy the ACR endpoint displayed by the script; you will need it in Task 6.
   2. Enter `2` — **Build and push API image to ACR**. This uses ACR Tasks and can take approximately 3–5 minutes.
   3. Enter `3` — **Create AKS cluster**. The cluster is configured with a managed identity, permission to pull images from ACR, and the RBAC role needed to write to persistent storage. This can take approximately 5–10 minutes.
   4. Enter `4` — **Get AKS credentials for kubectl**. This runs `az aks get-credentials` and configures `kubectl`.
   5. Enter `5` — **Check deployment status**. Confirm that ACR and AKS were deployed successfully.
   6. If all resources report success, enter `7` to exit the deployment script.

**If AKS deployment fails:**

- Run option `5` to inspect the reported state and error.
- If the error indicates `Standard_D2s_v7` is unavailable or the region lacks capacity, exit with option `7`, update `AKS_VM_SIZE` to one of the v5/v6 fallback sizes listed near the top of `azdeploy.py`, or change the `location` value to another suitable region. Then rerun the script and select option `3`.
- If the error reports insufficient quota, use a region where the subscription has available Dsv5-family quota or request a quota increase.
- If option `5` reports that the AKS cluster is `Failed` or `Canceled`, correct the underlying issue and select option `6` to delete the failed AKS deployment before retrying option `3`. This guarded option does not delete a healthy or in-progress cluster.
- If no AKS resource was created, there is no cluster to delete.

Do not continue until ACR and AKS are successfully provisioned and `kubectl` credentials are configured.

## Task 4: Create the ConfigMap

A ConfigMap stores non-sensitive configuration as key-value pairs. In this lab, it stores the student name, API version, and log path.

1. In VS Code, open `k8s/configmap.yaml`.

2. Add the following content to the file. You may replace `YourNameHere` with your name.

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: api-config
     labels:
       app: aks-config-api
   data:
     # Non-sensitive application configuration
     STUDENT_NAME: "YourNameHere"
     API_VERSION: "1.0.0"
     LOG_PATH: "/var/log/api"
   ```

3. Save the file. Use spaces rather than tabs and preserve the indentation.

## Task 5: Create the Kubernetes Secret

A Kubernetes Secret stores sensitive values separately from ordinary application configuration. The starter API is designed to display only masked secret information for demonstration purposes.

1. Open `k8s/secrets.yaml`.

2. Add the following content:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: api-secrets
     labels:
       app: aks-config-api
   type: Opaque
   stringData:
     # Demonstration credentials only; do not use real secrets in this lab
     secret-endpoint: "SecretEndpointValue"
     secret-access-key: "SecretAccessKey123456"
   ```

3. Save the file.

**Security note:** These are sample values for the exercise. Do not put real passwords, access keys, or production credentials into a training manifest or source repository. Kubernetes Secrets are not automatically encrypted simply because they are base64-encoded; production clusters should use appropriate encryption-at-rest and access-control protections.

## Task 6: Create the PersistentVolumeClaim

The PersistentVolumeClaim requests Azure Disk storage so API logs can persist across pod restarts.

1. Open `k8s/pvc.yaml`.

2. Add the following YAML:

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: api-logs-pvc
     labels:
       app: aks-config-api
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
     storageClassName: managed-csi
     volumeMode: Filesystem
   ```

3. Save the file.

The `managed-csi` storage class uses the Azure Disk CSI driver. `ReadWriteOnce` allows the volume to be mounted read/write by a single node at a time.

## Task 7: Update the Deployment image reference

The deployment manifest is already partially configured with environment variables, secret references, volume mounts, and probes. You only need to set the image to the ACR endpoint created in Task 3.

1. Open `k8s/deployment.yaml`.

2. Find the image reference similar to:

   ```yaml
   image: <YOUR_ACR_ENDPOINT>/aks-config-api:latest
   ```

3. Replace `<YOUR_ACR_ENDPOINT>` with the exact ACR endpoint you copied in Task 3. For example, if the endpoint shown by the script is `myregistry.azurecr.io`, the image line should be:

   ```yaml
   image: myregistry.azurecr.io/aks-config-api:latest
   ```

   Use your actual registry endpoint, not the example above.

4. Review the rest of the manifest and confirm that the existing configuration includes the ConfigMap and Secret environment references, the `/var/log/api` volume mount, and the liveness/readiness probes. Do not remove these existing settings.

5. Save the file.

## Task 8: Apply the Kubernetes manifests

Run these commands from the **project root** (not from the `client` directory). Apply resources in the order below so configuration and storage exist before the Deployment starts.

1. Apply the ConfigMap.

   ```powershell
   kubectl apply -f k8s/configmap.yaml
   ```

2. Apply the Secret.

   ```powershell
   kubectl apply -f k8s/secrets.yaml
   ```

3. Apply the PersistentVolumeClaim.

   ```powershell
   kubectl apply -f k8s/pvc.yaml
   ```

4. Apply the Deployment.

   ```powershell
   kubectl apply -f k8s/deployment.yaml
   ```

5. Create the Service.

   ```powershell
   kubectl apply -f k8s/service.yaml
   ```

6. Wait for the Service to receive an external IP address.

   ```powershell
   kubectl get svc aks-config-api-service --watch
   ```

   When `EXTERNAL-IP` changes from `<pending>` to an IP address, press **Ctrl+C** to stop watching.

7. Check the Deployment and Service.

   ```powershell
   kubectl get deploy,svc
   ```

8. Wait for the Deployment rollout to finish.

   ```powershell
   kubectl rollout status deployment/aks-config-api
   ```

9. From the project root, run the deployment script again.

   ```powershell
   python azdeploy.py
   ```

10. Enter `5` to run **Check deployment status**. The script checks ACR, AKS, the ConfigMap, Secret, PVC, Deployment availability, and the LoadBalancer IP. It also writes the API endpoint to `client/.env`.

11. When the status check finishes, enter `7` to exit the script.

**Expected result:** All required Kubernetes resources are ready, the Service has an external IP, and `client/.env` contains `API_ENDPOINT` set to that IP.

## Task 9: Configure and run the Python client

### Step 9.1: Set up the Python environment

1. Change to the client directory.

   ```powershell
   Set-Location .\client
   ```

2. Create a virtual environment.

   ```powershell
   python -m venv .venv
   ```

3. Activate the environment in PowerShell.

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

   If PowerShell blocks script execution, allow activation for the current process only:

   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```

   Then activate the environment again:

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

4. Install the client dependencies.

   ```powershell
   python -m pip install -r requirements.txt
   ```

### Step 9.2: Run the client app

1. Start the client application.

   ```powershell
   python main.py
   ```

2. Use the menu to test each feature:

   - Enter `1` — **Check API Health (Liveness)**. Confirms the API is running and displays the non-sensitive student name loaded from the ConfigMap.
   - Enter `2` — **Check API Readiness**. Confirms the ConfigMap settings are loaded, displays the API version, and reports whether the Secret values are loaded and persistent log storage is writable.
   - Enter `3` — **View Secrets Information**. Confirms the Secret values are loaded. The app masks the values in its output. This feature exists only for demonstration and is not a production secret-management pattern.
   - Enter `4` — **Get Single Product**. Enter a product ID from `1` through `10` to retrieve one mock product.
   - Enter `5` — **List All Products**. Displays the mock product data returned by the API.
   - Enter `6` — **View Log Summary**. Displays the log file path, total requests, first and last request times, and counts by endpoint. Note the `/healthz` and `/readyz` request counts; Kubernetes probes call these endpoints automatically based on the settings in `deployment.yaml`.
   - Enter `7` — Exit the client application when finished.

## Task 10: Verify persisted logs and troubleshoot

### Check Kubernetes resources

1. Check pods.

   ```powershell
   kubectl get pods
   ```

2. Check the PVC.

   ```powershell
   kubectl get pvc
   ```

   **Expected result:** `api-logs-pvc` should eventually show `Bound`.

3. Check the Service and its external IP.

   ```powershell
   kubectl get svc aks-config-api-service
   ```

4. Check Deployment status.

   ```powershell
   kubectl rollout status deployment/aks-config-api
   ```

5. If a pod is not running, retrieve its name and inspect its details. Replace `<pod-name>` with a real name from `kubectl get pods`.

   ```powershell
   kubectl describe pod "<pod-name>"
   ```

6. Inspect the pod logs. Replace `<pod-name>` with a real pod name.

   ```powershell
   kubectl logs "<pod-name>"
   ```

### If you update ConfigMap or Secret values

After editing and applying a ConfigMap or Secret, restart the Deployment so the running application reloads the configuration.

```powershell
kubectl rollout restart deployment/aks-config-api
```

Then wait for the rollout to finish:

```powershell
kubectl rollout status deployment/aks-config-api
```

### If the client cannot connect

1. Confirm that `client/.env` exists and contains `API_ENDPOINT` set to the Service's external IP.
2. If the file is missing or the IP is stale, return to the project root, run `python azdeploy.py`, select option `5` to check deployment status and rewrite `client/.env`, then select option `7` to exit.
3. Test the health endpoint by replacing `<external-ip>` with the Service IP.

   ```powershell
   Invoke-RestMethod -Uri "http://<external-ip>/healthz"
   ```

### If the Deployment or AKS cluster fails

- From the project root, run `python azdeploy.py` and select option `5` to check resource states.
- If the AKS cluster is `Failed` or `Canceled`, correct the underlying issue and select option `6` to delete the failed cluster before retrying option `3`.
- If the VM size `Standard_D2s_v7` is unavailable or there is insufficient regional capacity, update `AKS_VM_SIZE` to a v5/v6 fallback listed in `azdeploy.py`, or select a suitable region. If the problem is quota, select a region with available Dsv5-family quota or request a quota increase.
- Confirm the ACR endpoint in `deployment.yaml` is correct.
- If you change a YAML file after deployment, reapply the corresponding manifest from the project root, for example:

  ```powershell
  kubectl apply -f k8s/deployment.yaml
  ```

### If the PVC is not bound or storage is not writable

- Confirm `k8s/pvc.yaml` requests `1Gi` and uses `storageClassName: managed-csi`.
- Check the PVC status:

  ```powershell
  kubectl describe pvc api-logs-pvc
  ```

- Check the pod events and logs for mount or permission errors.

### If Python dependencies fail

Ensure the terminal is in `client` and the virtual environment is activated. Reinstall dependencies if needed:

```powershell
python -m pip install -r requirements.txt
```

Check installed packages:

```powershell
python -m pip list
```

## Task 11: Clean up Azure resources

**Warning:** Deleting a resource group deletes every resource inside it. If you chose an existing resource group containing other workloads, those resources will also be deleted.

1. Display the resource group name you configured in `azdeploy.py`.

   ```powershell
   Write-Output "Resource group to delete: <your-resource-group-name>"
   ```

2. Confirm the name is the dedicated lab resource group. Replace the placeholder with the actual resource group name and delete the group.

   ```powershell
   az group delete --name "<your-resource-group-name>" --no-wait --yes
   ```

The command starts deletion in the background. Verify completion in the Azure portal.

## Lab completion checklist

- [ ] Downloaded and extracted `aks-configure-python.zip`.
- [ ] Updated only the resource group and region in `azdeploy.py`.
- [ ] Signed in and registered `Microsoft.ContainerService`, `Microsoft.ContainerRegistry`, `Microsoft.Compute`, `Microsoft.Network`, and `Microsoft.Storage`.
- [ ] Created ACR, built and pushed the API image, created AKS, retrieved credentials, and checked deployment status.
- [ ] Completed `k8s/configmap.yaml` with student name, API version, and log path.
- [ ] Completed `k8s/secrets.yaml` with the sample secret values.
- [ ] Completed `k8s/pvc.yaml` with a 1 GiB `managed-csi` claim.
- [ ] Updated the image reference in `k8s/deployment.yaml` with the actual ACR endpoint.
- [ ] Applied ConfigMap, Secret, PVC, Deployment, and Service in the required order.
- [ ] Waited for the LoadBalancer external IP and ran deployment status option `5` to update `client/.env`.
- [ ] Tested health, readiness, secret loading, single product, all products, and log summary using the client app.
- [ ] Verified the PVC and Deployment status.
- [ ] Deleted the dedicated lab resource group if cleanup is required.

## Key takeaways

- **ConfigMaps** separate non-sensitive application settings from the container image.
- **Kubernetes Secrets** separate sensitive configuration from ordinary settings; production systems should also protect access and enable suitable encryption.
- **PersistentVolumeClaims** request durable storage so logs can survive pod restarts.
- **Deployments** manage application pods, while **Services** provide stable networking and external access.
- Liveness and readiness probes generate automatic requests that appear in the API logs.
- Centralized logging is generally preferred over storing production logs on persistent volumes.
