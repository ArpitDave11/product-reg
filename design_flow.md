# 🏗️ DPF Data Flow Architecture

> **Purpose:** This document traces every DPF JSON field from the **UI Wizard** through **API calls**, into **MetaQ database tables**, and finally into the assembled **DPF output**. It covers all 10 wizard steps — from Product Identity through Submit & Attestation.

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

## 🔐 Step 3 — Entitlements

> **Endpoint:** `PATCH /dataproducts/:id/entitlements`
> **Goal:** Define access policies — either self-managed or DPAS-aligned.

### Step 3 Flow Diagram

```mermaid
flowchart TD
    START(["🟢 Step 3 — Entitlements"]) --> DECISION{"Own entitlement\nprocess?"}

    DECISION -->|"✅ Yes"| OWN["Self-managed\nEntitlements"]
    DECISION -->|"❌ No"| DPAS["DPAS-aligned\nEntitlements"]

    OWN --> OWN_PAYLOAD["📤 PATCH\n• own_entitlement_process: true\n• bbs_entitlement_info\n• janus_bbs_maintainer_group"]

    DPAS --> DPAS_PAYLOAD["📤 PATCH\n• align_to_dpas_model: true\n• eligible_roles[]\n• approval_rules\n• field_level_restrictions\n• cross_domain_sharing"]

    OWN_PAYLOAD --> TX{{"🔒 DB Transaction"}}
    DPAS_PAYLOAD --> TX

    TX --> W1["UPSERT → access.principal\n+ access.bbs_ad_group\n(BBS/Janus groups)"]
    TX --> W2["INSERT → access.entitlement\n(per eligible role/group)"]
    TX --> W3["UPDATE → governance.\nschema_columns_governance\n(field-level restrictions)"]
    TX --> W4["INSERT → internal.resource_tag\n(flags: own_process,\nalign_dpas, cross_domain,\napproval_rules)"]

    W1 --> DONE["✅ DPF: has_policy[]"]
    W2 --> DONE
    W3 --> DONE
    W4 --> DONE

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style DECISION fill:#3498DB,stroke:#2C80B4,color:#fff
    style OWN fill:#F39C12,stroke:#D68910,color:#fff
    style DPAS fill:#E67E22,stroke:#CA6F1E,color:#fff
    style TX fill:#F39C12,stroke:#D68910,color:#fff
    style DONE fill:#2ECC71,stroke:#27AE60,color:#fff
```

### Payload — Own Entitlement Process

```json
{
  "own_entitlement_process": true,
  "bbs_entitlement_info": "BBS-GROUP-12345",
  "janus_bbs_maintainer_group": "WMA-Janus-Maintainers"
}
```

### Payload — DPAS-Aligned

```json
{
  "own_entitlement_process": false,
  "align_to_dpas_model": true,
  "eligible_roles": ["WMA-Data-Consumer", "WMA-Data-Analyst"],
  "approval_rules": "Manager + Data Owner approval required",
  "field_level_restrictions": "SSN masked, DOB truncated to year",
  "cross_domain_sharing": false,
  "janus_bbs_maintainer_group": "WMA-Janus-Maintainers"
}
```

### Step 3 — DB Write Summary

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `access.principal` + `access.bbs_ad_group` | **UPSERT** | BBS/Janus groups |
| `access.entitlement` | **INSERT** | Per eligible role/group |
| `governance.schema_columns_governance` | **UPDATE** | Field-level restrictions |
| `internal.resource_tag` | **INSERT** | Flags: `own_process`, `align_dpas`, `cross_domain`, `approval_rules` |

### Step 3 → DPF Field Populated

```
✅ has_policy[]
```

---

## 📖 Step 4 — KDE Semantics *(Read-Only)*

> **Endpoint:** `GET /dataproducts/:id/kde`
> **Goal:** Display column-level business definitions from the ontology.

```mermaid
flowchart LR
    REQ["GET /dataproducts/:id/kde"] --> READ["📖 Read from:\n• ontology.column_ontology_mapping\n• ontology.ontology_element"]
    READ --> DISPLAY["🖥️ UI displays column-level\nbusiness definitions\nper dataset"]

    style REQ fill:#3498DB,stroke:#2C80B4,color:#fff
    style READ fill:#7B68EE,stroke:#5A4CB5,color:#fff
    style DISPLAY fill:#50C878,stroke:#3A9B5A,color:#fff
```

> ℹ️ **No PATCH.** No API write. Purely reads ontology mappings for display.

---

## 📊 Step 5 — Scope & Data Quality

> **Endpoint:** `PATCH /dataproducts/:id/compliance` *(section: `data_quality`)*
> **Goal:** Define data quality rules and targets.

### Step 5 Flow Diagram

```mermaid
flowchart TD
    START(["🟢 Step 5 — Data Quality"]) --> PATCH["📤 PATCH /dataproducts/:id/compliance\nsection = 'data_quality'"]

    PATCH --> TX{{"🔒 DB Transaction"}}

    TX --> T1["INSERT/UPDATE →\ninternal.resource_tag\n(batch — one tag per field)"]

    T1 --> TAGS["Tags written:\n• dq.completeness_target = 99.5%\n• dq.accuracy_target = 99.9%\n• dq.timeliness_target = < 15 min\n• dq.consistency_rules\n• dq.uniqueness_rules\n• dq.validity_rules\n• dq.fitness_score = 85"]

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style TX fill:#F39C12,stroke:#D68910,color:#fff
    style TAGS fill:#EBF5FB,stroke:#3498DB
```

### Step 5 — Request Payload

```json
{
  "section": "data_quality",
  "data_scope": "Global",
  "refresh_frequency": "Daily",
  "quality_rules": "NOT NULL on account_id; market_value >= 0",
  "completeness_target": "99.5%",
  "accuracy_target": "99.9%",
  "timeliness_target": "< 15 minutes",
  "consistency_rules": "Cross-system reconciliation",
  "uniqueness_rules": "Unique on portfolio_id",
  "validity_rules": "ISO date formats",
  "fitness_score": "85"
}
```

### Step 5 — DB Write Summary

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `internal.resource_tag` | **INSERT/UPDATE** *(batch)* | One tag per field — `key=dq.<field>`, `value=<input>` |

---

## 📜 Step 6 — Data Contract

> **Endpoint:** `PATCH /dataproducts/:id/compliance` *(section: `data_contract`)*
> **Goal:** Define SLAs, retention, and change management policies.

### Step 6 Flow Diagram

```mermaid
flowchart TD
    START(["🟢 Step 6 — Data Contract"]) --> PATCH["📤 PATCH /dataproducts/:id/compliance\nsection = 'data_contract'"]

    PATCH --> TX{{"🔒 DB Transaction"}}

    TX --> T1["INSERT/UPDATE →\ninternal.resource_tag\n(batch)"]

    T1 --> TAGS["Tags written:\n• contract.data_retention = 7 years\n• contract.sla_uptime = 99.9%\n• contract.support_hours = 24/7\n• contract.change_notification_days = 30\n• contract.breaking_changes_policy\n• contract.consumer_onboarding\n• contract.escalation_path"]

    TAGS --> NOTE["📎 Note:\nserviceLevelObjectiveDescription\nin DPF outputs is assembled\nfrom these contract.* tags"]

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style TX fill:#F39C12,stroke:#D68910,color:#fff
    style TAGS fill:#F4ECF7,stroke:#8E44AD
    style NOTE fill:#FDF2E9,stroke:#E67E22
```

### Step 6 — Request Payload

```json
{
  "section": "data_contract",
  "data_retention": "7 years",
  "sla_uptime": "99.9%",
  "support_hours": "24/7",
  "change_notification_days": "30",
  "breaking_changes_policy": "Semantic versioning with 30-day deprecation",
  "consumer_onboarding": "Self-service via portal",
  "escalation_path": "L1 → Support → IT Lead → Owner"
}
```

### Step 6 — DB Write Summary

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `internal.resource_tag` | **INSERT/UPDATE** *(batch)* | `key=contract.<field>`, `value=<input>` |

---

## 🔀 Step 7 — Data Lineage *(Read-Only)*

> **Endpoint:** `GET /dataproducts/:id/lineage`
> **Goal:** Visualize data flow — input datasets → data product → output datasets.

```mermaid
flowchart LR
    REQ["GET /dataproducts/:id/lineage"] --> READ["📖 Derived from:\n• dprod.dataproduct_dataset\n• dprod.dataproduct_port\n(created in Step 2)"]
    READ --> VIZ["🖥️ Lineage Visualization\n\nInput Datasets\n→ Data Product\n→ Output Datasets"]

    style REQ fill:#3498DB,stroke:#2C80B4,color:#fff
    style READ fill:#7B68EE,stroke:#5A4CB5,color:#fff
    style VIZ fill:#50C878,stroke:#3A9B5A,color:#fff
```

> ℹ️ **No PATCH.** Purely derived from structures created in Step 2.

---

## 🛡️ Step 8 — Sensitivity & Compliance

> **Endpoint:** `PATCH /dataproducts/:id/compliance` *(section: `sensitivity`)*
> **Goal:** Classify data sensitivity, PII handling, and regulatory requirements.

### Step 8 Flow Diagram

```mermaid
flowchart TD
    START(["🟢 Step 8 — Sensitivity"]) --> PATCH["📤 PATCH /dataproducts/:id/compliance\nsection = 'sensitivity'"]

    PATCH --> TX{{"🔒 DB Transaction"}}

    TX --> W1["UPDATE → governance.\nresource_governance\n• data_classification\n• client_data_category\n• governance_status"]
    TX --> W2["INSERT → internal.resource_tag\n• pii flags\n• regulatory requirements\n• dpa flags"]

    W1 --> DPF["✅ DPF enriched:\n• has_policy[]\n• confidentialityClassification\n(on outputs)"]
    W2 --> DPF

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style TX fill:#F39C12,stroke:#D68910,color:#fff
    style DPF fill:#2ECC71,stroke:#27AE60,color:#fff
```

### Step 8 — Request Payload

```json
{
  "section": "sensitivity",
  "data_classification": "Confidential",
  "contains_pii": true,
  "regulatory_requirements": ["GDPR", "SOX", "FINMA"],
  "pii_handling_procedure": "Tokenization at ingestion, masking for non-prod",
  "dpa_required": false,
  "policies": ["GDPR-Data-Retention", "UBS-CID-Policy"]
}
```

### Step 8 — DB Write Summary

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `governance.resource_governance` | **UPDATE** | `data_classification`, `client_data_category`, `governance_status` |
| `internal.resource_tag` | **INSERT** | PII, regulatory, DPA flags |

### Step 8 → DPF Fields Populated

```
✅ has_policy[] (enriched)
✅ outputs[].confidentialityClassification
```

---

## 🤖 Step 9 — AI Readiness

> **Endpoint:** `PATCH /dataproducts/:id/product-info`
> **Goal:** Tag the data product with AI/ML readiness metadata.

### Step 9 Flow Diagram

```mermaid
flowchart TD
    START(["🟢 Step 9 — AI Readiness"]) --> PATCH["📤 PATCH /dataproducts/:id/product-info"]

    PATCH --> TX{{"🔒 DB Transaction"}}

    TX --> T1["INSERT/UPDATE →\ninternal.resource_tag\n(batch)"]

    T1 --> TAGS

    subgraph TAGS ["AI Tags Written"]
        direction TB
        TAG1["ai.ml_feature_store = Enabled"]
        TAG2["ai.model_usage = Training"]
        TAG3["ai.embedding_support = Supported"]
        TAG4["ai.bias_assessment = Completed"]
        TAG5["ai.explainability = Required"]
        TAG6["ai.model_governance = Centralized"]
    end

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style TX fill:#F39C12,stroke:#D68910,color:#fff
    style TAGS fill:#EBF5FB,stroke:#3498DB
```

### Step 9 — Request Payload

```json
{
  "tags": [
    { "key": "ai.ml_feature_store", "value": "Enabled" },
    { "key": "ai.model_usage", "value": "Training" },
    { "key": "ai.embedding_support", "value": "Supported" },
    { "key": "ai.bias_assessment", "value": "Completed" },
    { "key": "ai.explainability", "value": "Required" },
    { "key": "ai.model_governance", "value": "Centralized" }
  ]
}
```

### Step 9 — DB Write Summary

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `internal.resource_tag` | **INSERT/UPDATE** *(batch)* | All stored as `ai.*` tags |

---

## 🚀 Step 10 — Submit & Attestation

> **Endpoint:** `PATCH /dataproducts/:id/publish`
> **Goal:** Owner attests accuracy & ownership, then publishes the data product.

### Step 10 Flow Diagram

```mermaid
flowchart TD
    START(["🟢 Step 10 — Publish"]) --> PATCH["📤 PATCH /dataproducts/:id/publish\n• attest_accuracy: true\n• attest_ownership: true"]

    PATCH --> TX{{"🔒 DB Transaction"}}

    TX --> W1["UPDATE → dcatv3.resource\n• status = 'published'"]
    TX --> W2["UPDATE → dprod.dataproduct\n• lifecycle_status = 'published'"]
    TX --> W3["INSERT → internal.resource_tag\n• attestation records\n• timestamp"]

    W1 --> RESP
    W2 --> RESP
    W3 --> RESP

    RESP["✅ Response includes:\ndpf_yaml_data — full DPF JSON\nassembled by\napp_procs.get_dataproduct_dpf_yaml()"]

    RESP --> FINAL["📄 Complete DPF JSON\nready for consumption"]

    style START fill:#27AE60,stroke:#1E8449,color:#fff
    style TX fill:#F39C12,stroke:#D68910,color:#fff
    style RESP fill:#2ECC71,stroke:#27AE60,color:#fff
    style FINAL fill:#E74C3C,stroke:#B83A2E,color:#fff
```

### Step 10 — Request Payload

```json
{
  "attest_accuracy": true,
  "attest_ownership": true
}
```

### Step 10 — DB Write Summary

| Table | Operation | Key Fields |
|:------|:---------:|:-----------|
| `dcatv3.resource` | **UPDATE** | `status='published'` |
| `dprod.dataproduct` | **UPDATE** | `lifecycle_status='published'` |
| `internal.resource_tag` | **INSERT** | Attestation records with timestamp |

---

## 🔁 End-to-End Recap — All 10 Steps

```mermaid
flowchart TD
    subgraph STEP1 ["📝 Step 1 — Product Identity"]
        S1["POST /dataproducts\n→ 9 table writes\n→ DPF: name, domain,\nowner, platforms"]
    end

    subgraph STEP2 ["📦 Step 2 — Datasets & Ports"]
        S2["PATCH .../datasets\n→ 5 writes/dataset\n+ 6 writes/port\n→ DPF: inputs[], outputs[]"]
    end

    subgraph STEP3 ["🔐 Step 3 — Entitlements"]
        S3["PATCH .../entitlements\n→ access.*, governance\n→ DPF: has_policy[]"]
    end

    subgraph STEP4 ["📖 Step 4 — KDE Semantics"]
        S4["GET .../kde\n(read-only)\nOntology mappings"]
    end

    subgraph STEP5 ["📊 Step 5 — Data Quality"]
        S5["PATCH .../compliance\nsection=data_quality\n→ dq.* tags"]
    end

    subgraph STEP6 ["📜 Step 6 — Data Contract"]
        S6["PATCH .../compliance\nsection=data_contract\n→ contract.* tags"]
    end

    subgraph STEP7 ["🔀 Step 7 — Lineage"]
        S7["GET .../lineage\n(read-only)\nDerived from Step 2"]
    end

    subgraph STEP8 ["🛡️ Step 8 — Sensitivity"]
        S8["PATCH .../compliance\nsection=sensitivity\n→ governance + tags\n→ DPF: has_policy[]"]
    end

    subgraph STEP9 ["🤖 Step 9 — AI Readiness"]
        S9["PATCH .../product-info\n→ ai.* tags"]
    end

    subgraph STEP10 ["🚀 Step 10 — Publish"]
        S10["PATCH .../publish\n→ status = published\n→ Returns full DPF"]
    end

    subgraph ASSEMBLY ["🔧 DPF Assembly"]
        ASM["GET .../dpf\n→ reader_dataproduct view\n→ app_procs.get_dataproduct_dpf_yaml()\n→ 📄 Complete DPF JSON"]
    end

    STEP1 --> STEP2 --> STEP3 --> STEP4
    STEP4 --> STEP5 --> STEP6 --> STEP7
    STEP7 --> STEP8 --> STEP9 --> STEP10
    STEP10 --> ASSEMBLY

    style STEP1 fill:#EBF5FB,stroke:#3498DB
    style STEP2 fill:#F4ECF7,stroke:#8E44AD
    style STEP3 fill:#FEF9E7,stroke:#F1C40F
    style STEP4 fill:#EAFAF1,stroke:#2ECC71
    style STEP5 fill:#EBF5FB,stroke:#3498DB
    style STEP6 fill:#F4ECF7,stroke:#8E44AD
    style STEP7 fill:#EAFAF1,stroke:#2ECC71
    style STEP8 fill:#FDEDEC,stroke:#E74C3C
    style STEP9 fill:#EBF5FB,stroke:#3498DB
    style STEP10 fill:#D5F5E3,stroke:#27AE60
    style ASSEMBLY fill:#D5F5E3,stroke:#27AE60
```

---

## 🗺️ Complete API Surface Summary

```mermaid
flowchart LR
    subgraph WRITE ["✏️ Write Endpoints"]
        direction TB
        E1["POST /dataproducts\n— Step 1"]
        E2["PATCH .../datasets\n— Step 2"]
        E3["PATCH .../entitlements\n— Step 3"]
        E5["PATCH .../compliance\n— Steps 5, 6, 8"]
        E9["PATCH .../product-info\n— Step 9"]
        E10["PATCH .../publish\n— Step 10"]
    end

    subgraph READ ["👁️ Read-Only"]
        direction TB
        R4["GET .../kde\n— Step 4"]
        R7["GET .../lineage\n— Step 7"]
        RDPF["GET .../dpf\n— DPF Assembly"]
    end

    style WRITE fill:#FEF9E7,stroke:#F1C40F
    style READ fill:#EAFAF1,stroke:#2ECC71
```

| Endpoint | Method | Step | Writes To |
|:---------|:------:|:----:|:----------|
| `/dataproducts` | **POST** | 1 | `dcatv3`, `dprod`, `persona`, `ontology`, `internal`, `access` |
| `/dataproducts/:id/datasets` | **PATCH** | 2 | `dcatv3` (resource + dataset + dataservice + distribution), `dprod`, `platform.*`, `model`, `governance` |
| `/dataproducts/:id/entitlements` | **PATCH** | 3 | `access.*`, `governance`, `internal` |
| `/dataproducts/:id/kde` | **GET** | 4 | *(read-only)* |
| `/dataproducts/:id/compliance` | **PATCH** | 5, 6, 8 | `governance`, `internal` *(uses `section` discriminator)* |
| `/dataproducts/:id/lineage` | **GET** | 7 | *(read-only)* |
| `/dataproducts/:id/product-info` | **PATCH** | 9 | `internal.resource_tag` |
| `/dataproducts/:id/publish` | **PATCH** | 10 | `dcatv3`, `dprod`, `internal` |
| `/dataproducts/:id/dpf` | **GET** | — | Calls stored proc → returns DPF JSON |

---

*Document generated for internal architecture reference.*
