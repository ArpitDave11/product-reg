# 🏗️ DPF Data Flow Architecture

> **Purpose:** This document traces every DPF JSON field from the **UI Wizard** through **API calls**, into **MetaQ database tables**, and finally into the assembled **DPF output**. It covers Step 1 (Product Identity) and Step 2 (Dataset Configuration).

---

## 📌 High-Level Overview

The DPF JSON is the **target output**. Everything the wizard collects flows through MetaQ tables and gets assembled by `app_procs.get_dataproduct_dpf_yaml()` into the final format.

```mermaid
flowchart TD
    A["🖥️ UI Wizard\n(User fills fields)"] -->|POST / PATCH| B["⚙️ REST API\n(/dataproducts)"]
    B -->|Stored Procs| C["🗄️ MetaQ Tables\n(Normalized Storage)"]
    C -->|GET .../dpf| D["📖 reader_dataproduct View"]
    D --> E["🔧 app_procs.\nget_dataproduct_dpf_yaml\n(resource_iri)"]
    E -->|Reads all linked tables| F["📄 Assembled DPF JSON"]

    style A fill:#4A90D9,stroke:#2C5F8A,color:#fff
    style B fill:#F5A623,stroke:#C47D1A,color:#fff
    style C fill:#7B68EE,stroke:#5A4CB5,color:#fff
    style D fill:#50C878,stroke:#3A9B5A,color:#fff
    style E fill:#50C878,stroke:#3A9B5A,color:#fff
    style F fill:#E74C3C,stroke:#B83A2E,color:#fff
```

---

## 🔗 Complete Field Traceability

### DPF Field → MetaQ Table → API Stage

```mermaid
flowchart LR
    subgraph UI ["🖥️ UI Wizard"]
        direction TB
        U1["Display Name"]
        U2["Domain"]
        U3["Description"]
        U4["Product Type"]
        U5["Owner Email"]
    end

    subgraph API ["⚙️ API Layer"]
        direction TB
        A1["POST\n/dataproducts"]
    end

    subgraph DB ["🗄️ MetaQ Tables"]
        direction TB
        D1["dcatv3.resource.title"]
        D2["dprod.dataproduct.domain"]
        D3["dcatv3.resource.description"]
        D4["dprod.dataproduct.product_type"]
        D5["persona.person.email"]
    end

    subgraph DPF ["📄 DPF JSON"]
        direction TB
        J1["name"]
        J2["domain"]
        J3["description"]
        J4["dataProductType"]
        J5["dataProductOwner.email"]
    end

    U1 --> A1 --> D1 --> J1
    U2 --> A1 --> D2 --> J2
    U3 --> A1 --> D3 --> J3
    U4 --> A1 --> D4 --> J4
    U5 --> A1 --> D5 --> J5

    style UI fill:#E8F4FD,stroke:#4A90D9
    style API fill:#FFF3E0,stroke:#F5A623
    style DB fill:#F0E6FF,stroke:#7B68EE
    style DPF fill:#FDEDEC,stroke:#E74C3C
```

---

## 📝 Step 1 — Product Identity

> **Endpoint:** `POST /dataproducts`
> **Goal:** Register a new Data Product with its core metadata.

### Step 1 Flow Diagram

```mermaid
flowchart TD
    START(["🟢 User opens Wizard — Step 1"]) --> EXT

    subgraph EXT ["🌐 External Lookups (Frontend)"]
        direction LR
        EXT1["AppDir API\n→ Resolve source app name\n→ Returns app ID"]
        EXT2["AD Group API\n→ Resolve maintainers group\n→ Returns object_id"]
    end

    EXT --> POST["📤 POST /dataproducts\n(Full JSON payload)"]

    POST --> TX{{"🔒 Single DB Transaction"}}

    TX --> W1["INSERT → dcatv3.resource\n• resource_iri\n• title = name\n• description\n• status = 'draft'"]
    TX --> W2["INSERT → dprod.dataproduct\n• product_type\n• domain\n• business_division"]
    TX --> W3["UPSERT → persona.person\n• Find/create by email"]
    TX --> W4["INSERT → persona.person_assignment\n• role = 'Data Product Owner'"]
    TX --> W5["LOOKUP → ontology.domain\n/ ontology.subdomain"]
    TX --> W6["INSERT → internal.resource\n• source_application"]
    TX --> W7["INSERT (batch) →\ninternal.resource_allowed_platform\n• One row per platform"]
    TX --> W8["INSERT (batch) →\ninternal.resource_tag\n• One row per tag"]
    TX --> W9["UPSERT → access.principal\n+ access.bbs_ad_group\n• Maintainers group"]

    W1 --> RESP
    W2 --> RESP
    W3 --> RESP
    W4 --> RESP
    W5 --> RESP
    W6 --> RESP
    W7 --> RESP
    W8 --> RESP
    W9 --> RESP

    RESP["✅ Response\n• resource_id (uuid)\n• resource_iri\n• status: draft\n• etag"]

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style TX fill:#F39C12,stroke:#D68910,color:#fff
    style RESP fill:#2ECC71,stroke:#27AE60,color:#fff
    style EXT fill:#EBF5FB,stroke:#3498DB
```

### Step 1 — Request Payload

```json
{
  "display_name": "WMA Account Master",
  "name": "wma-account-master",
  "description": "Consolidated account master for WMA",
  "product_type": "source_aligned",
  "source_application": "APP-12345",
  "owning_business_division": "Wealth Management Americas",
  "domain": "Account",
  "sub_domain": "Client Accounts",
  "maintainers_group": { "PROD": ["WMA-Data-Maintainers"] },
  "allowed_platforms": ["powerbi", "devpod"],
  "tags": [
    { "key": "cost-center", "value": "CC-12345" },
    { "key": "priority", "value": "P1" }
  ],
  "owner_email": "arpit.dave@ubs.com"
}
```

### Step 1 — Response

```json
{
  "resource_id": "uuid",
  "resource_iri": "urn:ubs:dataproduct:wma-account-master",
  "status": "draft",
  "etag": "\"v1-abc\""
}
```

### Step 1 — DB Write Summary

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `dcatv3.resource` | **INSERT** | `resource_id`, `resource_iri`, `resource_type='dataproduct'`, `title`, `description`, `status='draft'` |
| `dprod.dataproduct` | **INSERT** | `dataproduct_id`, `resource_id` FK, `product_type`, `domain`, `business_division`, `purpose` |
| `persona.person` | **UPSERT** | Find/create person by email |
| `persona.person_assignment` | **INSERT** | `person_id`, `role='Data Product Owner'`, `resource_id` |
| `ontology.domain` / `subdomain` | **LOOKUP** | Link domain/subdomain if they exist |
| `internal.resource` | **INSERT** | `source_application_name/id` (from AppDir) |
| `internal.resource_allowed_platform` | **INSERT** *(batch)* | One row per platform |
| `internal.resource_tag` | **INSERT** *(batch)* | One row per tag key-value |
| `access.principal` + `access.bbs_ad_group` | **UPSERT** | Maintainers group from AD lookup |

### Step 1 → DPF Fields Populated

```
✅ name                        ✅ domain
✅ description                  ✅ dataProductType
✅ sourceApplication            ✅ owningBusinessEntity
✅ dataProductOwner             ✅ maintainerGroup
✅ allowedOperationalPlatform
```

---

## 📦 Step 2 — Create & Configure Datasets

> **Endpoint:** `PATCH /dataproducts/:id/datasets`
> **Goal:** Attach one or more datasets with their ports, distributions, and schemas.
> ⚠️ *This is the most complex step — multiple sub-entities are created per dataset.*

### Step 2 Master Flow

```mermaid
flowchart TD
    START(["🟢 User enters Step 2"]) --> MODEL["🌐 External: Models API\n→ Populate 'Related Model' dropdown"]
    MODEL --> PATCH["📤 PATCH /dataproducts/:id/datasets\n(Array of datasets)"]

    PATCH --> LOOP{{"🔄 For each Dataset"}}

    LOOP --> DS_TX{{"🔒 Dataset Transaction"}}

    DS_TX --> DS1["INSERT → dcatv3.resource\n• resource_type = 'dataset'\n• title = name"]
    DS_TX --> DS2["INSERT → dcatv3.dataset\n• dataset_id\n• schema_id FK (if model)"]
    DS_TX --> DS3["INSERT → governance.resource_governance\n• data_classification\n• client_data_category"]
    DS_TX --> DS4["INSERT → dprod.dataproduct_dataset\n• data_flow_direction"]
    DS_TX --> DS5["INSERT → model.schema\n+ model.schema_columns\n(if schema uploaded)"]

    DS1 --> PORT_LOOP
    DS2 --> PORT_LOOP
    DS3 --> PORT_LOOP
    DS4 --> PORT_LOOP
    DS5 --> PORT_LOOP

    PORT_LOOP{{"🔄 For each Port"}} --> PORT_TX{{"🔒 Port Transaction"}}

    PORT_TX --> P1["INSERT → dcatv3.resource\n• resource_type = 'dataservice'"]
    PORT_TX --> P2["INSERT → dcatv3.dataservice\n• platform = enum"]
    PORT_TX --> P3["INSERT → platform.dataservice_*\n• Platform-specific table"]
    PORT_TX --> P4["INSERT → dprod.dataproduct_port\n• port_type (input/output)"]
    PORT_TX --> P5["INSERT → dcatv3.distribution\n• format"]
    PORT_TX --> P6["INSERT → dcatv3.distribution_dataservice\n• Links distribution ↔ dataservice"]

    P1 --> DONE
    P2 --> DONE
    P3 --> DONE
    P4 --> DONE
    P5 --> DONE
    P6 --> DONE

    DONE["✅ Dataset + Ports Saved"]
    DONE --> LOOP

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style DS_TX fill:#F39C12,stroke:#D68910,color:#fff
    style PORT_TX fill:#E67E22,stroke:#CA6F1E,color:#fff
    style LOOP fill:#3498DB,stroke:#2C80B4,color:#fff
    style PORT_LOOP fill:#3498DB,stroke:#2C80B4,color:#fff
    style DONE fill:#2ECC71,stroke:#27AE60,color:#fff
```

### Step 2 — Request Payload (per dataset)

```json
{
  "datasets": [
    {
      "display_name": "Client Portfolio Holdings",
      "name": "wma_client_portfolio",
      "confidentiality_classification": "confidential",
      "cid_category": "CID",
      "description": "End-of-day portfolio positions",
      "related_model_id": null,
      "ports": [
        {
          "direction": "input",
          "platform_type": "adls",
          "auth_types": ["UAMI", "SPN"],
          "physical_schema_file": "<base64 or reference>",
          "platform_fields": {
            "storage_account": "wmastorageprod",
            "container_name": "portfolio-data",
            "protocol": "abfss",
            "format": "parquet",
            "jurisdiction": "US"
          }
        },
        {
          "direction": "output",
          "platform_type": "databricks",
          "auth_types": ["UAMI"],
          "physical_schema_file": null,
          "platform_fields": {
            "server_hostname": "adb-12345.azuredatabricks.net",
            "database_name": "wma_gold",
            "http_path": "/sql/1.0/warehouses/abc"
          }
        }
      ]
    }
  ]
}
```

### Step 2 — DB Writes per Dataset

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `dcatv3.resource` | **INSERT** | `resource_type='dataset'`, `title=name` |
| `dcatv3.dataset` | **INSERT** | `dataset_id`, `resource_id` FK, `schema_id` FK |
| `governance.resource_governance` | **INSERT** | `data_classification`, `client_data_category` |
| `dprod.dataproduct_dataset` | **INSERT** | `dataproduct_id`, `dataset_id`, `data_flow_direction` |
| `model.schema` + `model.schema_columns` | **INSERT** | If physical schema uploaded |

### Step 2 — DB Writes per Port

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `dcatv3.resource` | **INSERT** | `resource_type='dataservice'` |
| `dcatv3.dataservice` | **INSERT** | `dataservice_id`, `resource_id` FK, `platform=enum` |
| `platform.dataservice_*` | **INSERT** | Platform-specific table (see below) |
| `dprod.dataproduct_port` | **INSERT** | `dataproduct_id`, `dataservice_id`, `port_type` |
| `dcatv3.distribution` | **INSERT** | `distribution_id`, `dataset_id` FK, `format` |
| `dcatv3.distribution_dataservice` | **INSERT** | Links distribution ↔ dataservice |

### Platform-Specific Table Routing

```mermaid
flowchart LR
    PORT["🔌 Port\nplatform_type"] --> ADLS
    PORT --> DBX
    PORT --> PG
    PORT --> SQL
    PORT --> DB2
    PORT --> DEN

    ADLS["📂 ADLS\nplatform.dataservice_adls\n─────────────\n• storage_account\n• container_name\n• protocol\n• format / jurisdiction"]
    DBX["⚡ Databricks\nplatform.dataservice_databricks\n─────────────\n• server_hostname\n• database_name\n• http_path"]
    PG["🐘 Postgres\nplatform.dataservice_postgres\n─────────────\n• server_hostname\n• database_name\n• protocol"]
    SQL["🔷 SQL Server\nplatform.dataservice_sqlserver\n─────────────\n• server_hostname\n• database_name\n• protocol"]
    DB2["🔶 DB2\nplatform.dataservice_db2\n─────────────\n• server_hostname\n• database_name\n• location / protocol"]
    DEN["🟣 Denodo\nplatform.dataservice_denodo\n─────────────\n• server_hostname\n• vdp_database\n• base_view_name\n• protocol"]

    style PORT fill:#3498DB,stroke:#2C80B4,color:#fff
    style ADLS fill:#E8F8F5,stroke:#1ABC9C
    style DBX fill:#FEF9E7,stroke:#F1C40F
    style PG fill:#EBF5FB,stroke:#3498DB
    style SQL fill:#EBF5FB,stroke:#2980B9
    style DB2 fill:#FDF2E9,stroke:#E67E22
    style DEN fill:#F4ECF7,stroke:#8E44AD
```

### Step 2 → DPF Fields Populated

```
✅ inputs[]                    ✅ outputs[]
✅ inputs[].name               ✅ outputs[].confidentialityClassification
✅ inputs[].label              ✅ outputs[].clientIdentifyingDataCategory
✅ inputs[].source             ✅ distributions[]
✅ distributions[].name        ✅ distributions[].format
✅ distributions[].services[]  ✅ distributions[].physicalDataStructure[]
```

---

## ⚠️ DPF → MetaQ Field Mapping (Inputs & Outputs)

### Dataset-Level Fields

| DPF Field | MetaQ Table.Column |
|:----------|:-------------------|
| `inputs[].name` | `dcatv3.resource.title` *(prefixed by platform)* |
| `inputs[].label` | `dcatv3.resource.title` *(display name)* |
| `inputs[].source` | `internal.resource.source_application_url` |
| *direction* (input vs output) | `dprod.dataproduct_dataset.data_flow_direction` |

### Distribution-Level Fields

| DPF Field | MetaQ Table.Column |
|:----------|:-------------------|
| `distributions[].name` | `dcatv3.distribution` + `dcatv3.resource.title` |
| `distributions[].format` | `dcatv3.distribution.format` |
| `distributions[].services[].storageAccount` | `platform.dataservice_adls.storage_account` |
| `distributions[].services[].container` | `platform.dataservice_adls.container_name` |
| `distributions[].services[].protocol` | `platform.dataservice_adls.protocol` |
| `distributions[].services[].location` | `platform.dataservice_adls` *(derived)* |
| `distributions[].services[].environmentType` | `operational.distribution_environment.environment_type` |
| `distributions[].physicalDataStructure[].name` | `model.schema.schema_name` |
| `distributions[].physicalDataStructure[].path` | `operational.distribution.file_path` |
| `distributions[].physicalDataStructure[].physicalDataElement[]` | `model.schema_columns` |

### Output-Only Fields

| DPF Field | MetaQ Table.Column |
|:----------|:-------------------|
| `outputs[].confidentialityClassification` | `governance.resource_governance.data_classification` |
| `outputs[].clientIdentifyingDataCategory` | `governance.resource_governance.client_data_category` |
| `outputs[].distributions[].serviceLevelObjectiveDescription` | Step 5/6 SLO → `internal.resource_tag` or new column |

---

## 🚨 Schema Gaps

> These DPF fields currently have **no MetaQ home** and need resolution.

```mermaid
flowchart LR
    GAP1["cloudSubscription"] -->|"❌ No column"| REC1["➕ Add to\nplatform.dataservice_adls\nor JSONB\nconnection_parameters"]
    GAP2["cloudResourceGroup"] -->|"❌ No column"| REC1
    GAP3["serviceLevelObjective\nDescription"] -->|"❌ No column"| REC2["➕ Add to\noperational.distribution\nor internal.resource_tag"]

    style GAP1 fill:#FADBD8,stroke:#E74C3C
    style GAP2 fill:#FADBD8,stroke:#E74C3C
    style GAP3 fill:#FADBD8,stroke:#E74C3C
    style REC1 fill:#D5F5E3,stroke:#27AE60
    style REC2 fill:#D5F5E3,stroke:#27AE60
```

| DPF Field | Current Status | Recommendation |
|:----------|:--------------:|:---------------|
| `cloudSubscription` | ❌ Missing | Add to `platform.dataservice_adls` or store in JSONB `connection_parameters` |
| `cloudResourceGroup` | ❌ Missing | Same as above |
| `serviceLevelObjectiveDescription` | ❌ Missing | Add to `operational.distribution` or `internal.resource_tag` |

---

## 🔐 Auth Types Handling

Auth types (`UAMI`, `EVA`, `SPN`, `Certificates`) are stored as a **JSONB array** on the dataservice record, or as `internal.resource_tag` entries per dataservice resource.

---

## 🔁 End-to-End Recap

```mermaid
flowchart TD
    subgraph STEP1 ["📝 Step 1 — Product Identity"]
        S1A["UI: Name, Domain,\nOwner, Platforms, Tags"]
        S1B["POST /dataproducts"]
        S1C["9 MetaQ table writes"]
        S1D["DPF: name, domain,\ndescription, owner,\nmaintainer, platforms"]
        S1A --> S1B --> S1C --> S1D
    end

    subgraph STEP2 ["📦 Step 2 — Datasets & Ports"]
        S2A["UI: Dataset config,\nPorts, Schemas"]
        S2B["PATCH /dataproducts/:id\n/datasets"]
        S2C["5 writes/dataset\n+ 6 writes/port\n+ platform-specific"]
        S2D["DPF: inputs[], outputs[],\ndistributions[],\nservices[], schemas"]
        S2A --> S2B --> S2C --> S2D
    end

    subgraph ASSEMBLY ["🔧 DPF Assembly"]
        ASM1["GET /dataproducts/:id/dpf"]
        ASM2["reader_dataproduct view"]
        ASM3["app_procs.\nget_dataproduct_dpf_yaml()"]
        ASM4["📄 Complete DPF JSON"]
        ASM1 --> ASM2 --> ASM3 --> ASM4
    end

    STEP1 --> STEP2 --> ASSEMBLY

    style STEP1 fill:#EBF5FB,stroke:#3498DB
    style STEP2 fill:#F4ECF7,stroke:#8E44AD
    style ASSEMBLY fill:#D5F5E3,stroke:#27AE60
```

---

*Document generated for internal architecture reference.*
