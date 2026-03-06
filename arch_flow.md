# Data Product Registration — Wizard v2 Full Design

**Date:** 2026-03-06
**Status:** Approved
**Approach:** Hybrid (POST skeleton + progressive PATCH per wizard stage)

---

## Architecture Overview

```
UI (React 10-step wizard)
  |
  POST /v1/dataproducts                    --> Step 1: skeleton
  PATCH /v1/dataproducts/:id/datasets      --> Step 2: datasets + ports
  PATCH /v1/dataproducts/:id/entitlements  --> Step 3: access control
  GET   /v1/dataproducts/:id/kde           --> Step 4: read-only (future)
  PATCH /v1/dataproducts/:id/compliance    --> Step 5/6/8: DQ, contract, sensitivity
  GET   /v1/dataproducts/:id/lineage       --> Step 7: read-only visualization
  PATCH /v1/dataproducts/:id/product-info  --> Step 9: AI readiness tags
  PATCH /v1/dataproducts/:id/publish       --> Step 10: attestation + publish
  |
  v
DPAS API (FastAPI + asyncpg)
  |
  v
MetaQ PostgreSQL (12 schemas)
  |
  v
app_procs.get_dataproduct_dpf_yaml(resource_iri) --> DPF JSON output
```

**All PATCH endpoints require `If-Match` ETag header. All responses include updated `ETag`.**

---

## Step 1 — Product Identity

### UI Fields

| Field | Type | Required | Source |
|---|---|---|---|
| Owner | dropdown (pre-filled from login) | Yes | Auth context |
| Display Name | text | Yes | User input |
| Name | text (slug, beside Display Name) | Yes | Auto-gen / user input |
| Description | textarea | Yes | User input |
| Type | radio | Yes | `source_aligned` / `derived` / `consumer_aligned` |
| Source Application | search | Yes | AppDir API |
| Owning Business Division | dropdown | Yes | User input |
| Domain | dropdown | No | User input |
| Sub Domain | dropdown (filtered by Domain) | No | User input |
| Maintainers Group (Prod) | search | Yes | AD Group API |
| Allowed Platforms | checkboxes | Yes | `powerbi`, `devpod` |
| Tags | key-value pairs (+ button) | No | User input |

### API Call

```
POST /v1/dataproducts
Content-Type: application/json
Authorization: Bearer <token>
```

### Request Payload

```json
{
  "display_name": "WMA Account Master",
  "name": "wma-account-master",
  "description": "Consolidated account master for WMA",
  "product_type": "source_aligned",
  "source_application_id": "APP-12345",
  "source_application_name": "WMA Core Banking",
  "owning_business_division": "Wealth Management Americas",
  "domain": "Account",
  "sub_domain": "Client Accounts",
  "owner_email": "arpit.dave@ubs.com",
  "owner_gpn": "GPN-12345",
  "maintainers_group": {
    "PROD": ["WMA-Data-Maintainers"]
  },
  "allowed_platforms": ["powerbi", "devpod"],
  "tags": [
    { "key": "cost-center", "value": "CC-12345" },
    { "key": "priority", "value": "P1" }
  ]
}
```

### Response `201 Created`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "resource_iri": "urn:ubs:dataproduct:wma-account-master",
  "name": "wma-account-master",
  "display_name": "WMA Account Master",
  "status": "draft",
  "created_at": "2026-03-06T10:00:00Z",
  "etag": "\"v1-abc123\""
}
```

### Errors

| Code | Condition |
|---|---|
| `409` | Duplicate name |
| `422` | Missing required fields |

### DB Writes (single transaction)

| Table | Operation | Fields |
|---|---|---|
| `dcatv3.resource` | INSERT | resource_id, resource_iri, resource_type='dataproduct', title=name, description, status='draft' |
| `dprod.dataproduct` | INSERT | dataproduct_id, resource_id FK, product_type, domain, business_division, purpose |
| `persona.person` | UPSERT | Find/create by email+gpn |
| `persona.person_assignment` | INSERT | person_id, role='Data Product Owner', resource_id |
| `ontology.domain` / `subdomain` | LOOKUP | Link if they exist |
| `internal.resource` | INSERT | source_application_name, source_application_id |
| `internal.resource_allowed_platform` | INSERT batch | One row per platform |
| `internal.resource_tag` | INSERT batch | One row per tag |
| `access.principal` + `access.bbs_ad_group` | UPSERT | Maintainers group |

### DPF Fields Populated

`name`, `domain`, `description`, `dataProductType`, `sourceApplication`, `owningBusinessEntity`, `dataProductOwner`, `maintainerGroup`, `allowedOperationalPlatform`

### External API Calls (frontend, before POST)

- `GET /appdir/search?q=<name>` -- Source Application lookup
- `GET /adgroups/search?q=<name>` -- AD Group lookup

---

## Step 2 — Create & Configure Datasets

### UI Flow

1. User clicks "Add Dataset" --> Create Dataset dialog opens
2. Fills: Display Name, Name, Confidentiality Classification, CID Category, Description, Related Model
3. Clicks "Add to Data Product" --> dataset card appears
4. On dataset card: clicks "+Input" or "+Output"
5. Selects platform type (ADLS/Databricks/Postgres/SQLServer/DB2/Denodo)
6. Dynamic platform fields appear + common fields (Auth Type, Protocol, Physical Schema)
7. Saves port --> port appears on dataset card
8. Repeat for more datasets/ports
9. "Save & Continue" sends all datasets+ports to API

### API Call

```
PATCH /v1/dataproducts/:id/datasets
Content-Type: application/json
Authorization: Bearer <token>
If-Match: "v1-abc123"
```

### Request Payload

```json
{
  "datasets": [
    {
      "display_name": "Client Portfolio Holdings",
      "name": "wma_client_portfolio",
      "confidentiality_classification": "confidential",
      "cid_category": "CID",
      "description": "End-of-day portfolio positions for WMA client accounts",
      "related_model_id": null,
      "ports": [
        {
          "direction": "input",
          "platform_type": "adls",
          "auth_types": ["UAMI", "SPN"],
          "protocol": "abfss",
          "physical_schema": {
            "file_name": "portfolio_schema.json",
            "content": "<base64-encoded>"
          },
          "platform_fields": {
            "storage_account": "wmastorageprod",
            "container_name": "portfolio-data",
            "format": "parquet",
            "jurisdiction": "US"
          }
        },
        {
          "direction": "output",
          "platform_type": "databricks",
          "auth_types": ["UAMI"],
          "protocol": "JDBC",
          "physical_schema": null,
          "platform_fields": {
            "server_hostname": "adb-12345.azuredatabricks.net",
            "database_name": "wma_gold",
            "http_path": "/sql/1.0/warehouses/abc"
          }
        }
      ]
    },
    {
      "display_name": "FX Rates Daily",
      "name": "fx_rates_daily",
      "confidentiality_classification": "internal",
      "cid_category": "NON_CID",
      "description": "Daily FX rates from Reuters",
      "related_model_id": "model-uuid-if-selected",
      "ports": [
        {
          "direction": "input",
          "platform_type": "postgres",
          "auth_types": ["UAMI", "Certificates"],
          "protocol": "JDBC",
          "physical_schema": null,
          "platform_fields": {
            "server_hostname": "pgfx-prod.postgres.database.azure.com",
            "server_port": 5432,
            "database_name": "fx_rates"
          }
        }
      ]
    }
  ]
}
```

### Platform-specific `platform_fields`

**ADLS:**
```json
{ "storage_account": "string*", "container_name": "string*", "format": "csv|delta|parquet|json*", "jurisdiction": "string*" }
```

**Databricks:**
```json
{ "server_hostname": "string*", "database_name": "string*", "http_path": "string*" }
```

**Postgres:**
```json
{ "server_hostname": "string*", "server_port": "int (default 5432)", "database_name": "string*" }
```

**SQL Server:**
```json
{ "server_hostname": "string*", "server_port": "int (default 1433)", "database_name": "string*" }
```

**DB2:**
```json
{ "server_hostname": "string*", "database_name": "string*", "location": "string*" }
```

**Denodo:**
```json
{ "server_hostname": "string*", "vdp_database": "string*", "base_view_name": "string", "asis_database": "string" }
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "datasets": [
    {
      "dataset_id": "d1-uuid",
      "name": "wma_client_portfolio",
      "display_name": "Client Portfolio Holdings",
      "ports": [
        { "port_id": "p1-uuid", "direction": "input", "platform_type": "adls", "dataservice_id": "ds1-uuid" },
        { "port_id": "p2-uuid", "direction": "output", "platform_type": "databricks", "dataservice_id": "ds2-uuid" }
      ]
    },
    {
      "dataset_id": "d2-uuid",
      "name": "fx_rates_daily",
      "display_name": "FX Rates Daily",
      "ports": [
        { "port_id": "p3-uuid", "direction": "input", "platform_type": "postgres", "dataservice_id": "ds3-uuid" }
      ]
    }
  ],
  "etag": "\"v2-def456\""
}
```

### Errors

| Code | Condition |
|---|---|
| `404` | Product not found |
| `412` | ETag mismatch (concurrent edit) |
| `422` | Validation (missing required fields, invalid platform_type) |

### DB Writes (single transaction per dataset)

**Per dataset:**

| Table | Operation | Fields |
|---|---|---|
| `dcatv3.resource` | INSERT | resource_type='dataset', title=name, description |
| `dcatv3.dataset` | INSERT | dataset_id, resource_id FK, schema_id FK (if model) |
| `governance.resource_governance` | INSERT | data_classification, client_data_category |
| `dprod.dataproduct_dataset` | INSERT | dataproduct_id, dataset_id, data_flow_direction |
| `model.schema` + `model.schema_columns` | INSERT | If physical schema uploaded |

**Per port (within same transaction):**

| Table | Operation | Fields |
|---|---|---|
| `dcatv3.resource` | INSERT | resource_type='dataservice' |
| `dcatv3.dataservice` | INSERT | dataservice_id, resource_id FK, platform=enum |
| `platform.dataservice_*` | INSERT | Platform-specific table based on type |
| `dprod.dataproduct_port` | INSERT | dataproduct_id, dataservice_id, port_type |
| `dcatv3.distribution` | INSERT | distribution_id, dataset_id FK, format |
| `dcatv3.distribution_dataservice` | INSERT | Links distribution to dataservice |

### DPF Fields Populated

`inputs[]`, `outputs[]` -- including nested `distributions[]`, `services[]`, `physicalDataStructure[]`

### External API Calls (frontend)

- `GET /models/list?domain=<domain>` -- Related Model dropdown

---

## Step 3 — Entitlements

### UI Flow

1. "Do you have your own entitlement process?" -- Yes/No toggle
2. If Yes: show BBS entitlement info field + Janus BBS maintainer group
3. If No: show conditional fields (align to DPAS, eligible roles, approval rules, field-level restrictions, cross-domain sharing, Janus BBS maintainer group)

### API Call

```
PATCH /v1/dataproducts/:id/entitlements
Content-Type: application/json
Authorization: Bearer <token>
If-Match: "v2-def456"
```

### Request Payload (own process = true)

```json
{
  "own_entitlement_process": true,
  "bbs_entitlement_info": "BBS-GROUP-12345",
  "janus_bbs_maintainer_group": "WMA-Janus-Maintainers"
}
```

### Request Payload (own process = false)

```json
{
  "own_entitlement_process": false,
  "align_to_dpas_model": true,
  "align_existing_structure": false,
  "eligible_roles": ["WMA-Data-Consumer", "WMA-Data-Analyst"],
  "approval_rules": "Manager + Data Owner approval required",
  "field_level_restrictions": "SSN masked, DOB truncated to year",
  "cross_domain_sharing": false,
  "janus_bbs_maintainer_group": "WMA-Janus-Maintainers"
}
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "own_entitlement_process": true,
  "entitlements_configured": true,
  "etag": "\"v3-ghi789\""
}
```

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `access.principal` + `access.bbs_ad_group` | UPSERT | BBS/Janus groups |
| `access.entitlement` | INSERT | Per eligible role/group |
| `governance.schema_columns_governance` | UPDATE | Field-level restrictions |
| `internal.resource_tag` | INSERT | Flags: own_process, align_dpas, cross_domain, approval_rules |

### DPF Fields Populated

`has_policy[]`

---

## Step 4 — KDE Semantics (Read-only, Future)

### API Call

```
GET /v1/dataproducts/:id/kde?dataset_id=<optional-uuid>
Authorization: Bearer <token>
```

### Query Parameters

| Parameter | Type | Required | Description |
|---|---|---|---|
| `dataset_id` | UUID | No | Filter to specific dataset |

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "datasets": [
    {
      "dataset_id": "d1-uuid",
      "name": "wma_client_portfolio",
      "columns": [
        {
          "column_name": "account_id",
          "column_type": "VARCHAR(50)",
          "ontology_element": "Account Identifier",
          "business_definition": "Unique client account number",
          "confidence_score": 0.95,
          "key_data_element": true
        },
        {
          "column_name": "market_value",
          "column_type": "DECIMAL(18,2)",
          "ontology_element": "Market Value",
          "business_definition": "Current market value of the position",
          "confidence_score": 0.88,
          "key_data_element": true
        }
      ]
    }
  ]
}
```

**No write. Returns empty columns until ontology mappings exist.**

### DB Reads

`ontology.column_ontology_mapping` + `ontology.ontology_element` + `model.schema_columns`

---

## Step 5 — Scope & Data Quality

### API Call

```
PATCH /v1/dataproducts/:id/compliance
Content-Type: application/json
Authorization: Bearer <token>
If-Match: "v3-ghi789"
```

### Request Payload

```json
{
  "section": "data_quality",
  "data_scope": "Global",
  "refresh_frequency": "Daily",
  "quality_rules": "NOT NULL on account_id; market_value >= 0",
  "completeness_target": "99.5%",
  "accuracy_target": "99.9%",
  "timeliness_target": "< 15 minutes",
  "consistency_rules": "Cross-system reconciliation daily",
  "uniqueness_rules": "Unique on portfolio_id + snapshot_date",
  "validity_rules": "ISO date formats, positive quantities",
  "fitness_score": "85"
}
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "section": "data_quality",
  "saved": true,
  "etag": "\"v4-jkl012\""
}
```

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `internal.resource_tag` | UPSERT batch | `dq.data_scope`, `dq.refresh_frequency`, `dq.quality_rules`, `dq.completeness_target`, `dq.accuracy_target`, `dq.timeliness_target`, `dq.consistency_rules`, `dq.uniqueness_rules`, `dq.validity_rules`, `dq.fitness_score` |

---

## Step 6 — Data Contract

### API Call

```
PATCH /v1/dataproducts/:id/compliance
Content-Type: application/json
Authorization: Bearer <token>
If-Match: "v4-jkl012"
```

### Request Payload

```json
{
  "section": "data_contract",
  "data_retention": "7 years",
  "sla_uptime": "99.9%",
  "support_hours": "24/7",
  "change_notification_days": "30",
  "access_agreement": "Standard WMA data access terms apply. Consumer must have active entitlement.",
  "breaking_changes_policy": "Semantic versioning. 30-day deprecation notice for breaking changes.",
  "consumer_onboarding": "Self-service via DPAS portal. Requires manager approval.",
  "escalation_path": "L1: support-wma@ubs.com -> L2: IT Lead -> L3: Data Product Owner"
}
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "section": "data_contract",
  "saved": true,
  "etag": "\"v5-mno345\""
}
```

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `internal.resource_tag` | UPSERT batch | `contract.data_retention`, `contract.sla_uptime`, `contract.support_hours`, `contract.change_notification_days`, `contract.access_agreement`, `contract.breaking_changes_policy`, `contract.consumer_onboarding`, `contract.escalation_path` |

### DPF Field

`serviceLevelObjectiveDescription` on outputs assembled from `contract.sla_uptime` + `contract.support_hours`.

---

## Step 7 — Data Lineage (Read-only)

### API Call

```
GET /v1/dataproducts/:id/lineage
Authorization: Bearer <token>
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "product_name": "wma-account-master",
  "inputs": [
    {
      "dataset_id": "d1-uuid",
      "name": "wma_client_portfolio",
      "display_name": "Client Portfolio Holdings",
      "platform_type": "adls",
      "dataservice_id": "ds1-uuid"
    },
    {
      "dataset_id": "d2-uuid",
      "name": "fx_rates_daily",
      "display_name": "FX Rates Daily",
      "platform_type": "postgres",
      "dataservice_id": "ds3-uuid"
    }
  ],
  "outputs": [
    {
      "dataset_id": "d1-uuid",
      "name": "wma_client_portfolio",
      "display_name": "Client Portfolio Holdings",
      "platform_type": "databricks",
      "dataservice_id": "ds2-uuid"
    }
  ]
}
```

**No write. Derived from `dprod.dataproduct_dataset` + `dprod.dataproduct_port` created in Step 2.**

---

## Step 8 — Sensitivity & Compliance

### API Call

```
PATCH /v1/dataproducts/:id/compliance
Content-Type: application/json
Authorization: Bearer <token>
If-Match: "v5-mno345"
```

### Request Payload

```json
{
  "section": "sensitivity",
  "data_classification": "Confidential",
  "contains_pii": true,
  "regulatory_requirements": ["GDPR", "SOX", "FINMA"],
  "pii_handling_procedure": "Tokenization at ingestion. Column-level masking for non-prod environments.",
  "dpa_required": false,
  "policies": ["GDPR-Data-Retention", "UBS-CID-Policy"]
}
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "section": "sensitivity",
  "saved": true,
  "etag": "\"v6-pqr678\""
}
```

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `governance.resource_governance` | UPDATE | data_classification, governance_status |
| `internal.resource_tag` | UPSERT batch | `compliance.contains_pii`, `compliance.regulatory_requirements`, `compliance.pii_handling_procedure`, `compliance.dpa_required`, `compliance.policies` |

### DPF Fields Populated

`has_policy[]` enriched. `confidentialityClassification` on output datasets.

---

## Step 9 — AI Readiness

### API Call

```
PATCH /v1/dataproducts/:id/product-info
Content-Type: application/json
Authorization: Bearer <token>
If-Match: "v6-pqr678"
```

### Request Payload

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

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "tags_saved": 6,
  "etag": "\"v7-stu901\""
}
```

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `internal.resource_tag` | UPSERT batch | All 6 AI tags |

---

## Step 10 — Submit & Attestation

### API Call

```
PATCH /v1/dataproducts/:id/publish
Content-Type: application/json
Authorization: Bearer <token>
If-Match: "v7-stu901"
```

### Request Payload

```json
{
  "attest_accuracy": true,
  "attest_ownership": true
}
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "resource_iri": "urn:ubs:dataproduct:wma-account-master",
  "status": "published",
  "published_at": "2026-03-06T12:00:00Z",
  "dpf_yaml_data": {
    "name": "wma-account-master",
    "domain": "Account",
    "description": "Consolidated account master for WMA",
    "dataProductType": "source_aligned",
    "sourceApplication": "WMA Core Banking",
    "owningBusinessEntity": "Wealth Management Americas",
    "dataProductOwner": { "gpn": "GPN-12345", "email": "arpit.dave@ubs.com" },
    "maintainerGroup": { "PROD": ["WMA-Data-Maintainers"] },
    "allowedOperationalPlatform": ["powerbi", "devpod"],
    "contactPointEmail": ["arpit.dave@ubs.com"],
    "dpf_model_version": "1.0.0",
    "inputs": [],
    "outputs": [],
    "has_policy": []
  },
  "etag": "\"v8-vwx234\""
}
```

### Errors

| Code | Condition |
|---|---|
| `422` | Attestations false, or required steps incomplete |
| `412` | ETag mismatch |

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `dcatv3.resource` | UPDATE | status='published' |
| `dprod.dataproduct` | UPDATE | lifecycle_status='published' |
| `internal.resource_tag` | INSERT | `attestation.accuracy`, `attestation.ownership` with timestamp |

---

## Utility Endpoints

### List Products

```
GET /v1/dataproducts?page=1&pageSize=25&status=draft&owner=arpit.dave@ubs.com
```

| Query Param | Type | Default | Description |
|---|---|---|---|
| `page` | int | 1 | Page number |
| `pageSize` | int | 25 | Items per page |
| `status` | enum | (all) | `draft` / `published` / `retired` |
| `owner` | string | (all) | Filter by owner email |

### Get Single Product

```
GET /v1/dataproducts/:id
```

Returns full product detail + `ETag` header.

### Get DPF JSON

```
GET /v1/dataproducts/:id/dpf
```

Calls `app_procs.get_dataproduct_dpf_yaml(resource_iri)`. Available at any time (returns partial DPF for draft products).

### Delete Product

```
DELETE /v1/dataproducts/:id
```

Soft delete: sets status to `retired`.

---

## External API Calls (Frontend-only)

| Endpoint | Used In | Purpose |
|---|---|---|
| `GET /appdir/search?q=<name>` | Step 1 | Source Application lookup |
| `GET /adgroups/search?q=<name>` | Step 1, Step 3 | AD Group lookup |
| `GET /models/list?domain=<domain>` | Step 2 | Related Model dropdown |

---

## DPF JSON Output Structure

The stored procedure `app_procs.get_dataproduct_dpf_yaml()` assembles this from all MetaQ tables:

```json
{
  "name": "<name>",
  "domain": "<domain>",
  "description": "<description>",
  "dataProductType": "<type>",
  "sourceApplication": "<source_url>",
  "owningBusinessEntity": "<business_division>",
  "dpf_model_version": "<version>",
  "dataProductOwner": { "gpn": "<gpn>", "email": "<email>" },
  "contactPointEmail": ["<emails>"],
  "maintainerGroup": { "<env>": ["<groups>"] },
  "allowedOperationalPlatform": ["<platform_ids>"],
  "inputs": [
    {
      "name": "<dataset_name>",
      "label": "<display_name>",
      "source": "<source_url>",
      "references": [],
      "distributions": [
        {
          "name": "<dist_name>",
          "format": "<format>",
          "services": [
            {
              "location": "<location>",
              "protocol": "<protocol>",
              "container": "<container>",
              "storageAccount": "<storage_account>",
              "environmentType": "<env>",
              "cloudSubscription": "<subscription>",
              "cloudResourceGroup": "<resource_group>"
            }
          ],
          "physicalDataStructure": [
            {
              "name": "<name>",
              "path": "<path>",
              "physicalDataElement": []
            }
          ]
        }
      ]
    }
  ],
  "outputs": [
    {
      "name": "<dataset_name>",
      "label": "<display_name>",
      "source": "<source_url>",
      "distributions": [
        {
          "...same as inputs...",
          "serviceLevelObjectiveDescription": "<slo>"
        }
      ],
      "clientIdentifyingDataCategory": "<cid_category>",
      "confidentialityClassification": "<classification>"
    }
  ],
  "has_policy": [
    { "name": "<policy_name>" }
  ]
}
```

---

## Schema Gaps (require DB migration)

| DPF Field | Current Status | Recommendation |
|---|---|---|
| `cloudSubscription` | Not in `platform.dataservice_adls` | Add column or store in connection_parameters JSONB |
| `cloudResourceGroup` | Not in `platform.dataservice_adls` | Same as above |
| `serviceLevelObjectiveDescription` | Not in any table | Assemble from `internal.resource_tag` contract keys |

---

## API Surface Summary

| Endpoint | Method | Step | Tables Written |
|---|---|---|---|
| `/v1/dataproducts` | POST | 1 | dcatv3, dprod, persona, ontology, internal, access |
| `/v1/dataproducts/:id/datasets` | PATCH | 2 | dcatv3 (resource+dataset+dataservice+distribution), dprod, platform.*, model, governance |
| `/v1/dataproducts/:id/entitlements` | PATCH | 3 | access.*, governance, internal |
| `/v1/dataproducts/:id/kde` | GET | 4 | (read-only) |
| `/v1/dataproducts/:id/compliance` | PATCH | 5,6,8 | governance, internal (section discriminator) |
| `/v1/dataproducts/:id/lineage` | GET | 7 | (read-only) |
| `/v1/dataproducts/:id/product-info` | PATCH | 9 | internal.resource_tag |
| `/v1/dataproducts/:id/publish` | PATCH | 10 | dcatv3, dprod, internal |
| `/v1/dataproducts/:id/dpf` | GET | any | Calls stored proc |
| `/v1/dataproducts` | GET | -- | List/paginate |
| `/v1/dataproducts/:id` | GET | -- | Single detail |
| `/v1/dataproducts/:id` | DELETE | -- | Soft delete |
