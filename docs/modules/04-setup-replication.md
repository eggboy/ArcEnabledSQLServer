# Module 4: Setup Replication with Managed Instance Link

## Objective

Establish near real-time data replication between your Arc-enabled SQL Managed Instance (on-premises/edge) and Azure SQL Managed Instance in the cloud using the Managed Instance Link feature with distributed availability groups.

## Overview

The **Managed Instance Link** creates a distributed availability group between:
- **Primary**: Your Arc-enabled SQL Managed Instance (on your Kubernetes cluster)
- **Secondary**: Azure SQL Managed Instance (in Azure cloud)

This provides:
- Near real-time data replication
- Disaster recovery capability
- Read-scale offloading to Azure
- Migration pathway to Azure

## Architecture

```
┌────────────────────────────────┐         ┌────────────────────────────────┐
│  Arc SQL MI (Primary)          │         │  Azure SQL MI (Secondary)      │
│  On-Premises / Edge            │         │  Azure Cloud                   │
│                                │         │                                │
│  ┌──────────────────────────┐  │   AG    │  ┌──────────────────────────┐  │
│  │  WorkshopDB              │──┼────────►│  │  WorkshopDB (Read-Only)  │  │
│  │  (Read-Write)            │  │  Link   │  │  (Synchronized)          │  │
│  └──────────────────────────┘  │         │  └──────────────────────────┘  │
│                                │         │                                │
└────────────────────────────────┘         └────────────────────────────────┘
```

## Prerequisites

Before starting this module, ensure:
- [x] Module 3 completed - SQL MI deployed and running
- [x] `WorkshopDB` database created with FULL recovery model
- [x] Full backup of `WorkshopDB` taken
- [ ] Azure SQL Managed Instance provisioned in Azure (Step 1 below)

## Step 1: Provision Azure SQL Managed Instance (Target)

Create an Azure SQL Managed Instance as the replication target:

```bash
source workshop-env.sh

# Create a VNet for the Azure SQL MI (if not existing)
az network vnet create \
  --name sql-mi-vnet \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --address-prefix 10.0.0.0/16

# Create a subnet (dedicated for SQL MI)
az network vnet subnet create \
  --name sql-mi-subnet \
  --resource-group $RESOURCE_GROUP \
  --vnet-name sql-mi-vnet \
  --address-prefix 10.0.0.0/24 \
  --delegations Microsoft.Sql/managedInstances

# Set credentials (use secure methods in production - e.g., Azure Key Vault)
export SQL_ADMIN_USER="sqladmin"
read -sp "Enter SQL MI admin password: " SQL_ADMIN_PASSWORD && echo

# Create the Azure SQL Managed Instance
az sql mi create \
  --name $AZURE_SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --admin-user $SQL_ADMIN_USER \
  --admin-password "$SQL_ADMIN_PASSWORD" \
  --subnet /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Network/virtualNetworks/sql-mi-vnet/subnets/sql-mi-subnet \
  --capacity 4 \
  --storage 32GB \
  --edition GeneralPurpose \
  --family Gen5
```

> **Note:** Azure SQL MI provisioning can take 30-60 minutes. Continue with network setup while waiting.

> **Security Note:** Never store passwords in scripts or commit them to source control. Use Azure Key Vault, environment variables, or interactive prompts for credentials in production environments. Record your admin username and password securely — you will need them for connection steps later.

## Step 2: Configure Network Connectivity

Establish network connectivity between your on-premises cluster and Azure SQL MI:

### Option A: VPN Gateway (Production)

```bash
# Create VPN Gateway for site-to-site connectivity
az network vnet-gateway create \
  --name arc-vpn-gateway \
  --resource-group $RESOURCE_GROUP \
  --vnet sql-mi-vnet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1 \
  --location $LOCATION
```

### Option B: Public Endpoint (Workshop/Dev only)

For workshop purposes, enable the public endpoint on Azure SQL MI:

```bash
# Enable public endpoint on Azure SQL MI
az sql mi update \
  --name $AZURE_SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --public-data-endpoint-enabled true
```

### Configure NSG Rules

First, find the NSG associated with your SQL MI subnet:

```bash
# Get the NSG name associated with the SQL MI subnet
export SQL_MI_NSG_NAME=$(az network vnet subnet show \
  --resource-group $RESOURCE_GROUP \
  --vnet-name sql-mi-vnet \
  --name sql-mi-subnet \
  --query "networkSecurityGroup.id" -o tsv | xargs -I {} basename {})

echo "NSG Name: $SQL_MI_NSG_NAME"

# If no NSG exists, create one and associate it
if [ -z "$SQL_MI_NSG_NAME" ]; then
  az network nsg create --name sql-mi-nsg --resource-group $RESOURCE_GROUP --location $LOCATION
  az network vnet subnet update \
    --resource-group $RESOURCE_GROUP \
    --vnet-name sql-mi-vnet \
    --name sql-mi-subnet \
    --network-security-group sql-mi-nsg
  export SQL_MI_NSG_NAME="sql-mi-nsg"
fi

# Allow inbound traffic on port 5022 (AG endpoint)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $SQL_MI_NSG_NAME \
  --name AllowAGEndpoint \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 5022

# Allow inbound traffic on ports 11000-11999 (redirect)
az network nsg rule create \
  --resource-group $RESOURCE_GROUP \
  --nsg-name $SQL_MI_NSG_NAME \
  --name AllowRedirect \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 11000-11999
```

## Step 3: Configure SQL MI for Availability Groups

Connect to your Arc SQL MI and enable the AG endpoint:

```sql
-- Connect to Arc SQL MI
-- Enable the AG endpoint (typically already enabled on Arc SQL MI)
-- Verify AG endpoint
SELECT name, state_desc, port 
FROM sys.tcp_endpoints 
WHERE type_desc = 'DATABASE_MIRRORING';
GO

-- Verify the database is ready
SELECT name, recovery_model_desc, state_desc
FROM sys.databases 
WHERE name = 'WorkshopDB';
GO
```

## Step 4: Create the Managed Instance Link

### Using Azure CLI

```bash
# Get the Azure SQL MI resource ID
export AZURE_SQL_MI_ID=$(az sql mi show \
  --name $AZURE_SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Get the Azure SQL MI FQDN (public endpoint)
export AZURE_SQL_MI_FQDN=$(az sql mi show \
  --name $AZURE_SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --query fullyQualifiedDomainName -o tsv)

# Derive the public endpoint hostname (replaces first '.' with '.public.')
export AZURE_SQL_MI_PUBLIC_FQDN=$(echo $AZURE_SQL_MI_FQDN | sed 's/\./.public./')

echo "Azure SQL MI Public Endpoint: $AZURE_SQL_MI_PUBLIC_FQDN"

# Create the Managed Instance Link
az sql mi-arc dag create \
  --name "workshop-dag" \
  --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --partner-mi-id $AZURE_SQL_MI_ID \
  --partner-endpoint "tcp://${AZURE_SQL_MI_PUBLIC_FQDN}:3342" \
  --database-name "WorkshopDB"
```

### Using Azure Portal (Alternative)

1. Navigate to your **Azure SQL Managed Instance** in the portal
2. Go to **Managed Instance Link** under Settings
3. Click **+ New link**
4. Select your Arc-enabled SQL MI as the source
5. Select the database (`WorkshopDB`)
6. Follow the wizard to configure:
   - Network connectivity validation
   - Certificate exchange (automated)
   - Link creation

## Step 5: Monitor Replication Status

### Check Link Status via CLI

```bash
# View the distributed availability group status
az sql mi-arc dag show \
  --name "workshop-dag" \
  --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table
```

### Check from SQL MI (Primary)

```sql
-- Check AG health on primary (Arc SQL MI)
SELECT 
    ag.name AS ag_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    ars.role_desc,
    ars.synchronization_health_desc,
    ars.connected_state_desc
FROM sys.dm_hadr_availability_replica_states ars
JOIN sys.availability_replicas ar ON ars.replica_id = ar.replica_id
JOIN sys.availability_groups ag ON ar.group_id = ag.group_id;
GO

-- Check database replication status
SELECT 
    d.name AS database_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.last_commit_time,
    drs.log_send_queue_size,
    drs.redo_queue_size
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.databases d ON drs.database_id = d.database_id
WHERE drs.is_local = 1;
GO
```

### Check from Azure SQL MI (Secondary)

Connect to your Azure SQL MI and verify data is replicating:

```sql
-- Connect to Azure SQL MI
-- Verify the database exists and is synchronized
SELECT name, state_desc, is_read_only
FROM sys.databases
WHERE name = 'WorkshopDB';
GO

-- Check replica status
SELECT 
    synchronization_state_desc,
    synchronization_health_desc,
    database_state_desc,
    last_received_time,
    last_redone_time
FROM sys.dm_hadr_database_replica_states;
GO
```

## Step 6: Test Replication

### Insert Data on Primary

```sql
-- Connect to Arc SQL MI (Primary)
USE WorkshopDB;
GO

-- Insert new records
INSERT INTO Employees (FirstName, LastName, Department, HireDate, Salary)
VALUES ('Alice', 'TestReplication', 'QA', GETDATE(), 88000.00);
GO

INSERT INTO Orders (EmployeeID, OrderDate, Amount, Status)
VALUES (6, GETDATE(), 12500.00, 'New');
GO

-- Note the last inserted values
SELECT TOP 1 * FROM Employees ORDER BY EmployeeID DESC;
SELECT TOP 1 * FROM Orders ORDER BY OrderID DESC;
GO
```

### Verify on Secondary

```sql
-- Connect to Azure SQL MI (Secondary)
USE WorkshopDB;
GO

-- Verify the replicated data appears
-- (May take a few seconds for near real-time replication)
SELECT TOP 1 * FROM Employees ORDER BY EmployeeID DESC;
SELECT TOP 1 * FROM Orders ORDER BY OrderID DESC;
GO
```

You should see the same data on both instances, confirming replication is working.

## Step 7: Failover (Optional)

To perform a planned failover to Azure SQL MI:

```bash
# Initiate planned failover to Azure SQL MI
az sql mi-arc dag update \
  --name "workshop-dag" \
  --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --role secondary
```

> **Warning:** After failover, the Azure SQL MI becomes primary (read-write) and the Arc SQL MI becomes secondary (read-only).

### Failback (SQL Server 2022+)

```bash
# Failback to Arc SQL MI
az sql mi-arc dag update \
  --name "workshop-dag" \
  --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --role primary
```

## Validation

```bash
echo "=== DAG Status ==="
az sql mi-arc dag show --name "workshop-dag" \
  --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.dagStatus" -o tsv

echo "=== Replication Mode ==="
az sql mi-arc dag show --name "workshop-dag" \
  --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.replicationMode" -o tsv

echo "=== Link Health ==="
az sql mi-arc dag show --name "workshop-dag" \
  --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "properties.healthStatus" -o tsv
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Link creation fails | Verify network connectivity between Arc SQL MI and Azure SQL MI |
| Synchronization unhealthy | Check firewall rules for ports 5022 and 11000-11999 |
| High replication lag | Check network bandwidth and latency between sites |
| Certificate errors | Ensure certificates are properly exchanged (portal wizard handles this) |
| Database not in FULL recovery | `ALTER DATABASE WorkshopDB SET RECOVERY FULL` |

### Checking Connectivity

```bash
# Retrieve the actual FQDN for connectivity checks
export AZURE_SQL_MI_FQDN=$(az sql mi show \
  --name $AZURE_SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --query fullyQualifiedDomainName -o tsv)
export AZURE_SQL_MI_PUBLIC_FQDN=$(echo $AZURE_SQL_MI_FQDN | sed 's/\./.public./')

# Test connectivity from Arc SQL MI pod to Azure SQL MI
kubectl exec -n $ARC_NAMESPACE ${SQL_MI_NAME}-0 -- \
  /opt/mssql-tools/bin/sqlcmd -S ${AZURE_SQL_MI_PUBLIC_FQDN},3342 \
  -U sqladmin -P '<password>' -Q "SELECT 1" -C

# Check if AG endpoint port is accessible
kubectl exec -n $ARC_NAMESPACE ${SQL_MI_NAME}-0 -- \
  nc -zv ${AZURE_SQL_MI_PUBLIC_FQDN} 5022
```

---

**Previous:** [Module 3: Deploy Arc Data Services and SQL Managed Instance](03-deploy-sql-server.md)  
**Next:** [Module 5: Monitoring, Validation, and Operations](05-monitoring-operations.md)
