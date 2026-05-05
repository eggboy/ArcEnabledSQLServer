# Hybrid database cloud with Azure Database for SQL and SQL Server enabled by Azure Arc

A hands-on workshop demonstrating the hybrid cloud model with Azure Arc enabled SQL Server, Arc enabled Kubernetes, and data replication to Azure SQL Managed Instance.

## Workshop Overview

This workshop guides you through deploying and managing SQL Server in a hybrid cloud environment using Azure Arc. You will connect an existing Kubernetes cluster to Azure Arc, deploy SQL Server data services, and set up near real-time replication to Azure SQL Managed Instance using the Managed Instance Link feature.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Azure Cloud                                  │
│  ┌──────────────────┐    ┌──────────────────────────────────────┐   │
│  │  Azure Portal    │    │  Azure SQL Managed Instance          │   │
│  │  (Management)    │    │  (DR / Read-Scale Secondary)         │   │
│  └──────────────────┘    └──────────────────────────────────────┘   │
│           │                            ▲                             │
│           │                            │ Managed Instance Link       │
│           │                            │ (Distributed AG)            │
└───────────┼────────────────────────────┼────────────────────────────┘
            │                            │
            ▼                            │
┌───────────────────────────────────────────────────────────────────────┐
│                  On-Premises / Other Cloud                             │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │            Kubernetes Cluster (Arc Enabled)                      │  │
│  │  ┌──────────────────┐   ┌────────────────────────────────────┐ │  │
│  │  │  Azure Arc Agent │   │  Arc Data Controller               │ │  │
│  │  └──────────────────┘   └────────────────────────────────────┘ │  │
│  │                          ┌────────────────────────────────────┐ │  │
│  │                          │  SQL Managed Instance (Primary)    │ │  │
│  │                          └────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- An existing vanilla Kubernetes cluster (AKS, EKS, GKE, or on-premises K8s)
- Azure subscription with Owner or Contributor access
- Azure CLI (v2.60+) with extensions installed
- `kubectl` configured and connected to your cluster
- Network connectivity from the cluster to Azure endpoints (port 443)

## Workshop Modules

| Module | Title | Duration |
|--------|-------|----------|
| 1 | [Prerequisites and Environment Setup](docs/modules/01-prerequisites.md) | 30 min |
| 2 | [Connect Kubernetes to Azure Arc](docs/modules/02-arc-enable-kubernetes.md) | 20 min |
| 3 | [Deploy Arc Data Services and SQL Managed Instance](docs/modules/03-deploy-sql-server.md) | 45 min |
| 4 | [Setup Replication with Managed Instance Link](docs/modules/04-setup-replication.md) | 45 min |
| 5 | [Monitoring, Validation, and Operations](docs/modules/05-monitoring-operations.md) | 30 min |

**Total Duration: ~3 hours**

## Getting Started

Start with [Module 1: Prerequisites and Environment Setup](docs/modules/01-prerequisites.md) to prepare your environment.

## Key Concepts

- **Azure Arc**: Extends Azure management to any infrastructure (on-premises, multi-cloud, edge)
- **Arc Enabled Kubernetes**: Connects existing Kubernetes clusters to Azure for centralized management
- **Arc Data Services**: Runs Azure data services (SQL MI, PostgreSQL) on any Kubernetes cluster
- **Managed Instance Link**: Provides near real-time replication between Arc-enabled SQL Managed Instance and Azure SQL Managed Instance using distributed availability groups

## References

- [Azure Arc Documentation](https://learn.microsoft.com/en-us/azure/azure-arc/)
- [Azure Arc-enabled Data Services](https://learn.microsoft.com/en-us/azure/azure-arc/data/)
- [Azure Arc-enabled SQL Server](https://learn.microsoft.com/en-us/sql/sql-server/azure-arc/overview)
- [Managed Instance Link Overview](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-feature-overview)
- [Azure Arc-enabled Kubernetes](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview)
