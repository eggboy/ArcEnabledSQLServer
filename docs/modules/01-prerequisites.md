# Module 1: Prerequisites and Environment Setup

## Objective

Prepare your environment with all necessary tools, permissions, and configurations to deploy Azure Arc enabled SQL Server on Kubernetes.

## Prerequisites Checklist

- [ ] Azure subscription with Owner or Contributor access
- [ ] Existing Kubernetes cluster (vanilla K8s, AKS, EKS, GKE, or on-premises)
- [ ] `kubectl` installed and configured
- [ ] Azure CLI installed (v2.60+)
- [ ] Network connectivity to Azure endpoints

## Step 1: Install Azure CLI

If you don't have Azure CLI installed:

```bash
# Linux
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# macOS
brew update && brew install azure-cli

# Windows (PowerShell)
winget install -e --id Microsoft.AzureCLI
```

Verify installation:

```bash
az --version
# Ensure version is 2.60 or higher
```

## Step 2: Install Required Azure CLI Extensions

```bash
# Install/update the connectedk8s extension for Arc-enabled Kubernetes
az extension add --name connectedk8s --upgrade

# Install/update the k8s-extension for managing extensions on Arc clusters
az extension add --name k8s-extension --upgrade

# Install/update the arcdata extension for Arc data services
az extension add --name arcdata --upgrade

# Install/update the customlocation extension
az extension add --name customlocation --upgrade
```

## Step 3: Register Azure Resource Providers

```bash
# Login to Azure
az login

# Set your subscription
az account set --subscription "<your-subscription-id>"

# Register required resource providers
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.ExtendedLocation
az provider register --namespace Microsoft.AzureArcData

# Verify registration status
az provider show -n Microsoft.Kubernetes -o table
az provider show -n Microsoft.KubernetesConfiguration -o table
az provider show -n Microsoft.ExtendedLocation -o table
az provider show -n Microsoft.AzureArcData -o table
```

> **Note:** Provider registration can take up to 10 minutes. Wait until the status shows `Registered`.

## Step 4: Create Azure Resource Group

```bash
# Set variables
export RESOURCE_GROUP="arc-sqlserver-workshop"
export LOCATION="eastus"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION
```

## Step 5: Verify Kubernetes Cluster Access

Ensure your `kubectl` is configured and connected to your cluster:

```bash
# Verify cluster connectivity
kubectl get nodes

# Verify cluster info
kubectl cluster-info

# Check available storage classes (needed for data services)
kubectl get storageclass
```

Expected output should show your cluster nodes in `Ready` state.

## Step 6: Verify Cluster Requirements

Azure Arc Data Services require minimum resources on your cluster:

| Resource | Minimum Requirement |
|----------|-------------------|
| Nodes | 3 worker nodes |
| CPU per node | 4 cores |
| RAM per node | 16 GB |
| Storage | Default StorageClass with dynamic provisioning |

Verify node resources:

```bash
# Check node resources
kubectl describe nodes | grep -A 5 "Allocatable"

# Verify a default storage class exists
kubectl get storageclass | grep "(default)"
```

## Step 7: Configure Network Requirements

Ensure your cluster has outbound connectivity to the following Azure endpoints over port 443 (HTTPS):

| Endpoint | Purpose |
|----------|---------|
| `management.azure.com` | Azure Resource Manager |
| `login.microsoftonline.com` | Azure Active Directory |
| `*.his.arc.azure.com` | Azure Arc services |
| `*.guestconfiguration.azure.com` | Arc guest configuration |
| `*.dp.kubernetesconfiguration.azure.com` | Kubernetes configuration |
| `mcr.microsoft.com` | Microsoft Container Registry |
| `*.data.mcr.microsoft.com` | Container image layers |

Test connectivity:

```bash
# Test connectivity from a pod in the cluster
kubectl run test-connectivity --image=curlimages/curl --rm -it --restart=Never -- \
  curl -s -o /dev/null -w "%{http_code}" https://management.azure.com
```

## Step 8: Set Environment Variables

Create a configuration file for use throughout the workshop:

```bash
# Create environment configuration
cat << 'EOF' > workshop-env.sh
#!/bin/bash

# Azure Configuration
export SUBSCRIPTION_ID="<your-subscription-id>"
export RESOURCE_GROUP="arc-sqlserver-workshop"
export LOCATION="eastus"

# Arc Kubernetes Configuration
export ARC_CLUSTER_NAME="arc-k8s-cluster"
export ARC_NAMESPACE="arc-data"

# Data Controller Configuration
export DATA_CONTROLLER_NAME="arc-dc"
export CONNECTIVITY_MODE="indirect"  # Use "direct" if cluster has direct Azure connectivity

# SQL Managed Instance Configuration
export SQL_MI_NAME="arc-sql-mi"
export SQL_MI_STORAGE_CLASS="default"  # Replace with your storage class name
export SQL_MI_REPLICAS=1

# Azure SQL MI (Target for replication)
export AZURE_SQL_MI_NAME="azure-sql-mi-target"
export AZURE_SQL_MI_RG="arc-sqlserver-workshop"

EOF

# Source the environment
source workshop-env.sh
```

> **Important:** Replace placeholder values with your actual configuration before sourcing.

## Validation

Run the following to confirm your environment is ready:

```bash
# Verify Azure CLI
az version --output table

# Verify extensions
az extension list --output table | grep -E "connectedk8s|k8s-extension|arcdata|customlocation"

# Verify kubectl
kubectl version --client

# Verify cluster access
kubectl get nodes -o wide

# Verify providers
az provider show -n Microsoft.AzureArcData --query "registrationState" -o tsv
```

All checks should pass before proceeding to the next module.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Provider registration stuck | Wait 10-15 minutes, then re-check status |
| kubectl not connecting | Verify kubeconfig: `kubectl config current-context` |
| No default storage class | Create one or specify the class name in subsequent steps |
| Insufficient cluster resources | Scale up your cluster or add worker nodes |

---

**Next:** [Module 2: Connect Kubernetes to Azure Arc](02-arc-enable-kubernetes.md)
