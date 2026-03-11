# Data Product Registration — Wizard v2 Design (Final)

**Date:** 2026-03-11
**Status:** Confirmed
**Approach:** Single PATCH — leveraging existing DPAS OpenAPI spec

---

## Architecture Overview

```
UI (React 10-step wizard)
  |
  POST /v1/dataproducts                   --> Step 1: create skeleton
  PATCH /v1/dataproducts/{id}             --> Steps 2-10: section-based updates
  GET   /v1/dataproducts/{id}/kde         --> Step 4: read-only (future)
  GET   /v1/dataproducts/{id}/lineage     --> Step 7: read-only visualization
  GET   /v1/dataproducts/{id}/dpf         --> DPF JSON output (any time)
  |
  v
DPAS API (FastAPI + asyncpg, existing OpenAPI spec)
  |
  v
MetaQ PostgreSQL (12 schemas)
  |
  v
app_procs.get_dataproduct_dpf_yaml(resource_iri) --> DPF JSON output
```

**Single PATCH endpoint** — `PATCH /v1/dataproducts/{id}` handles all wizard step updates.
The backend routes internally based on the `section` field in the payload.
The OpenAPI spec uses `additionalProperties: true` making this extensible without new endpoints.

---

## Existing Endpoints (from DPAS OpenAPI spec)

| # | Method | Endpoint | Status |
|---|--------|----------|--------|
| 1 | `GET` | `/health` | Existing |
| 2 | `GET` | `/v1/dataproducts` | Existing — list with page, pageSize, lifecycle_status |
| 3 | `POST` | `/v1/dataproducts` | Existing — title req, desc/product_type/purpose/domain/business_division/ports[]/datasets[] opt |
| 4 | `GET` | `/v1/dataproducts/{id}` | Existing |
| 5 | `PATCH` | `/v1/dataproducts/{id}` | Existing — `additionalProperties: true` |
| 6 | `DELETE` | `/v1/dataproducts/{id}` | Existing — soft delete (lifecycle_status='retire') |
| 7 | `GET` | `/v1/datasets` | Existing — title filter, limit, offset |
| 8 | `GET` | `/v1/dataset/{title}` | Existing |
| 9 | `POST` | `/v1/dataset` | Existing — p_dataset_data + p_internal |
| 10 | `PATCH` | `/v1/dataset/{title}` | Existing |

## New Endpoints (to be built)

| # | Method | Endpoint | Purpose |
|---|--------|----------|---------|
| 11 | `GET` | `/v1/dataproducts/{id}/kde` | Step 4 — read-only KDE semantics |
| 12 | `GET` | `/v1/dataproducts/{id}/lineage` | Step 7 — read-only lineage visualization |
| 13 | `GET` | `/v1/dataproducts/{id}/dpf` | DPF JSON output (calls stored proc) |

## External APIs (frontend-only)

| # | Endpoint | Used In |
|---|----------|---------|
| 14 | `GET /appdir/search?q=<name>` | Step 1 — Source Application lookup |
| 15 | `GET /adgroups/search?q=<name>` | Step 1, Step 3 — AD Group lookup |
| 16 | `GET /models/list?domain=<domain>` | Step 2 — Related Model dropdown |

**Total: 10 existing + 3 new + 3 external = 16 endpoints**

---

## Existing Response Schemas (from OpenAPI spec)

### DataProduct

```json
{
  "resource_iri": "urn:ubs:dataproduct:account-master",
  "resource_id": "123e4567-e89b-12d3-a456-426614174000",
  "dataproduct_id": "223e4567-e89b-12d3-a456-426614174001",
  "title": "Account Master Product",
  "description": "Central master data for all customer accounts",
  "lifecycle_status": "build",
  "product_type": "source_aligned",
  "purpose": "Provide authoritative account reference data",
  "domain": "Customer & Accounts",
  "business_division": "Wealth Management"
}
```

`additionalProperties: true` — backend can return extended fields.

### DataProductList

```json
{ "items": [{}], "total": 50, "page": 1, "pageSize": 25 }
```

### DatasetRequest

```json
{
  "p_dataset_data": { "resource_iri": "string", "title": "string", "description": "string", "status": "string" },
  "p_internal": { "source_system_code": "string", "owner_team": "string" }
}
```

### ErrorResponse

```json
{ "error": "ValidationError", "message": "The field 'title' is required", "timestamp": "2026-03-11T14:45:00Z" }
```

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
  "title": "WMA Account Master",
  "description": "Consolidated account master for WMA",
  "product_type": "source_aligned",
  "purpose": "Provide authoritative account master data",
  "domain": "Account",
  "business_division": "Wealth Management Americas",
  "source_application_id": "APP-12345",
  "source_application_name": "WMA Core Banking",
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

Note: `title` is the only required field per OpenAPI spec. The API auto-generates `resource_iri` from title and defaults `lifecycle_status` to `build`.

### Response `201 Created`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "resource_iri": "urn:ubs:dataproduct:wma-account-master",
  "dataproduct_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "title": "WMA Account Master",
  "description": "Consolidated account master for WMA",
  "lifecycle_status": "build",
  "product_type": "source_aligned",
  "purpose": "Provide authoritative account master data",
  "domain": "Account",
  "business_division": "Wealth Management Americas",
  "created_at": "2026-03-11T10:00:00Z"
}
```

### Errors

| Code | Condition |
|---|---|
| `409` | Duplicate title (title already exists) |
| `422` | Missing required fields |

### DB Writes (single transaction)

| Table | Operation | Fields |
|---|---|---|
| `dcatv3.resource` | INSERT | resource_id, resource_iri, resource_type='dataproduct', title, description, status='draft' |
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

- `GET /appdir/search?q=<name>` — Source Application lookup
- `GET /adgroups/search?q=<name>` — AD Group lookup

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
9. "Save & Continue" sends all datasets+ports via PATCH

### API Call

```
PATCH /v1/dataproducts/{id}
Content-Type: application/json
Authorization: Bearer <token>
```

### Request Payload

```json
{
  "section": "datasets",
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
  ]
}
```

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

`inputs[]`, `outputs[]` — including nested `distributions[]`, `services[]`, `physicalDataStructure[]`

### External API Call (frontend)

- `GET /models/list?domain=<domain>` — Related Model dropdown

---

## Step 3 — Entitlements

### UI Flow

1. "Do you have your own entitlement process?" — Yes/No toggle
2. If Yes: show BBS entitlement info field + Janus BBS maintainer group
3. If No: show conditional fields (align to DPAS, eligible roles, approval rules, field-level restrictions, cross-domain sharing, Janus BBS maintainer group)

### API Call

```
PATCH /v1/dataproducts/{id}
Content-Type: application/json
Authorization: Bearer <token>
```

### Request Payload (own process = true)

```json
{
  "section": "entitlements",
  "own_entitlement_process": true,
  "bbs_entitlement_info": "BBS-GROUP-12345",
  "janus_bbs_maintainer_group": "WMA-Janus-Maintainers"
}
```

### Request Payload (own process = false)

```json
{
  "section": "entitlements",
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
  "entitlements_configured": true
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
GET /v1/dataproducts/{id}/kde?dataset_id=<optional-uuid>
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
PATCH /v1/dataproducts/{id}
Content-Type: application/json
Authorization: Bearer <token>
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
  "saved": true
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
PATCH /v1/dataproducts/{id}
Content-Type: application/json
Authorization: Bearer <token>
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
  "saved": true
}
```

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `internal.resource_tag` | UPSERT batch | `contract.data_retention`, `contract.sla_uptime`, `contract.support_hours`, `contract.change_notification_days`, `contract.access_agreement`, `contract.breaking_changes_policy`, `contract.consumer_onboarding`, `contract.escalation_path` |

### DPF Field

`serviceLevelObjectiveDescription` on outputs — assembled from `contract.sla_uptime` + `contract.support_hours`.

---

## Step 7 — Data Lineage (Read-only)

### API Call

```
GET /v1/dataproducts/{id}/lineage
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
PATCH /v1/dataproducts/{id}
Content-Type: application/json
Authorization: Bearer <token>
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
  "saved": true
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
PATCH /v1/dataproducts/{id}
Content-Type: application/json
Authorization: Bearer <token>
```

### Request Payload

```json
{
  "section": "product_info",
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
  "tags_saved": 6
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
PATCH /v1/dataproducts/{id}
Content-Type: application/json
Authorization: Bearer <token>
```

### Request Payload

```json
{
  "section": "publish",
  "attest_accuracy": true,
  "attest_ownership": true
}
```

### Response `200 OK`

```json
{
  "resource_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "resource_iri": "urn:ubs:dataproduct:wma-account-master",
  "lifecycle_status": "deploy",
  "published_at": "2026-03-11T12:00:00Z",
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
    "inputs": ["..."],
    "outputs": ["..."],
    "has_policy": ["..."]
  }
}
```

### Errors

| Code | Condition |
|---|---|
| `422` | Attestations false, or required steps incomplete |

### DB Writes

| Table | Operation | Fields |
|---|---|---|
| `dcatv3.resource` | UPDATE | status='published' |
| `dprod.dataproduct` | UPDATE | lifecycle_status='deploy' |
| `internal.resource_tag` | INSERT | `attestation.accuracy`, `attestation.ownership` with timestamp |

---

## Utility Endpoints

### List Products

```
GET /v1/dataproducts?page=1&pageSize=25&lifecycle_status=build
```

| Query Param | Type | Default | Description |
|---|---|---|---|
| `page` | int | 1 | Page number (min: 1) |
| `pageSize` | int | 25 | Items per page (min: 10, max: 100) |
| `lifecycle_status` | enum | (all) | `build` / `deploy` / `consume` / `retire` |

### Get Single Product

```
GET /v1/dataproducts/{id}
```

Returns full product detail.

### Get DPF JSON

```
GET /v1/dataproducts/{id}/dpf
```

Calls `app_procs.get_dataproduct_dpf_yaml(resource_iri)`. Available at any time (returns partial DPF for draft products).

### Delete Product

```
DELETE /v1/dataproducts/{id}
```

Soft delete: sets `lifecycle_status` to `retire`.

### List Datasets

```
GET /v1/datasets?title=<filter>&limit=100&offset=0
```

### Get Dataset by Title

```
GET /v1/dataset/{title}
```

### Create Dataset (standalone)

```
POST /v1/dataset
```

### Update Dataset

```
PATCH /v1/dataset/{title}
```

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

## PATCH Section Routing Summary

All wizard updates (Steps 2-10) use the same endpoint:

```
PATCH /v1/dataproducts/{id}
```

The backend routes based on the `section` field:

| Section Value | Wizard Step | Backend Handler |
|---|---|---|
| `"datasets"` | Step 2 | Creates datasets + ports in dcatv3, dprod, platform.*, governance |
| `"entitlements"` | Step 3 | Updates access.*, governance, internal |
| `"data_quality"` | Step 5 | Stores DQ targets in internal.resource_tag |
| `"data_contract"` | Step 6 | Stores contract terms in internal.resource_tag |
| `"sensitivity"` | Step 8 | Updates governance.resource_governance + internal.resource_tag |
| `"product_info"` | Step 9 | Stores AI tags in internal.resource_tag |
| `"publish"` | Step 10 | Updates dcatv3.resource + dprod.dataproduct status |

Steps 4 and 7 are read-only GETs — no PATCH needed.

---

## Schema Gaps (require DB migration)

| DPF Field | Current Status | Recommendation |
|---|---|---|
| `cloudSubscription` | Not in `platform.dataservice_adls` | Add column or store in connection_parameters JSONB |
| `cloudResourceGroup` | Not in `platform.dataservice_adls` | Same as above |
| `serviceLevelObjectiveDescription` | Not in any table | Assemble from `internal.resource_tag` contract keys |
