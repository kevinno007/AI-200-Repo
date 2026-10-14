## **Lab: Set Up an AKS Cluster Using Azure Portal**

### **Objective**
This lab will guide you through the process of creating an AKS cluster using the Azure Portal.

---

### **Prerequisites**
1. An active Azure account with appropriate subscription permissions.
2. Basic knowledge of Kubernetes and AKS.

---

### **Steps to Set Up the AKS Cluster**

#### **Step 1: Log in to the Azure Portal**
1. Open a web browser and navigate to the [Azure Portal](https://portal.azure.com/).
2. Sign in with your Azure credentials.

---

#### **Step 2: Navigate to the AKS Service**
1. In the search bar at the top, type **Kubernetes Services** and select it from the results.
2. Click on the **Create** button and select **Kubernetes cluster**.

---

#### **Step 3: Configure the Basic Settings**
1. **Subscription**: Select the appropriate subscription for billing.
2. **Resource Group**:
   - Click **Create new** and name it (e.g., `aks-resource-group`).
   - Alternatively, select an existing resource group.
3. **Cluster Name**:
   - Enter a name for your AKS cluster (e.g., `my-aks-cluster`).
4. **Region**:
   - Select the region where you want the cluster to be deployed (e.g., East US, West Europe).
5. **Kubernetes Version**:
   - Select the Kubernetes version you wish to use (default or latest is recommended).

---

#### **Step 4: Configure Node Settings**
1. **Node Size**:
   - Choose a virtual machine size for your nodes (e.g., `Standard_B2s` for basic workloads).
2. **Node Count**:
   - Set the initial number of nodes (e.g., `3` for a 3-node cluster).
3. Leave **Enable Virtual Nodes** unchecked unless you need to scale quickly with Azure Container Instances.

---

#### **Step 5: Configure Authentication**
1. Under the **Authentication** tab:
   - Choose **System-assigned managed identity** for simplicity.
   - Enable **RBAC** (Role-Based Access Control) to manage access and permissions.

---

#### **Step 6: Configure Networking**
1. Under the **Networking** tab:
   - Choose **Azure CNI** for better integration with Azure services.
   - Either allow Azure to create a virtual network automatically or select an existing one.
   - Decide whether you want to enable the **HTTP application routing** add-on.

---

#### **Step 7: Enable Add-ons (Optional)**
1. **Container Insights**:
   - Enable monitoring for your cluster to track performance metrics and logs.
2. **Azure Policy**:
   - Enable this if you need governance and compliance for your cluster.

---

#### **Step 8: Review and Create**
1. Click **Review + Create** to validate your configuration.
2. If there are no validation errors, click **Create** to begin provisioning the AKS cluster.

---

#### **Step 9: Access the AKS Cluster**
1. Once the deployment is complete, navigate to the resource by clicking **Go to Resource**.
2. Note the **Cluster API Server Endpoint**.

---

#### **Step 10: Connect to the Cluster**
1. Install Azure CLI and configure it if not already done:
   ```bash
   az login
   ```
2. Download the credentials to connect to your AKS cluster:
   ```bash
   az aks get-credentials --resource-group aks-resource-group --name my-aks-cluster
   ```
3. Verify the cluster connection:
   ```bash
   kubectl get nodes
   ```
