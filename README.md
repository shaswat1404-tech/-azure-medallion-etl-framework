# Azure Medallion ETL Framework
### Multi-Source Data Platform | Senior Housing / Property Management Industry

> Production-grade data platform ingesting from multiple heterogeneous sources — property management REST APIs, an on-premises SQL Server, and a client-managed PostgreSQL MDM database — through a medallion lakehouse architecture on Azure Databricks, served to Power BI via a SQL Warehouse endpoint.

---

## Overview

This project delivers a unified operational analytics platform for the Senior Housing / Property Management industry. Data arrives from structurally different sources on different schedules, with different load strategies — the platform normalises all of it into a consistent medallion layer and surfaces KPIs for business consumption.

**Sources:**
- Property management REST APIs — ManageAmerica, Rent Manager, HubSpot, Airtable, and others
- On-premises SQL Server — operational data not exposed via API, ingested via ADF into Blob staging
- PostgreSQL (client MDM) — master/mapping tables (entity hierarchies, code lookups) not available from any API

**Serving layer:** Delta tables → Databricks SQL Warehouse → Power BI
*(SQL Warehouse chosen over all-purpose clusters as Power BI runs read-only analytical queries — SQL Warehouse is purpose-built and cost-optimised for this pattern)*

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        SOURCE LAYER                         │
│                                                             │
│  Property Mgmt APIs     On-Prem SQL Server     PostgreSQL   │
│  (MA, RM, etc.         (ADF Self-Hosted IR    (MDM: master  │
│    )                     → Blob staging)        & mapping   │
│                                                  tables)    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    INGESTION LAYER                          │
│                                                             │
│         Azure Function App (Durable Functions)              │
│         ┌─────────────────────────────────────┐             │
│         │  HTTP Starter                       │             │
│         │  Receives: source, date, load flags │             │
│         │  Starts: new Orchestrator instance  │             │
│         └──────────────┬──────────────────────┘             │
│                        │                                    │
│         ┌──────────────▼──────────────────────┐             │
│         │  Orchestrator Function              │             │
│         │  Routes by: source name + flags     │             │
│         │  (itd_flag, is_ops, extractall...)  │             │
│         └──────────────┬──────────────────────┘             │
│                        │                                    │
│         ┌──────────────▼──────────────────────┐             │
│         │  Activity Functions                 │             │
│         │  - Per source (MA, RM, ...)         │             │
│         │  - Per load type (full / incremental│             │
│         │    / inception-to-date)             │             │
│         │  - Per domain (ops / finance)       │             │
│         │  - Shared helper classes (.py)      │             │
│         └──────────────┬──────────────────────┘             │
│                        │                                    │
│                        ▼                                    │
│                 Azure Blob Storage                          │
│                 (raw landing zone)                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    MEDALLION LAYERS                         │
│                  (Azure Databricks)                         │
│                                                             │
│   BRONZE  ── Raw ingested data, schema-on-read,             │
│              audit columns, no business logic               │
│      │                                                      │
│   SILVER  ── Cleansed, validated, joined datasets,          │
│              MDM/mapping tables applied (from PostgreSQL)   │
│      │                                                      │
│   GOLD    ── Aggregated KPIs, business-ready models         │
│              Delinquency tracking, inventory control,       │
│              financial summaries                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    SERVING LAYER                            │
│                                                             │
│         Databricks SQL Warehouse                            │
│         (optimised for BI read queries)                     │
│                    │                                        │
│                    ▼                                        │
│              Power BI Reports & Dashboards                  │
└─────────────────────────────────────────────────────────────┘

         Throughout: Azure Data Factory (orchestration)
                   + Logic Apps (monitoring & alerting)
```

---

## Function App Design

The ingestion layer is built on **Azure Durable Functions** — chosen because property management APIs vary significantly in response times and payload sizes, and standard HTTP-triggered functions would time out on heavier endpoints.

### Parallelism via ADF

**ADF dispatches multiple HTTP requests in parallel** to the same Function App, each carrying a different set of parameters. Each request spins up an **independent Durable Orchestrator instance** — so multiple sources and domains are ingested concurrently without interfering with each other:

```
ADF Pipeline (parallel Web Activities)
  ├── → HTTP Starter (source=rentmanager, is_ops=1, itd_flag=0)  → Orchestrator Instance A
  ├── → HTTP Starter (source=rentmanager, is_ops=0, itd_flag=0)  → Orchestrator Instance B
  ├── → HTTP Starter (source=manageamerica, ...)                 → Orchestrator Instance C
```

Each instance runs independently — a failure in one does not block others.

### Routing Logic

The **HTTP Starter** accepts parameters (`source`, `yyyymmdd`, `itd_flag`, `is_ops`, `extractall`, `hh`, etc.) and immediately returns a status-check URL — the caller is non-blocking. The **Orchestrator** routes to the correct Activity Function(s) based on `source`, with further sub-routing via flags for load type and domain:

> Sequential execution within a branch is intentional where downstream activity results depend on upstream data being landed first.

### Code Structure

```
function_app/
├── func-starter-extraction/        # HTTP trigger — accepts params, starts orchestrator
├── func-orchestrator-extraction/   # Durable orchestrator — routes by source + flags
├── ma-hourly-api-loads/            # Activity: ManageAmerica incremental loads
├── get_ops_controllers/            # Activity: Rent Manager — ops controllers (incremental)
├── get_ops_controllers_full_load/  # Activity: Rent Manager — ops controllers (full load)
├── get-local-attractions/          # Activity: local attractions / reference data
└── shared/
    └── helpers.py                  # Shared classes and utility functions
                                    # (API clients, Blob writers, error handlers, etc.)
```

---

## On-Premises SQL Server Ingestion

Data not available via API is pulled from an on-premises SQL Server using **ADF with a Self-Hosted Integration Runtime (SHIR)**:

```
On-Prem SQL Server
       │
  ADF + Self-Hosted IR  ←— bridges on-prem network to Azure
       │
  Azure Blob Storage (staging)
       │
  Databricks Bronze notebook
  (same medallion processing as API-sourced data)
```

Runs as a separate ADF pipeline — staged to Blob before being picked up by the medallion layer, keeping ingestion concerns cleanly separated from transformation.

---

## PostgreSQL MDM Integration

The client maintains a **PostgreSQL database as their MDM (Master Data Management) system** — containing entity hierarchies, code-to-description mappings, and reference data that no API exposes. This data is critical at the Silver layer to correctly join and interpret operational records.

- Ingested via ADF into Azure Blob Storage (staging) and then Databricks Bronze notebook
- Applied at the **Silver layer** as lookup and mapping joins
- Kept separate from operational data to maintain a clean MDM boundary
- Ensures consistent entity resolution across all API sources

---

## Secrets Management

No credentials appear in source code at any layer:

| Layer | Mechanism |
|---|---|
| Databricks notebooks | Azure Key Vault-backed secret scope (`dbutils.secrets.get`) |
| Function App | Key Vault references via App Settings / environment variables |
| ADF linked services | Managed Identity + Key Vault reference for all connection strings |

---

## Monitoring & Alerting

- **Logic Apps** trigger on ADF pipeline failure events — alerts include pipeline name, run ID, error message, and timestamp
- Each Durable Orchestrator instance exposes a **custom status field** (`Success` / `Fail`) — ADF polls the status endpoint post-trigger and acts on failures
- Failed runs logged to a monitoring Delta table for trend analysis and SLA tracking

---

## Serving Layer — SQL Warehouse + Power BI

Gold layer Delta tables are exposed to Power BI via a **Databricks SQL Warehouse endpoint**.

SQL Warehouse is used over all-purpose clusters for this layer because:
- Power BI runs read-only analytical queries — SQL Warehouse is purpose-optimised for this workload
- Auto-suspends when idle — zero cost between BI sessions
- Faster query response for concurrent BI users
- Decouples BI consumption compute from pipeline compute entirely

---

## Key Outcomes

- Concurrent multi-source ingestion with independent failure isolation per orchestrator instance
- Full audit trail — every orchestrator instance carries a unique ID, custom status, and is trackable via ADF monitoring
- Load strategy flexibility — same Function App handles full loads, incremental loads, and inception-to-date loads via parameter flags with no separate deployments
- Zero credentials in source code — Key Vault referenced at all three layers
- Cost-optimised serving — SQL Warehouse auto-suspends between BI sessions; job clusters used for pipeline compute

---

## Tech Stack

`Azure Data Factory` `Azure Durable Functions` `Azure Databricks` `Delta Lake` `Logic Apps` `Azure Key Vault` `Azure Blob Storage` `PostgreSQL` `SQL Server` `PySpark` `Python` `Power BI` `Databricks SQL Warehouse` `Rent Manager API` `ManageAmerica API`
