# Lab 8: Troubleshoot Apps on Azure Kubernetes Service

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Azure services:** Azure Container Registry (ACR), Azure Kubernetes Service (AKS)  
**Tools:** Azure CLI, `kubectl`, Python, Visual Studio Code  
**Shell:** Windows PowerShell only

## Lab objectives

In this lab, you will deploy a containerized API to AKS and troubleshoot three common Kubernetes failures:

1. A Service selector that does not match pod labels.
2. A container crash caused by a missing required environment variable.
3. A readiness probe configured with an invalid path.

You will use `kubectl` to inspect pods, Services, EndpointSlices, logs, and events; edit live Kubernetes resources; and verify end-to-end connectivity after fixing the issues.

## Important notes

- Azure resource deployment can take **10–15 minutes**.
- Microsoft Learn notes that ACR task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan**.
- Azure resources may incur charges until you delete the lab resource group.

## Prerequisites

Before starting, ensure you have:
- An Azure subscription with permissions and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- `kubectl` installed.
- Python 3.12 or later installed.

Check the tools in the VS Code PowerShell terminal:

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

   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aks-troubleshoot-python.zip>

2. Move the ZIP file to your preferred working location and extract it into a folder.

3. Open Visual Studio Code. Select **File > Open Folder...** and open the extracted project folder.

4. Open `azdeploy.py`. Change only the resource group name and Azure region values at the top of the script, then save the file. Do not change anything else in the script.

   ```python
   rg = "<your-resource-group-name>"  # Resource Group name
   location = "<your-azure-region>"   # Azure region for the resources
   ```

   Record the resource group name for cleanup.

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

These providers support AKS, ACR, and AKS dependencies. Registering them explicitly can help avoid occasional cluster-creation failures on new subscriptions.

## Task 3: Deploy the Azure resources

1. From the project root, start the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. Follow the menu in this order, waiting for each operation to finish before selecting the next option:

   1. Enter `1` — **Create Azure Container Registry (ACR)**.
   2. Enter `2` — **Build and push API image to ACR**. This uses ACR Tasks and can take approximately 3–5 minutes.
   3. Enter `3` — **Create AKS cluster**. The cluster is configured with a managed identity and permission to pull images from ACR.
   4. Enter `4` — **Get AKS credentials for kubectl**. This retrieves credentials and configures `kubectl`.
   5. Enter `5` — **Deploy application to AKS**.
   6. Enter `6` — **Check deployment status**. Confirm that the resources have been deployed successfully.
   7. If all services report success, enter `8` to exit the script.

3. Leave the terminal open. You will continue using it throughout the exercise.

**If AKS deployment fails:**

- Run option `6` to inspect the resource status and error.
- If the AKS cluster is `Failed` or `Canceled`, fix the underlying issue, then run option `7` to delete the failed AKS deployment before retrying option `3`. This guarded option does not delete a healthy or in-progress cluster.
- If no AKS resource was created, there is no cluster to delete.
- If `Standard_D2s_v7` is unavailable or the region lacks capacity, update `AKS_VM_SIZE` to one of the v5/v6 fallback sizes listed near the top of `azdeploy.py`, or change the region to one with capacity. Then retry option `3`.
- If the error reports insufficient quota, choose a region where your subscription has available Dsv5-family quota or request a quota increase.

## Task 4: Verify the original deployment

The deployment script creates the Kubernetes resources in the `aks-troubleshoot` namespace. Always include `-n aks-troubleshoot` in commands for this lab.

1. Verify that the pod is running.

   ```powershell
   kubectl get pods -n aks-troubleshoot
   ```

   **Expected result:** One pod with `Running` status and `1/1` in the `READY` column.

2. Verify that the Service has an endpoint.

   ```powershell
   kubectl get endpointslices -l kubernetes.io/service-name=api-service -n aks-troubleshoot
   ```

   **Expected result:** An EndpointSlice with a pod IP address in the `ENDPOINTS` column.

3. Start port forwarding from the local machine to the Kubernetes Service.

   ```powershell
   kubectl port-forward service/api-service 8080:80 -n aks-troubleshoot
   ```

4. Keep that terminal running. Open a **second PowerShell terminal** in VS Code and test the health endpoint.

   ```powershell
   Invoke-RestMethod http://localhost:8080/healthz
   ```

   **Expected result:** JSON with `"status": "healthy"`.

5. Return to the first terminal and press **Ctrl+C** to stop port forwarding.

You have confirmed that the original deployment is working before introducing errors.

## Task 5: Diagnose and fix a Service selector mismatch

A Kubernetes Service routes requests to pods based on label selectors. If the Service selector does not match pod labels, the Service has no endpoints.

### Step 5.1: Introduce the problem

1. Apply the Service manifest that intentionally changes the selector to `app: api-v2`.

   ```powershell
   kubectl apply -f k8s/label-mismatch-service.yaml -n aks-troubleshoot
   ```

2. Confirm that the pod is still running and inspect its labels.

   ```powershell
   kubectl get pods --show-labels -n aks-troubleshoot
   ```

   **Expected result:** The pod is running and has the label `app=api`.

3. Check the Service EndpointSlices.

   ```powershell
   kubectl get endpointslices -l kubernetes.io/service-name=api-service -n aks-troubleshoot
   ```

   **Expected result:** The endpoint is shown as `<unset>` or there are no usable pod endpoints, because the selector does not match.

4. Inspect the Service.

   ```powershell
   kubectl describe service api-service -n aks-troubleshoot
   ```

   **Expected result:** The selector shows `app=api-v2`, while the pod label is `app=api`.

### Step 5.2: Fix the selector

1. Open the live Service configuration.

   ```powershell
   kubectl edit service api-service -n aks-troubleshoot
   ```

2. In the editor, find the `selector` section and change `app: api-v2` to `app: api`.

3. Save the changes and close the editor. On Windows PowerShell, `kubectl edit` typically opens Notepad. Use **Ctrl+S** to save, then close Notepad.

4. Verify that EndpointSlices now contain an IP address.

   ```powershell
   kubectl get endpointslices -l kubernetes.io/service-name=api-service -n aks-troubleshoot
   ```

You have fixed the label mismatch. The Service can now select the API pod.

## Task 6: Diagnose and fix a CrashLoopBackOff

A container can enter `CrashLoopBackOff` when the application repeatedly fails to start. In this scenario, the deployment manifest removes the required `API_KEY` environment variable.

### Step 6.1: Introduce the problem

1. Apply the manifest that removes the required environment variable.

   ```powershell
   kubectl apply -f k8s/crashloop-deployment.yaml -n aks-troubleshoot
   ```

2. Watch the pod status.

   ```powershell
   kubectl get pods -n aks-troubleshoot --watch
   ```

   After a short time, the new pod should enter `CrashLoopBackOff`. Press **Ctrl+C** to stop watching.

3. Inspect the logs to find the application error.

   ```powershell
   kubectl logs -l app=api -n aks-troubleshoot
   ```

   **Expected result:** The logs indicate that the required environment variable is missing.

### Step 6.2: Restore the environment variable

1. Edit the live Deployment.

   ```powershell
   kubectl edit deployment api-deployment -n aks-troubleshoot
   ```

2. Under `spec.template.spec.containers`, find the container named `api`. Add the following `env` block directly under the container's `name` entry, preserving YAML indentation:

   ```yaml
   - name: api
     env:
       - name: API_KEY
         value: "demo-api-key-12345"
   ```

   Do not replace the rest of the container specification. Add the `env` block to the existing container.

3. Save the file and close the editor.

4. Watch the pod status again.

   ```powershell
   kubectl get pods -n aks-troubleshoot --watch
   ```

   **Expected result:** After the Deployment rolls out, the new pod reaches `Running` and becomes ready. Press **Ctrl+C** to stop watching.

You have resolved the CrashLoopBackOff by restoring the required environment variable.

## Task 7: Diagnose and fix a readiness probe failure

A readiness probe failure can leave a pod running but not ready. Kubernetes does not add a not-ready pod to Service endpoints. During a rolling update, the old healthy pod may continue serving traffic while the new pod remains unready.

### Step 7.1: Introduce the problem

1. Apply the manifest that contains an invalid readiness-probe path.

   ```powershell
   kubectl apply -f k8s/probe-failure-deployment.yaml -n aks-troubleshoot
   ```

2. Inspect the pods.

   ```powershell
   kubectl get pods -n aks-troubleshoot
   ```

   **Expected result:** The new pod shows `Running` but `0/1` ready, while the old pod remains `1/1` ready. The rolling update is blocked.

3. Inspect unhealthy events.

   ```powershell
   kubectl get events -n aks-troubleshoot --field-selector reason=Unhealthy
   ```

   **Expected result:** An event reports a readiness probe failure, typically with HTTP `404`, because the probe uses `/invalid-path`.

### Step 7.2: Fix the readiness probe

1. Edit the live Deployment.

   ```powershell
   kubectl edit deployment api-deployment -n aks-troubleshoot
   ```

2. Find the `readinessProbe` section and change the path from `/invalid-path` to `/healthz`.

3. Save the changes and close the editor.

4. Verify that the new pod becomes ready and the old pod terminates.

   ```powershell
   kubectl get pods -n aks-troubleshoot
   ```

   **Expected result:** One pod remains with `Running` status and `1/1` in the `READY` column.

You have fixed the readiness probe configuration.

## Task 8: Verify end-to-end connectivity

After all three troubleshooting scenarios, confirm that the application is working correctly.

1. Start port forwarding again.

   ```powershell
   kubectl port-forward service/api-service 8080:80 -n aks-troubleshoot
   ```

2. Keep port forwarding running. Open a **second PowerShell terminal** and test all three endpoints.

   **Health endpoint:**

   ```powershell
   Invoke-RestMethod http://localhost:8080/healthz
   ```

   **Readiness endpoint:**

   ```powershell
   Invoke-RestMethod http://localhost:8080/readyz
   ```

   **Application information endpoint:**

   ```powershell
   Invoke-RestMethod http://localhost:8080/api/info
   ```

   **Expected result:** Each endpoint returns a valid response. Health and readiness checks should succeed, and `/api/info` should return application information.

3. Review the API logs to confirm requests reached the container.

   ```powershell
   kubectl logs -l app=api -n aks-troubleshoot
   ```

4. Return to the terminal running port forwarding and press **Ctrl+C**.

## Task 9: Troubleshooting reference

### Check the deployment script status

From the project root, run the script and choose option `6` to check deployment status.

```powershell
python azdeploy.py
```

This checks ACR provisioning, AKS provisioning, and Kubernetes resource availability. When finished, use option `8` to exit the script.

### ACR image pull errors

If pods show `ImagePullBackOff` or `ErrImagePull`:

- Run the deployment script and select option `2` to build and push the API image again if needed.
- Confirm that the image was pushed to ACR successfully.
- Confirm that the AKS cluster has permission to pull from ACR.

### `kubectl` connection issues

1. Run the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. Select option `4` to retrieve AKS credentials again, then exit using option `8`.

3. Check the cluster state. Replace the placeholders with your resource group and AKS cluster names.

   ```powershell
   az aks show --resource-group "<your-resource-group-name>" --name "<your-aks-name>" --query provisioningState --output tsv
   ```

### Reset the troubleshooting scenarios

If you need to restart the exercise scenarios, run the deployment script and select option `5` to redeploy the original working application configuration. This reapplies the base Deployment and Service manifests and resets the changes made during the exercise.

```powershell
python azdeploy.py
```

After selecting option `5`, wait for the redeployment to complete, then select option `6` to check status and option `8` to exit.

## Task 10: Clean up Azure resources

**Warning:** Deleting a resource group deletes every resource inside it. If you chose an existing resource group containing other workloads, those resources will also be deleted.

1. Display the resource group name you configured in `azdeploy.py`.

   ```powershell
   Write-Output "Resource group to delete: <your-resource-group-name>"
   ```

2. Confirm that the resource group is dedicated to this lab. Replace the placeholder with the actual resource group name, then delete the resource group.

   ```powershell
   az group delete --name "<your-resource-group-name>" --no-wait --yes
   ```

The command starts deletion in the background. Verify completion in the Azure portal.

## Lab completion checklist

- [ ] Downloaded and extracted `aks-troubleshoot-python.zip`.
- [ ] Updated only the resource group and region in `azdeploy.py`.
- [ ] Signed in to Azure and registered the five required resource providers.
- [ ] Created ACR, built and pushed the image, created AKS, retrieved credentials, deployed the application, and checked deployment status.
- [ ] Verified the original pod was `Running` and the Service had an endpoint.
- [ ] Reproduced and fixed the Service selector mismatch (`app: api-v2` → `app: api`).
- [ ] Reproduced and fixed the CrashLoopBackOff by restoring `API_KEY`.
- [ ] Reproduced and fixed the readiness probe failure (`/invalid-path` → `/healthz`).
- [ ] Verified `/healthz`, `/readyz`, and `/api/info` through port forwarding.
- [ ] Reviewed pod logs and Kubernetes events.
- [ ] Deleted the dedicated lab resource group if cleanup is required.

## Key takeaways

- A Kubernetes Service selector must match pod labels; otherwise, the Service has no usable endpoints.
- A missing required environment variable can prevent an application from starting and cause `CrashLoopBackOff`.
- Readiness probes determine whether a pod can receive Service traffic; an incorrect path can block a rolling update.
- `kubectl get`, `describe`, `logs`, `get events`, and `edit` are core tools for diagnosing live Kubernetes workloads.
- Always verify service connectivity after correcting a Kubernetes configuration issue.
