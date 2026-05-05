# Module 3: Deploy Arc Data Services and SQL Managed Instance

## Objective

Deploy the Azure Arc Data Controller and a SQL Managed Instance on your Arc-enabled Kubernetes cluster, providing a cloud-managed SQL Server experience on your own infrastructure.

## Overview

The deployment consists of two main components:
1. **Data Controller**: The control plane that manages the lifecycle of data services
2. **SQL Managed Instance**: The SQL Server instance running as a container on Kubernetes

## Step 1: Source Environment Variables

```bash
source workshop-env.sh
```

## Step 2: Create the Data Controller

The Data Controller is required before deploying any data services:

```bash
# Create the Arc data controller
az arcdata dc create \
  --name $DATA_CONTROLLER_NAME \
  --resource-group $RESOURCE_GROUP \
  --custom-location "arc-sql-location" \
  --connectivity-mode $CONNECTIVITY_MODE \
  --profile-name "azure-arc-kubeadm" \
  --storage-class $SQL_MI_STORAGE_CLASS \
  --infrastructure "other"
```

> **Note:** Choose the appropriate `--profile-name` based on your cluster type:
> - `azure-arc-kubeadm` - For vanilla Kubernetes / kubeadm clusters
> - `azure-arc-aks-default-storage` - For AKS
> - `azure-arc-eks` - For EKS
> - `azure-arc-gke` - For GKE
> - `azure-arc-openshift` - For OpenShift

### Monitor Data Controller Deployment

```bash
# Watch the data controller deployment
kubectl get datacontroller -n $ARC_NAMESPACE --watch

# Check all pods in the namespace
kubectl get pods -n $ARC_NAMESPACE

# View data controller status
az arcdata dc status show --name $DATA_CONTROLLER_NAME --resource-group $RESOURCE_GROUP
```

Wait until the data controller state shows `Ready`. This typically takes 5-10 minutes.

Expected pods when ready:

```
NAME                                    READY   STATUS    RESTARTS   AGE
bootstrapper-xxx                        1/1     Running   0          5m
control-xxx                             2/2     Running   0          5m
controldb-0                             2/2     Running   0          5m
logsdb-0                                3/3     Running   0          5m
logsui-xxx                              3/3     Running   0          5m
metricsdb-0                             2/2     Running   0          5m
metricsdc-xxx                           2/2     Running   0          5m
metricsui-xxx                           2/2     Running   0          5m
```

## Step 3: Deploy SQL Managed Instance

Create a SQL Managed Instance on the Arc-enabled cluster:

```bash
# Create SQL Managed Instance
az sql mi-arc create \
  --name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --custom-location "arc-sql-location" \
  --storage-class-data $SQL_MI_STORAGE_CLASS \
  --storage-class-logs $SQL_MI_STORAGE_CLASS \
  --storage-class-backups $SQL_MI_STORAGE_CLASS \
  --volume-size-data 20Gi \
  --volume-size-logs 5Gi \
  --volume-size-backups 20Gi \
  --cores-limit 4 \
  --cores-request 2 \
  --memory-limit 8Gi \
  --memory-request 4Gi \
  --tier GeneralPurpose \
  --dev
```

> **Note:** The `--dev` flag deploys with reduced resource requirements suitable for development/workshop scenarios.

### Monitor SQL MI Deployment

```bash
# Watch SQL MI creation
kubectl get sqlmi -n $ARC_NAMESPACE --watch

# Check pod status
kubectl get pods -n $ARC_NAMESPACE | grep $SQL_MI_NAME

# View detailed status
az sql mi-arc show --name $SQL_MI_NAME --resource-group $RESOURCE_GROUP --output table
```

Wait until the SQL MI state shows `Ready`. This typically takes 5-15 minutes.

## Step 4: Retrieve Connection Information

```bash
# Get the SQL MI endpoint
export SQL_MI_ENDPOINT=$(az sql mi-arc show \
  --name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.k8SRaw.status.endpoints.primary" -o tsv)

echo "SQL MI Primary Endpoint: $SQL_MI_ENDPOINT"

# Get the external IP (if using LoadBalancer service)
kubectl get svc -n $ARC_NAMESPACE | grep $SQL_MI_NAME
```

## Step 5: Connect to SQL Managed Instance

### Option A: Using sqlcmd

```bash
# Install sqlcmd if not present
# Linux (Debian/Ubuntu)
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
echo "deb [signed-by=/usr/share/keyrings/microsoft-prod.gpg] https://packages.microsoft.com/ubuntu/$(lsb_release -rs)/prod $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/mssql-release.list
sudo apt-get update && sudo ACCEPT_EULA=Y apt-get install -y mssql-tools18

# Connect using sqlcmd
sqlcmd -S $SQL_MI_ENDPOINT -U <username> -P '<password>' -C
```

### Option B: Using Azure Data Studio

1. Open Azure Data Studio
2. Click **New Connection**
3. Set:
   - Server: `<SQL_MI_ENDPOINT>`
   - Authentication: SQL Login
   - User name: The admin username you specified
   - Password: The password you specified
4. Click **Connect**

### Option C: Port-Forward for Local Access

```bash
# Port-forward the SQL MI service to localhost
kubectl port-forward svc/${SQL_MI_NAME}-external-svc 1433:1433 -n $ARC_NAMESPACE &

# Connect via localhost
sqlcmd -S localhost,1433 -U <username> -P '<password>' -C
```

## Step 6: Create a Sample Database

Connect to the SQL MI and create a sample database for the replication exercise:

```sql
-- Create a sample database
CREATE DATABASE WorkshopDB;
GO

USE WorkshopDB;
GO

-- Create sample tables
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY IDENTITY(1,1),
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    Department NVARCHAR(50),
    HireDate DATE,
    Salary DECIMAL(10,2)
);
GO

CREATE TABLE Orders (
    OrderID INT PRIMARY KEY IDENTITY(1,1),
    EmployeeID INT FOREIGN KEY REFERENCES Employees(EmployeeID),
    OrderDate DATETIME2,
    Amount DECIMAL(10,2),
    Status NVARCHAR(20)
);
GO

-- Insert sample data
INSERT INTO Employees (FirstName, LastName, Department, HireDate, Salary)
VALUES 
    ('John', 'Smith', 'Engineering', '2020-01-15', 95000.00),
    ('Sarah', 'Johnson', 'Marketing', '2019-06-01', 85000.00),
    ('Michael', 'Williams', 'Engineering', '2021-03-10', 92000.00),
    ('Emily', 'Brown', 'Sales', '2018-11-20', 78000.00),
    ('David', 'Jones', 'Engineering', '2022-02-28', 98000.00);
GO

INSERT INTO Orders (EmployeeID, OrderDate, Amount, Status)
VALUES 
    (1, GETDATE(), 15000.00, 'Completed'),
    (2, GETDATE(), 8500.00, 'Pending'),
    (3, GETDATE(), 22000.00, 'Completed'),
    (4, GETDATE(), 5600.00, 'Processing'),
    (5, GETDATE(), 31000.00, 'Completed');
GO

-- Verify data
SELECT * FROM Employees;
SELECT * FROM Orders;
GO
```

## Step 7: Configure Database for Replication

Set the database to FULL recovery model (required for availability groups):

```sql
-- Set recovery model to FULL
ALTER DATABASE WorkshopDB SET RECOVERY FULL;
GO

-- Take a full backup (required before AG setup)
BACKUP DATABASE WorkshopDB 
TO DISK = '/var/opt/mssql/backups/WorkshopDB_Full.bak'
WITH FORMAT, INIT, COMPRESSION;
GO

-- Verify recovery model
SELECT name, recovery_model_desc 
FROM sys.databases 
WHERE name = 'WorkshopDB';
GO
```

## Step 8: Verify Deployment

```bash
echo "=== Data Controller Status ==="
az arcdata dc status show --name $DATA_CONTROLLER_NAME --resource-group $RESOURCE_GROUP \
  --query "properties.k8SRaw.status.state" -o tsv

echo "=== SQL MI Status ==="
az sql mi-arc show --name $SQL_MI_NAME --resource-group $RESOURCE_GROUP \
  --query "properties.k8SRaw.status.state" -o tsv

echo "=== SQL MI Endpoint ==="
az sql mi-arc show --name $SQL_MI_NAME --resource-group $RESOURCE_GROUP \
  --query "properties.k8SRaw.status.endpoints.primary" -o tsv

echo "=== Pods Running ==="
kubectl get pods -n $ARC_NAMESPACE -o wide
```

## Understanding the Deployment

### What Was Created

| Component | Description |
|-----------|-------------|
| Data Controller | Manages lifecycle, monitoring, and operations for data services |
| SQL Managed Instance | Container-based SQL Server with Azure management |
| Persistent Volumes | Storage for data, logs, and backups |
| Services | Kubernetes services for SQL connectivity |
| Monitoring | Built-in Grafana and Kibana dashboards |

### Storage Architecture

```
┌─────────────────────────────────────────┐
│           SQL Managed Instance           │
├─────────────────────────────────────────┤
│  Data Volume (20Gi)     - PVC           │
│  Log Volume (5Gi)       - PVC           │
│  Backup Volume (20Gi)   - PVC           │
└─────────────────────────────────────────┘
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Data controller stuck in provisioning | Check pod logs: `kubectl logs -n $ARC_NAMESPACE <control-pod>` |
| SQL MI pod in Pending state | Check PVC status: `kubectl get pvc -n $ARC_NAMESPACE` |
| Cannot connect to SQL MI | Verify service type and external IP: `kubectl get svc -n $ARC_NAMESPACE` |
| Insufficient resources | Scale cluster or reduce resource requests in SQL MI spec |

### Checking Logs

```bash
# Data controller logs
kubectl logs -n $ARC_NAMESPACE deployment/control --all-containers

# SQL MI logs
kubectl logs -n $ARC_NAMESPACE ${SQL_MI_NAME}-0 --all-containers

# Events in namespace
kubectl get events -n $ARC_NAMESPACE --sort-by='.lastTimestamp'
```

---

**Previous:** [Module 2: Connect Kubernetes to Azure Arc](02-arc-enable-kubernetes.md)  
**Next:** [Module 4: Setup Replication with Managed Instance Link](04-setup-replication.md)
