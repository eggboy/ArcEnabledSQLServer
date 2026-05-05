# Module 5: Monitoring, Validation, and Operations

## Objective

Learn how to monitor your Arc-enabled SQL Managed Instance deployment, set up alerting, and perform common operational tasks using Azure's unified management plane.

## Overview

Azure Arc provides a single pane of glass for managing your SQL infrastructure regardless of where it runs. This module covers:
- Built-in monitoring dashboards (Grafana/Kibana)
- Azure Monitor integration
- Azure Policy for governance
- Backup and restore operations
- Scaling and maintenance

## Step 1: Access Built-in Monitoring Dashboards

Arc Data Services include Grafana (metrics) and Kibana (logs) dashboards:

```bash
source workshop-env.sh

# Get Grafana endpoint
kubectl get svc -n $ARC_NAMESPACE | grep metricsui

# Get Kibana endpoint
kubectl get svc -n $ARC_NAMESPACE | grep logsui

# Port-forward Grafana for local access
kubectl port-forward svc/metricsui-external-svc 3000:3000 -n $ARC_NAMESPACE &

# Port-forward Kibana for local access
kubectl port-forward svc/logsui-external-svc 5601:5601 -n $ARC_NAMESPACE &
```

Access the dashboards:
- **Grafana**: `http://localhost:3000` - SQL MI performance metrics
- **Kibana**: `http://localhost:5601` - SQL MI logs and diagnostics

### Key Grafana Dashboards

| Dashboard | What it Shows |
|-----------|--------------|
| SQL MI Overview | CPU, memory, storage utilization |
| SQL MI Performance | Query performance, wait stats |
| SQL MI Availability | AG health, replication lag |
| Infrastructure | Node and pod-level metrics |

## Step 2: Configure Azure Monitor Integration

### Create Log Analytics Workspace

Create the workspace first so it can be referenced when installing the monitoring extension:

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --workspace-name "arc-sql-logs" \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Get workspace resource ID
export WORKSPACE_RESOURCE_ID=$(az monitor log-analytics workspace show \
  --workspace-name "arc-sql-logs" \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

echo "Workspace Resource ID: $WORKSPACE_RESOURCE_ID"
```

### Enable Monitoring Extension

```bash
# Install Azure Monitor extension on the Arc-enabled cluster, linked to the workspace
az k8s-extension create \
  --name azuremonitor-containers \
  --extension-type Microsoft.AzureMonitor.Containers \
  --cluster-type connectedClusters \
  --cluster-name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --configuration-settings "logAnalyticsWorkspaceResourceID=${WORKSPACE_RESOURCE_ID}"
```

### Query SQL MI Metrics in Azure Monitor

```kusto
// Sample KQL query - SQL MI CPU utilization
InsightsMetrics
| where Namespace == "sqlmi"
| where Name == "cpu_percent"
| summarize avg(Val) by bin(TimeGenerated, 5m), Computer
| render timechart

// Sample KQL query - Replication lag
InsightsMetrics
| where Namespace == "sqlmi"
| where Name == "log_send_queue_size"
| summarize max(Val) by bin(TimeGenerated, 1m)
| render timechart
```

## Step 3: Set Up Alerts

### Alert on High CPU

```bash
# Create an alert rule for high CPU
az monitor metrics alert create \
  --name "arc-sql-high-cpu" \
  --resource-group $RESOURCE_GROUP \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.AzureArcData/sqlManagedInstances/$SQL_MI_NAME" \
  --condition "avg cpu_percent > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --description "Alert when Arc SQL MI CPU exceeds 80%"
```

### Alert on Replication Lag

```bash
# Create an alert for replication health
az monitor metrics alert create \
  --name "arc-sql-replication-lag" \
  --resource-group $RESOURCE_GROUP \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.AzureArcData/sqlManagedInstances/$SQL_MI_NAME" \
  --condition "max log_send_queue_size > 500" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --description "Alert when replication lag exceeds threshold"
```

## Step 4: Azure Policy for Governance

Apply governance policies to your Arc-enabled SQL resources:

```bash
# Assign built-in policy: SQL MI should have TDE enabled
az policy assignment create \
  --name "arc-sql-tde-policy" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/<policy-definition-id>" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"

# List available Arc SQL policies
az policy definition list \
  --query "[?contains(displayName, 'Arc') && contains(displayName, 'SQL')].[displayName, name]" \
  --output table
```

### Common Governance Policies

| Policy | Purpose |
|--------|---------|
| Require TDE encryption | Ensure data at rest is encrypted |
| Enforce backup retention | Minimum backup retention period |
| Deny public endpoints | Network security |
| Require audit logging | Compliance |

## Step 5: Backup and Restore Operations

### Automatic Backups

Arc SQL MI includes automatic backups. Configure retention:

```bash
# Set backup retention to 7 days
az sql mi-arc update \
  --name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --retention-days 7
```

### Manual Backup

```sql
-- Connect to Arc SQL MI
-- Perform a manual backup
BACKUP DATABASE WorkshopDB
TO DISK = '/var/opt/mssql/backups/WorkshopDB_Manual.bak'
WITH FORMAT, INIT, COMPRESSION,
NAME = 'WorkshopDB Manual Backup';
GO

-- Verify backups
SELECT 
    database_name,
    backup_start_date,
    backup_finish_date,
    type,
    compressed_backup_size / 1024 / 1024 AS size_mb
FROM msdb.dbo.backupset
WHERE database_name = 'WorkshopDB'
ORDER BY backup_start_date DESC;
GO
```

### Point-in-Time Restore

```bash
# Restore to a specific point in time
az sql mi-arc restore \
  --name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --dest-name "${SQL_MI_NAME}-restored" \
  --time "2025-01-15T10:00:00Z"
```

## Step 6: Scaling Operations

### Scale Up (Vertical)

```bash
# Increase CPU and memory
az sql mi-arc update \
  --name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --cores-limit 8 \
  --cores-request 4 \
  --memory-limit 16Gi \
  --memory-request 8Gi
```

### Scale Storage

```bash
# Increase data storage
az sql mi-arc update \
  --name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --volume-size-data 50Gi
```

> **Note:** Storage can only be scaled up, not down.

## Step 7: Maintenance and Updates

### Check Available Updates

```bash
# List available updates for the data controller
az arcdata dc list-upgrades \
  --resource-group $RESOURCE_GROUP
```

### Perform Upgrade

```bash
# Upgrade the data controller (must be done first)
az arcdata dc upgrade \
  --name $DATA_CONTROLLER_NAME \
  --resource-group $RESOURCE_GROUP \
  --desired-version <target-version>

# Then upgrade SQL MI
az sql mi-arc upgrade \
  --name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP \
  --desired-version <target-version>
```

## Step 8: Security Best Practices

### Enable Transparent Data Encryption (TDE)

```sql
-- Enable TDE on WorkshopDB
USE master;
GO

CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<StrongMasterKeyPassword!>';
GO

CREATE CERTIFICATE TDE_Cert WITH SUBJECT = 'TDE Certificate';
GO

USE WorkshopDB;
GO

CREATE DATABASE ENCRYPTION KEY
WITH ALGORITHM = AES_256
ENCRYPTION BY SERVER CERTIFICATE TDE_Cert;
GO

ALTER DATABASE WorkshopDB SET ENCRYPTION ON;
GO

-- Verify TDE status
SELECT db.name, db.is_encrypted, dm.encryption_state
FROM sys.databases db
LEFT JOIN sys.dm_database_encryption_keys dm ON db.database_id = dm.database_id
WHERE db.name = 'WorkshopDB';
GO
```

### Enable Auditing

```sql
-- Create server audit
CREATE SERVER AUDIT ArcSQLAudit
TO FILE (FILEPATH = '/var/opt/mssql/audit/', MAXSIZE = 100 MB);
GO

ALTER SERVER AUDIT ArcSQLAudit WITH (STATE = ON);
GO

-- Create audit specification
CREATE SERVER AUDIT SPECIFICATION ArcSQLAuditSpec
FOR SERVER AUDIT ArcSQLAudit
ADD (FAILED_LOGIN_GROUP),
ADD (DATABASE_CHANGE_GROUP),
ADD (SCHEMA_OBJECT_CHANGE_GROUP);
GO

ALTER SERVER AUDIT SPECIFICATION ArcSQLAuditSpec WITH (STATE = ON);
GO
```

## Step 9: Final Validation - End-to-End Health Check

Run this comprehensive health check to validate the entire deployment:

```bash
#!/bin/bash
echo "============================================"
echo " Arc Enabled SQL Server - Health Check"
echo "============================================"

source workshop-env.sh

echo ""
echo "1. Arc Cluster Connection"
echo "-------------------------"
az connectedk8s show --name $ARC_CLUSTER_NAME --resource-group $RESOURCE_GROUP \
  --query "{Name:name, Status:connectivityStatus, Version:agentVersion}" -o table

echo ""
echo "2. Data Controller Status"
echo "-------------------------"
az arcdata dc status show --name $DATA_CONTROLLER_NAME --resource-group $RESOURCE_GROUP \
  --query "{Name:name, State:properties.k8SRaw.status.state}" -o table 2>/dev/null || echo "Check via kubectl"

echo ""
echo "3. SQL Managed Instance Status"
echo "------------------------------"
az sql mi-arc show --name $SQL_MI_NAME --resource-group $RESOURCE_GROUP \
  --query "{Name:name, State:properties.k8SRaw.status.state, Endpoint:properties.k8SRaw.status.endpoints.primary}" -o table

echo ""
echo "4. Replication Link Status"
echo "--------------------------"
az sql mi-arc dag show --name "workshop-dag" --mi-name $SQL_MI_NAME \
  --resource-group $RESOURCE_GROUP -o table 2>/dev/null || echo "DAG not configured or check status"

echo ""
echo "5. Pod Health"
echo "-------------"
echo "Total pods in ${ARC_NAMESPACE}:"
kubectl get pods -n $ARC_NAMESPACE --no-headers | wc -l
echo "Running pods:"
kubectl get pods -n $ARC_NAMESPACE --field-selector=status.phase=Running --no-headers | wc -l

echo ""
echo "6. Azure SQL MI (Target) Status"
echo "--------------------------------"
az sql mi show --name $AZURE_SQL_MI_NAME --resource-group $RESOURCE_GROUP \
  --query "{Name:name, State:state, FQDN:fullyQualifiedDomainName}" -o table 2>/dev/null || echo "Azure SQL MI not yet provisioned"

echo ""
echo "============================================"
echo " Health Check Complete"
echo "============================================"
```

## Summary

In this module, you learned how to:

- ✅ Access built-in Grafana and Kibana dashboards
- ✅ Integrate with Azure Monitor for centralized monitoring
- ✅ Set up alerts for CPU and replication lag
- ✅ Apply Azure Policy for governance
- ✅ Perform backup and restore operations
- ✅ Scale your SQL MI resources
- ✅ Apply security best practices (TDE, Auditing)
- ✅ Run end-to-end health checks

## Workshop Cleanup

> **⚠️ Warning:** The following commands will permanently delete all resources created during this workshop, including databases, networking, and monitoring data. Ensure you have exported any data you need before proceeding.

When you're done with the workshop, clean up resources:

```bash
# Delete the distributed availability group link
az sql mi-arc dag delete --name "workshop-dag" --mi-name $SQL_MI_NAME --resource-group $RESOURCE_GROUP

# Delete Arc SQL MI
az sql mi-arc delete --name $SQL_MI_NAME --resource-group $RESOURCE_GROUP

# Delete Data Controller
az arcdata dc delete --name $DATA_CONTROLLER_NAME --resource-group $RESOURCE_GROUP

# Delete Arc data services extension
az k8s-extension delete --name arc-data-services \
  --cluster-type connectedClusters \
  --cluster-name $ARC_CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP

# Disconnect Kubernetes from Arc
az connectedk8s delete --name $ARC_CLUSTER_NAME --resource-group $RESOURCE_GROUP

# Delete Azure SQL MI (target)
az sql mi delete --name $AZURE_SQL_MI_NAME --resource-group $RESOURCE_GROUP

# Delete the resource group (removes everything)
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

---

**Previous:** [Module 4: Setup Replication with Managed Instance Link](04-setup-replication.md)

## Congratulations! 🎉

You have successfully completed the Azure Arc Enabled SQL Server workshop. You now have hands-on experience with:

1. **Arc-enabled Kubernetes** - Connecting any K8s cluster to Azure
2. **Arc Data Services** - Running Azure-managed SQL on your infrastructure  
3. **Managed Instance Link** - Real-time replication to Azure for DR/migration
4. **Hybrid Operations** - Monitoring, scaling, and governing from Azure

This hybrid cloud model enables you to keep data where it needs to be while leveraging Azure's management, security, and disaster recovery capabilities.
