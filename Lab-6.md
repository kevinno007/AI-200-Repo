# Lab 6: Deploy an AI Inference API to Azure Kubernetes Service

**Course:** AI-200 – Azure AI Cloud Developer Associate  
**Estimated duration:** 30 minutes  
**Azure services:** Microsoft Foundry, Azure Container Registry (ACR), Azure Kubernetes Service (AKS)  
**Tools:** Azure CLI, `kubectl`, Python, Visual Studio Code  
**Shell:** Windows PowerShell only

## Lab objectives

In this lab, you will deploy an AI inference API to AKS. You will provision a Microsoft Foundry `gpt-5-mini` model, build and store the API container image in ACR, create an AKS cluster, complete Kubernetes Deployment and Service manifests, and test the deployed API with a Python client.

You will configure:
- The container image and exposed port.
- A Kubernetes liveness probe.
- CPU and memory requests and limits.
- A public Kubernetes `LoadBalancer` Service.
- Health, readiness, inference, and streaming chat tests.

## Important subscription and region notes

- The Microsoft Learn lab states that ACR task runs are temporarily paused for Azure free credits. Use a **Pay-As-You-Go subscription or another paid plan**.
- Resource deployment can take **15–20 minutes**.
- The lab recommends **East US 2 (`eastus2`), Sweden Central (`swedencentral`), or Australia East (`australiaeast`)** because these regions support the model used in this exercise.
- Azure resources may incur charges until you delete the lab resource group.

## Prerequisites

Before starting, ensure you have:
- An Azure subscription with permissions and quota to create the required resources.
- Visual Studio Code installed.
- The latest Azure CLI installed.
- `kubectl` installed.
- Python 3.12 or later installed.
- Optional: the YAML extension for Visual Studio Code.

Check that the tools are available from the VS Code PowerShell terminal:

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

1. Download the starter ZIP file from the official Microsoft Learning repository:

   <https://github.com/MicrosoftLearning/mslearn-azure-ai/raw/main/downloads/python/aks-deploy-python.zip>

2. Move the ZIP file to your preferred working location and extract it into a folder.

3. Open Visual Studio Code. Select **File > Open Folder...** and open the extracted project folder.

4. Open `azdeploy.py`. At the top of the file, change **only** the resource group name and region values, then save the file. Do not change other script settings unless you need to resolve a deployment issue later.

   ```python
   rg = "<your-resource-group-name>"  # Resource Group name
   location = "<your-azure-region>"   # Azure region for the resources
   ```

   Choose one of the recommended regions: `eastus2`, `swedencentral`, or `australiaeast`. Record the resource group name for cleanup.

5. In VS Code, select **Terminal > New Terminal**. Confirm that the terminal is running **PowerShell** and that the current directory is the project root—the folder containing `azdeploy.py`.

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

4. Register the Microsoft Foundry / Cognitive Services provider.

   ```powershell
   az provider register --namespace Microsoft.CognitiveServices
   ```

5. Register the AKS provider.

   ```powershell
   az provider register --namespace Microsoft.ContainerService
   ```

6. Register the ACR provider.

   ```powershell
   az provider register --namespace Microsoft.ContainerRegistry
   ```

7. Register the compute provider.

   ```powershell
   az provider register --namespace Microsoft.Compute
   ```

8. Register the network provider.

   ```powershell
   az provider register --namespace Microsoft.Network
   ```

9. Register the storage provider.

   ```powershell
   az provider register --namespace Microsoft.Storage
   ```

These providers support the Foundry model deployment, ACR, AKS, and AKS dependencies.

## Task 3: Run the deployment script and provision Azure resources

1. From the project root, start the deployment script.

   ```powershell
   python azdeploy.py
   ```

2. Follow the menu in the order below. Wait for each operation to finish before selecting the next option.

   1. Enter `1` — **Provision gpt-5-mini model in Microsoft Foundry**. This creates the resource group if needed, creates the Foundry resource, and deploys the model.
   2. Enter `2` — **Create Azure Container Registry (ACR)**.
   3. Enter `3` — **Build and push API image to ACR**. This uses ACR Tasks and can take approximately 3–5 minutes.
   4. Enter `4` — **Create AKS cluster**. The cluster is configured with a managed identity and permission to pull images from ACR. This can take approximately 5–10 minutes.
   5. Enter `5` — **Check deployment status**. Confirm that the Foundry model, ACR, and AKS cluster report success.
   6. If all resources deployed successfully, enter `9` to exit the script.

**If the Foundry model deployment fails:**

- Enter `7` to delete/purge the failed Foundry deployment.
- Exit the script, change `location` in `azdeploy.py` to another recommended region, and rerun the script.
- Select option `1` again to provision the model before continuing through the remaining options.

**If AKS deployment fails:**

- Use option `5` to check the deployment status and read the reported error.
- Resolve the underlying quota or capacity issue.
- If the AKS cluster is in a `Failed` or `Canceled` state, use option `8` to delete the failed AKS deployment before retrying option `4`. This guarded option should not delete a healthy or in-progress cluster.
- If the error mentions `Standard_D2s_v7` being unavailable, update `AKS_VM_SIZE` to one of the v5 or v6 fallback sizes listed near the top of `azdeploy.py`, or select another recommended region. If the issue is quota, use a region with available Dsv5-family quota or request a quota increase.

Do not continue until the required Foundry model, ACR image, and AKS cluster have been provisioned successfully.

## Task 4: Complete the Kubernetes Deployment manifest

The `k8s/deployment.yaml` file defines how AKS runs the API container. You will fill in the container specification, liveness probe, and resource requests and limits.

1. In VS Code, open `k8s/deployment.yaml`.

2. Locate the `# BEGIN: Container specification` marker. Add the following block at the appropriate location, preserving the file's existing structure and YAML indentation:

   ```yaml
   containers:  # List of containers to run in the pod
   - name: api
     image: ACR_ENDPOINT/aks-api:latest  # Container image from ACR
     imagePullPolicy: Always  # Always pull the latest image from registry
     ports:  # Ports exposed by the container
     - name: http
       containerPort: 8000
       protocol: TCP
   ```

   `ACR_ENDPOINT` is a placeholder. The deployment script substitutes the actual registry endpoint when you select the deploy option later.

3. Locate the `# BEGIN: Liveness Probe Configuration` marker and add:

   ```yaml
   livenessProbe:  # Detects if container is alive or needs restart
     httpGet:
       path: /healthz  # Health check endpoint path
       port: http
     initialDelaySeconds: 10  # Seconds to wait before first check
     periodSeconds: 30
     timeoutSeconds: 5
     failureThreshold: 3  # Consecutive failures before restarting container
   ```

   Kubernetes checks `/healthz` after the initial delay and periodically thereafter. Three consecutive failures cause Kubernetes to restart the container.

4. Locate the `# BEGIN: Resource Limits Configuration` marker and add:

   ```yaml
   resources:  # CPU and memory resource specifications
     requests:  # Minimum resources guaranteed to the container
       memory: "256Mi"
       cpu: "250m"
     limits:  # Maximum resources the container can use
       memory: "512Mi"
       cpu: "500m"
   ```

   Requests are the minimum resources Kubernetes schedules for the container. Limits cap the CPU and memory the container can consume.

5. Save `deployment.yaml`. Review the entire file and confirm that all blocks are between their corresponding markers and indentation uses spaces, not tabs.

**Indentation tip:** Paste each block at column 1, select the pasted lines, and press **Tab** to indent them to match the surrounding YAML. Use **Shift+Tab** to outdent if needed.

## Task 5: Complete the Kubernetes Service manifest

The Service exposes the API outside the AKS cluster using an Azure public Load Balancer.

1. Open `k8s/service.yaml`. The starter file is expected to be empty.

2. Add the following YAML and preserve the indentation exactly:

   ```yaml
   apiVersion: v1
   kind: Service  # Service: exposes pods on a network and provides load balancing
   metadata:
     name: aks-api-service  # Unique name for the service
     labels:
       app: aks-api # Matches deployment and pod labels
     annotations:
       service.beta.kubernetes.io/azure-load-balancer-internal: "false"  # Use public load balancer
   spec:  # Service specification
     type: LoadBalancer  # Exposes service externally
     selector:  # Selects which pods to route traffic to based on labels
       app: aks-api
       version: v1
     ports:  # Port mappings between service and pods
     - name: http
       port: 80  # Service port exposed externally
       targetPort: http  # Pod container port to forward traffic to
       protocol: TCP
     sessionAffinity: None  # Client requests not pinned to specific pods
   ```

3. Save `service.yaml`.

The Service listens on port `80` and forwards requests to the container's named port `http` (port `8000`). The selectors `app: aks-api` and `version: v1` must match the pod labels in the Deployment manifest.

## Task 6: Deploy the manifests to AKS

1. Return to the project root—the directory containing `azdeploy.py`.

2. Start the deployment script.

   ```powershell
   python azdeploy.py
   ```

3. Enter `6` to select **Deploy to AKS**. This option:
   - Retrieves AKS credentials and configures `kubectl`.
   - Assigns the **Cognitive Services OpenAI User** role to the AKS kubelet managed identity so the API can authenticate to Foundry using Microsoft Entra ID.
   - Replaces the ACR and Foundry endpoint placeholders in the Deployment manifest.
   - Applies `deployment.yaml` and `service.yaml` to the cluster.

4. Wait for the deployment operation to finish, then enter `9` to exit the script.

5. Verify the Deployment and Service.

   ```powershell
   kubectl get deploy,svc
   ```

   **Expected result:** The `aks-api` Deployment shows `READY` as `1/1` (or the configured replica count). The `aks-api-service` Service should show a public `EXTERNAL-IP`, not `<pending>`.

6. Wait for the rollout to complete successfully.

   ```powershell
   kubectl rollout status deploy/aks-api
   ```

7. If needed, verify that the pods are running.

   ```powershell
   kubectl get pods
   ```

   If the external IP remains `<pending>`, allow more time for Azure Load Balancer provisioning and check the pod and Service status before proceeding.

## Task 7: Configure the Python client environment

The client application provides a menu for testing API health, readiness, model inference, and streaming chat.

1. Change to the `client` directory.

   ```powershell
   Set-Location .\client
   ```

2. Create a Python virtual environment.

   ```powershell
   python -m venv .venv
   ```

3. Activate the environment in PowerShell.

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

   If PowerShell blocks the activation script, allow script execution for the current terminal process only:

   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   ```

   Then activate the environment again:

   ```powershell
   .\.venv\Scripts\Activate.ps1
   ```

4. Install the required packages.

   ```powershell
   python -m pip install -r requirements.txt
   ```

5. Start the client app.

   ```powershell
   python main.py
   ```

## Task 8: Test the API with the client app

Use the menu in the running Python client. Enter each option and review the result before moving to the next one.

1. Enter `1` — **Check API Health (Liveness)**. This verifies that the API is running and responds to the same `/healthz` endpoint used by the Kubernetes liveness probe.

2. Enter `2` — **Check API Readiness (Foundry Connectivity)**. This verifies that the API can connect to the Foundry model endpoint and is ready to process inference requests.

3. Enter `3` — **Send Inference Request**. The client sends a prompt to the deployed API and displays the model's complete response.

4. Enter `4` — **Start Chat Session (Streaming)**. Start an interactive chat and observe responses as they stream from the model.

5. When finished, enter `5` to exit the client app.

## Task 9: Troubleshooting

### Verify Azure resource deployment

- In the [Azure portal](https://portal.azure.com), open the lab resource group.
- Confirm that the Microsoft Foundry resource has provisioning state `Succeeded` and the `gpt-5-mini` model deployment exists.
- Confirm that ACR contains the `aks-api` image.
- Confirm that AKS has provisioning state `Succeeded` and its nodes are running.

### Check AKS Deployment and Service

1. Check pods.

   ```powershell
   kubectl get pods
   ```

2. Check the Service and its external IP.

   ```powershell
   kubectl get svc
   ```

3. Inspect a pod and its events. Replace `<pod-name>` with a real pod name from `kubectl get pods`.

   ```powershell
   kubectl describe pod "<pod-name>"
   ```

4. Inspect container logs. Replace `<pod-name>` with a real pod name.

   ```powershell
   kubectl logs "<pod-name>"
   ```

5. Verify the rollout.

   ```powershell
   kubectl rollout status deploy/aks-api
   ```

### Fix AKS cluster creation failures

- Run `python azdeploy.py` and select option `5` to inspect deployment status.
- If `Standard_D2s_v7` is unavailable or the region lacks capacity, exit with option `9`, adjust `AKS_VM_SIZE` to one of the v5 or v6 fallback sizes listed in the script, or choose another recommended region. Then rerun option `4`.
- If the error reports insufficient quota, select a region where the subscription has available Dsv5-family quota or request a quota increase.
- If option `5` reports `Failed` or `Canceled`, fix the underlying issue and select option `8` to delete the failed AKS deployment before retrying option `4`. If no AKS resource was created, deletion is unnecessary.

### Fix YAML or pod configuration problems

- Confirm that all YAML sections are inside the correct `BEGIN`/`END` markers in `deployment.yaml`.
- Use spaces rather than tabs and verify indentation.
- Confirm that the Deployment labels match the Service selectors (`app: aks-api` and `version: v1`).
- Confirm the image endpoint was substituted by the deployment script.
- Confirm the liveness probe path is `/healthz` and the container's named port is `http`.

### Verify client connectivity

1. Get the public IP address.

   ```powershell
   kubectl get svc aks-api-service
   ```

2. Replace `<external-ip>` with the public IP from the previous command and test the health endpoint.

   ```powershell
   Invoke-RestMethod -Uri "http://<external-ip>/healthz"
   ```

If the endpoint does not respond, confirm the Service has an external IP, the pod is running, and the Service selector matches the pod labels.

### Fix Python environment or dependency issues

- Confirm that the current directory is `client`.
- Confirm the virtual environment is activated.
- Reinstall dependencies if needed.

  ```powershell
  python -m pip install -r requirements.txt
  ```

- List installed packages.

  ```powershell
  python -m pip list
  ```

## Task 10: Clean up Azure resources

**Warning:** Deleting a resource group deletes every resource inside it. If you selected an existing resource group containing other workloads, those resources will also be deleted.

1. Display the resource group name you configured in `azdeploy.py`.

   ```powershell
   Write-Output "Resource group to delete: <your-resource-group-name>"
   ```

2. Confirm that the resource group is dedicated to this lab. Replace `<your-resource-group-name>` with the actual value, then delete the resource group and all its resources.

   ```powershell
   az group delete --name "<your-resource-group-name>" --no-wait --yes
   ```

The command starts deletion in the background. Verify completion in the Azure portal.

## Lab completion checklist

- [ ] Downloaded and extracted `aks-deploy-python.zip`.
- [ ] Updated only the resource group and region in `azdeploy.py`.
- [ ] Signed in to the correct Azure subscription and registered all six required providers.
- [ ] Provisioned the Foundry `gpt-5-mini` model, ACR, image, and AKS cluster using menu options `1`, `2`, `3`, and `4`.
- [ ] Checked deployment status with option `5` and exited with option `9`.
- [ ] Completed the container specification, liveness probe, and resource requests/limits in `k8s/deployment.yaml`.
- [ ] Completed `k8s/service.yaml` with a public LoadBalancer and correct selectors.
- [ ] Deployed the manifests using option `6`.
- [ ] Verified the Deployment rollout and the Service external IP.
- [ ] Created and activated the Python virtual environment in `client`.
- [ ] Tested API health, readiness, inference, and streaming chat.
- [ ] Deleted the dedicated lab resource group if cleanup is required.

## Key takeaways

- AKS Deployments manage container replicas and maintain the desired state.
- Liveness probes help Kubernetes detect and restart unhealthy containers.
- Resource requests and limits improve scheduling and constrain resource consumption.
- Kubernetes Services provide stable networking and can expose an API through an Azure Load Balancer.
- AKS can use managed identities to pull images from ACR and authenticate to Microsoft Foundry without embedding credentials in the container.
