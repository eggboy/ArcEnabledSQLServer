# Module 2: Connect Kubernetes to Azure Arc

## Objective

Connect your existing Kubernetes cluster to Azure Arc, enabling centralized management and the ability to deploy Arc data services.

## Overview

Azure Arc-enabled Kubernetes connects your cluster to Azure by deploying agents that:
- Establish secure connectivity to Azure
- Enable cluster management from the Azure portal
- Allow deployment of Azure extensions and services
- Provide Azure Policy and monitoring capabilities

## Step 1: Source Environment Variables

```bash
source workshop-env.sh
```

## Step 2: Connect Kubernetes Cluster to Azure Arc

```bash
# Connect the cluster to Azure Arc
az connectedk8s connect \
  --name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION
```

This command:
- Deploys Azure Arc agents to your cluster
- Creates an Azure Arc-enabled Kubernetes resource in your resource group
- Establishes secure connectivity between your cluster and Azure

> **Note:** This process takes 2-5 minutes to complete.

## Step 3: Verify Arc Connection

```bash
# Verify the Arc connection status
az connectedk8s show \
  --name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table

# Check Arc agent pods are running
kubectl get pods -n azure-arc
```

Expected output for the pods:

```
NAME                                         READY   STATUS    RESTARTS   AGE
cluster-metadata-operator-xxx                2/2     Running   0          3m
clusterconnect-agent-xxx                     3/3     Running   0          3m
clusteridentityoperator-xxx                  2/2     Running   0          3m
config-agent-xxx                             2/2     Running   0          3m
controller-manager-xxx                       2/2     Running   0          3m
extension-manager-xxx                        2/2     Running   0          3m
flux-logs-agent-xxx                          1/1     Running   0          3m
kube-aad-proxy-xxx                           2/2     Running   0          3m
metrics-agent-xxx                            2/2     Running   0          3m
resource-sync-agent-xxx                      2/2     Running   0          3m
```

## Step 4: Enable Custom Locations

Custom Locations allow you to deploy Azure services (like SQL MI) to specific namespaces on your cluster:

```bash
# Enable the custom-locations feature on the connected cluster
az connectedk8s enable-features \
  --name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --features cluster-connect custom-locations
```

## Step 5: Install Arc Data Services Extension

Install the Azure Arc Data Services extension on your connected cluster:

```bash
# Install the Arc data services extension
az k8s-extension create \
  --name arc-data-services \
  --extension-type microsoft.arcdataservices \
  --cluster-type connectedClusters \
  --cluster-name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --auto-upgrade false \
  --scope cluster \
  --release-namespace $ARC_NAMESPACE \
  --config Microsoft.CustomLocation.ServiceAccount=sa-bootstrapper
```

Wait for the extension to be installed:

```bash
# Check extension status
az k8s-extension show \
  --name arc-data-services \
  --cluster-type connectedClusters \
  --cluster-name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table
```

> Wait until `Provisioning State` shows `Succeeded`.

## Step 6: Create Custom Location

```bash
# Get the connected cluster ID
export CONNECTED_CLUSTER_ID=$(az connectedk8s show \
  --name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Get the extension ID
export EXTENSION_ID=$(az k8s-extension show \
  --name arc-data-services \
  --cluster-type connectedClusters \
  --cluster-name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Create custom location
az customlocation create \
  --name "arc-sql-location" \
  --resource-group $RESOURCE_GROUP \
  --namespace $ARC_NAMESPACE \
  --host-resource-id $CONNECTED_CLUSTER_ID \
  --cluster-extension-ids $EXTENSION_ID
```

## Step 7: Verify Custom Location

```bash
# Verify the custom location
az customlocation show \
  --name "arc-sql-location" \
  --resource-group $RESOURCE_GROUP \
  --output table
```

## Step 8: Verify in Azure Portal

1. Navigate to the [Azure Portal](https://portal.azure.com)
2. Go to your resource group: `arc-sqlserver-workshop`
3. You should see:
   - **Kubernetes - Azure Arc** resource (your connected cluster)
   - **Custom Location** resource

## Validation

Run the following checks to confirm everything is properly configured:

```bash
echo "=== Arc Connection Status ==="
az connectedk8s show --name $ARC_CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "connectivityStatus" -o tsv

echo "=== Extension Status ==="
az k8s-extension show --name arc-data-services \
  --cluster-type connectedClusters \
  --cluster-name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "provisioningState" -o tsv

echo "=== Custom Location ==="
az customlocation show --name "arc-sql-location" \
  --resource-group $RESOURCE_GROUP \
  --query "provisioningState" -o tsv

echo "=== Arc Agent Pods ==="
kubectl get pods -n azure-arc --field-selector=status.phase=Running --no-headers | wc -l
echo "pods running in azure-arc namespace"

echo "=== Data Services Pods ==="
kubectl get pods -n $ARC_NAMESPACE --no-headers 2>/dev/null | wc -l
echo "pods in arc-data namespace"
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `connectedk8s connect` fails | Check outbound connectivity to Azure endpoints |
| Arc agents in CrashLoopBackOff | Check logs: `kubectl logs -n azure-arc <pod-name>` |
| Extension install fails | Ensure cluster meets minimum resource requirements |
| Custom location creation fails | Verify extension is in `Succeeded` state first |
| `custom-locations` feature not available | Update Azure CLI: `az upgrade` |

### Common Errors

**Error: Helm install failed**
```bash
# Clean up and retry
az connectedk8s delete --name $ARC_CLUSTER_NAME --resource-group $RESOURCE_GROUP
az connectedk8s connect --name $ARC_CLUSTER_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
```

**Error: Insufficient permissions**
```bash
# Ensure proper RBAC
kubectl create clusterrolebinding arc-admin --clusterrole=cluster-admin --user=<your-user>
```

---

**Previous:** [Module 1: Prerequisites and Environment Setup](01-prerequisites.md)  
**Next:** [Module 3: Deploy Arc Data Services and SQL Managed Instance](03-deploy-sql-server.md)
