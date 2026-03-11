MANDATORY: Before writing ANY code, read STYLE_BIBLE.md in the project root. Every line of code must follow these patterns exactly.
Key rules you MUST follow:

Inputs: Always use fieldClass(errors, 'fieldName') — never write custom input classes
Labels: Always use labelClass — never write custom label styles
Grid: Always grid grid-cols-2 gap-x-6 gap-y-5 for form fields — never flex or space-y
Colors: Always hex like text-[#374151] — never Tailwind names like text-gray-600
Sizes: Always pixel like text-[13px] — never text-sm or text-base
Sections: Always start with <SectionHeader title="..." /> — never raw <h2> or <h3>
Errors: Every field must have <FieldError message={errors.fieldName} /> after it
No wrapping cards inside steps — the parent RegistrationForm already wraps in a card
No shadcn <Input>, <Label>, <Select> — use plain HTML elements with our constants

The existing RegistrationForm.tsx is the gold standard. If you're unsure, copy an existing step's exact pattern.



# WMA v2 — ATOMIC TASK LIST
## One Task = One File = One Prompt to Claude Code

**RULE: Execute these IN ORDER. Do NOT skip ahead. Each task assumes all previous tasks are done.**
**RULE: Each prompt to Claude Code touches EXACTLY ONE file. Nothing else.**
**RULE: After each task, RUN `npm run dev` and verify it compiles before moving on.**

---

## EXECUTION ORDER & DEPENDENCY MAP

```
PHASE 1 — FOUNDATION (Tasks 1-8, strictly sequential)
  Task 1:  src/api/types.ts                 ← everything depends on this
  Task 2:  src/config/apiConfig.ts           ← services depend on this
  Task 3:  src/api/dpasService.ts            ← hooks depend on this
  Task 4:  src/api/externalService.ts [NEW]  ← hooks depend on this
  Task 5:  src/mocks/mockData.ts             ← handlers depend on this
  Task 6:  src/mocks/handlers.ts             ← app depends on this
  Task 7:  src/stores/useRegistrationStore.ts ← all UI depends on this
  Task 8:  src/hooks/useDataProducts.ts      ← form depends on this

PHASE 2 — WIRING (Tasks 9-13, strictly sequential)
  Task 9:  src/hooks/useFormPersistence.ts
  Task 10: src/utils/fieldMapping.ts
  Task 11: src/App.tsx
  Task 12: src/components/Sidebar.tsx
  Task 13: src/components/Header.tsx

  *** CHECKPOINT: npm run dev — app compiles, 10 steps in sidebar, navigation works ***

PHASE 3 — STEP UIs (Tasks 14-23, sequential within RegistrationForm.tsx)
  Task 14: src/components/RegistrationForm.tsx — scaffold (shared helpers + 10 placeholders)
  Task 15: src/components/RegistrationForm.tsx — Step 1 (Product Identity)
  Task 16: src/components/RegistrationForm.tsx — Step 3 (Entitlements)
  Task 17: src/components/RegistrationForm.tsx — Step 5 (Scope & Data Quality)
  Task 18: src/components/RegistrationForm.tsx — Step 6 (Data Contract)
  Task 19: src/components/RegistrationForm.tsx — Step 8 (Sensitivity & Compliance)
  Task 20: src/components/RegistrationForm.tsx — Step 9 (AI Readiness)
  Task 21: src/components/RegistrationForm.tsx — Step 10 (Submit & Attestation)
  Task 22: src/components/DatasetBuilder.tsx [NEW] — Step 2 (Datasets & Ports)
  Task 23: src/components/ReadOnlySteps.tsx [NEW] — Steps 4 & 7 (KDE + Lineage)

PHASE 4 — POLISH (Tasks 24-27)
  Task 24: src/components/SummaryPanel.tsx
  Task 25: src/validation/schemas.ts
  Task 26: src/hooks/useExternalSearch.ts [NEW]
  Task 27: vite.config.ts
```

**Total: 27 tasks. Each one is a single file. Execute in order.**

---
---

# PHASE 1 — FOUNDATION

---

## TASK 1 · `src/api/types.ts` · REPLACE ENTIRE FILE

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md first. Then REPLACE the entire file `src/api/types.ts` with the v2 types below.
> Do NOT touch any other file. Only this one file.
>
> Define these exact interfaces and types:
>
> ```typescript
> // Lifecycle statuses
> export type LifecycleStatus = "build" | "deploy" | "consume" | "retire";
> export type ProductType = "source_aligned" | "derived" | "consumer_aligned";
> export type PlatformType = "adls" | "databricks" | "postgres" | "sqlserver" | "db2" | "denodo";
> export type PortDirection = "input" | "output";
> export type SectionName = "datasets" | "entitlements" | "data_quality" | "data_contract" | "sensitivity" | "product_info" | "publish";
>
> // ── DPAS Response Types ──
> export interface DataProduct {
>   resource_iri: string;
>   resource_id: string;
>   dataproduct_id: string;
>   title: string;
>   description: string | null;
>   lifecycle_status: LifecycleStatus;
>   product_type: ProductType | null;
>   purpose: string | null;
>   domain: string | null;
>   business_division: string | null;
>   owner_email: string | null;
>   source_application_id: string | null;
>   source_application_name: string | null;
>   sub_domain: string | null;
>   maintainers_group: Record<string, string[]> | null;
>   allowed_platforms: string[] | null;
>   tags: Array<{ key: string; value: string }> | null;
>   created_at: string;
>   updated_at: string;
> }
>
> export interface DataProductListResponse {
>   items: DataProduct[];
>   total: number;
>   page: number;
>   pageSize: number;
> }
>
> // ── DPAS Payloads ──
> export interface DataProductCreatePayload {
>   title: string;
>   description?: string;
>   product_type?: ProductType;
>   purpose?: string;
>   domain?: string;
>   business_division?: string;
>   source_application_id?: string;
>   source_application_name?: string;
>   sub_domain?: string;
>   owner_email?: string;
>   owner_gpn?: string;
>   maintainers_group?: Record<string, string[]>;
>   allowed_platforms?: string[];
>   tags?: Array<{ key: string; value: string }>;
> }
>
> export interface DataProductUpdatePayload {
>   section: SectionName;
>   [key: string]: unknown;
> }
>
> // ── Dataset & Port Types (Step 2) ──
> export interface PlatformFieldsADLS {
>   storage_account: string;
>   container_name: string;
>   format: "csv" | "delta" | "parquet" | "json";
>   jurisdiction: string;
> }
> export interface PlatformFieldsDatabricks {
>   server_hostname: string;
>   database_name: string;
>   http_path: string;
> }
> export interface PlatformFieldsPostgres {
>   server_hostname: string;
>   server_port: number;
>   database_name: string;
> }
> export interface PlatformFieldsSQLServer {
>   server_hostname: string;
>   server_port: number;
>   database_name: string;
> }
> export interface PlatformFieldsDB2 {
>   server_hostname: string;
>   database_name: string;
>   location: string;
> }
> export interface PlatformFieldsDenodo {
>   server_hostname: string;
>   vdp_database: string;
>   base_view_name?: string;
>   asis_database?: string;
> }
>
> export type PlatformFields =
>   | PlatformFieldsADLS
>   | PlatformFieldsDatabricks
>   | PlatformFieldsPostgres
>   | PlatformFieldsSQLServer
>   | PlatformFieldsDB2
>   | PlatformFieldsDenodo;
>
> export interface PortConfig {
>   direction: PortDirection;
>   platform_type: PlatformType;
>   auth_types: string[];
>   protocol: string;
>   physical_schema: { file_name: string; content: string } | null;
>   platform_fields: PlatformFields;
> }
>
> export interface DatasetConfig {
>   display_name: string;
>   name: string;
>   confidentiality_classification: string;
>   cid_category: "CID" | "NON_CID" | "POTENTIAL_CID";
>   description: string;
>   related_model_id: string | null;
>   ports: PortConfig[];
> }
>
> // ── Section-specific Payloads ──
> export interface EntitlementsPayload extends DataProductUpdatePayload {
>   section: "entitlements";
>   own_entitlement_process: boolean;
>   bbs_entitlement_info?: string;
>   janus_bbs_maintainer_group: string;
>   align_to_dpas_model?: boolean;
>   align_existing_structure?: boolean;
>   eligible_roles?: string[];
>   approval_rules?: string;
>   field_level_restrictions?: string;
>   cross_domain_sharing?: boolean;
> }
>
> export interface DataQualityPayload extends DataProductUpdatePayload {
>   section: "data_quality";
>   data_scope: string;
>   refresh_frequency: string;
>   quality_rules: string;
>   completeness_target: string;
>   accuracy_target: string;
>   timeliness_target: string;
>   consistency_rules: string;
>   uniqueness_rules: string;
>   validity_rules: string;
>   fitness_score: string;
> }
>
> export interface DataContractPayload extends DataProductUpdatePayload {
>   section: "data_contract";
>   data_retention: string;
>   sla_uptime: string;
>   support_hours: string;
>   change_notification_days: string;
>   access_agreement: string;
>   breaking_changes_policy: string;
>   consumer_onboarding: string;
>   escalation_path: string;
> }
>
> export interface SensitivityPayload extends DataProductUpdatePayload {
>   section: "sensitivity";
>   data_classification: string;
>   contains_pii: boolean;
>   regulatory_requirements: string[];
>   pii_handling_procedure: string;
>   dpa_required: boolean;
>   policies: string[];
> }
>
> export interface AIReadinessPayload extends DataProductUpdatePayload {
>   section: "product_info";
>   tags: Array<{ key: string; value: string }>;
> }
>
> export interface PublishPayload extends DataProductUpdatePayload {
>   section: "publish";
>   attest_accuracy: boolean;
>   attest_ownership: boolean;
> }
>
> export interface DatasetsPayload extends DataProductUpdatePayload {
>   section: "datasets";
>   datasets: DatasetConfig[];
> }
>
> // ── Read-only Response Types ──
> export interface KDEColumn {
>   column_name: string;
>   column_type: string;
>   ontology_element: string;
>   business_definition: string;
>   confidence_score: number;
>   key_data_element: boolean;
> }
>
> export interface KDEDataset {
>   dataset_id: string;
>   name: string;
>   columns: KDEColumn[];
> }
>
> export interface KDEResponse {
>   resource_id: string;
>   datasets: KDEDataset[];
> }
>
> export interface LineageNode {
>   dataset_id: string;
>   name: string;
>   display_name: string;
>   platform_type: PlatformType;
>   dataservice_id: string;
> }
>
> export interface LineageResponse {
>   resource_id: string;
>   product_name: string;
>   inputs: LineageNode[];
>   outputs: LineageNode[];
> }
>
> // ── External API Types ──
> export interface AppDirResult {
>   app_id: string;
>   app_name: string;
>   description: string;
>   owner_team: string;
> }
>
> export interface ADGroupResult {
>   group_id: string;
>   group_name: string;
>   description: string;
>   member_count: number;
> }
>
> export interface ModelResult {
>   model_id: string;
>   model_name: string;
>   domain: string;
>   description: string;
> }
>
> // ── MetaQ Types (KEEP from current — used by catalog import) ──
> export interface MetaqDataset {
>   dataset_title: string | null;
>   name: string | null;
>   description: string | null;
>   sourcedescription: string | null;
>   aigenerateddescription: string | null;
>   ontologyid: string | null;
>   ontologyname: string | null;
>   cidstaterag: string | null;
>   cidcategory: string | null;
>   cidjustification: string | null;
>   kde: string | null;
>   dqobject: string | null;
>   recommendedusage: string | null;
>   [key: string]: unknown;
> }
>
> export interface MetaqHealthResponse {
>   status: string;
> }
>
> // ── API Error ──
> export interface ApiErrorResponse {
>   error: string;
>   message: string;
>   timestamp: string;
> }
> ```
>
> Write the file exactly as shown. Do not add, remove, or rename anything.

**Verify:** `npm run dev` — will show TypeScript errors in OTHER files (expected, they still use old types). That's fine. We'll fix those files in later tasks.

---

## TASK 2 · `src/config/apiConfig.ts` · MODIFY

**Prompt to Claude Code:**

> Modify ONLY `src/config/apiConfig.ts`. Do NOT touch any other file.
>
> Add new endpoint patterns to the `dpas` config, and add `external` config for the 3 new APIs.
>
> The file should become:
>
> ```typescript
> export const isMockMode = (): boolean =>
>   import.meta.env.VITE_USE_MOCKS === "true";
>
> export const API_CONFIG = {
>   metaq: {
>     baseUrl: import.meta.env.VITE_METAQ_API_BASE_URL || "/metaq-api",
>     endpoints: {
>       health: "/health",
>       datasets: "/datasets",
>       datasetByTitle: (title: string) => `/dataset/${encodeURIComponent(title)}`,
>       upsert: "/",
>     },
>   },
>   dpas: {
>     baseUrl: import.meta.env.VITE_DPAS_API_BASE_URL || "/v1",
>     endpoints: {
>       dataProducts: "/dataproducts",
>       dataProductById: (id: string) => `/dataproducts/${id}`,
>       dataProductKDE: (id: string) => `/dataproducts/${id}/kde`,
>       dataProductLineage: (id: string) => `/dataproducts/${id}/lineage`,
>       dataProductDPF: (id: string) => `/dataproducts/${id}/dpf`,
>     },
>   },
>   external: {
>     appDir: { search: (query: string) => `/appdir/search?q=${encodeURIComponent(query)}` },
>     adGroups: { search: (query: string) => `/adgroups/search?q=${encodeURIComponent(query)}` },
>     models: { list: (domain: string) => `/models/list?domain=${encodeURIComponent(domain)}` },
>   },
> } as const;
> ```

---

## TASK 3 · `src/api/dpasService.ts` · REPLACE ENTIRE FILE

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md. Then REPLACE the entire file `src/api/dpasService.ts`. Do NOT touch any other file.
>
> Keep the same structure (import dpasClient from httpClient, import API_CONFIG). Update type imports to match the new types.ts. Add 3 new methods: getKDE, getLineage, getDPF.
>
> The update() method signature changes: it now takes `(id: string, payload: DataProductUpdatePayload, etag?: string)`.
> The create() method uses `DataProductCreatePayload`.
> Both return `DataProduct` from the new types.
>
> New methods:
> - `getKDE(id: string, datasetId?: string): Promise<KDEResponse>` — GET `endpoints.dataProductKDE(id)` with optional `?dataset_id=` query param
> - `getLineage(id: string): Promise<LineageResponse>` — GET `endpoints.dataProductLineage(id)`
> - `getDPF(id: string): Promise<unknown>` — GET `endpoints.dataProductDPF(id)`
>
> Keep the existing list(), getById(), delete() methods but update their return types to use the new DataProduct/DataProductListResponse interfaces.

---

## TASK 4 · `src/api/externalService.ts` · CREATE NEW FILE

**Prompt to Claude Code:**

> Create a NEW file `src/api/externalService.ts`. Do NOT touch any other file.
>
> This service calls the 3 external APIs (AppDir, AD Groups, Models). In mock mode, MSW will intercept these. In real mode, the Vite proxy forwards them.
>
> Use axios directly (import axios from "axios") for these since they don't need the DPAS bearer token interceptor. Or alternatively, create a simple axios instance without interceptors.
>
> ```typescript
> import axios from "axios";
> import { API_CONFIG } from "../config/apiConfig";
> import type { AppDirResult, ADGroupResult, ModelResult } from "./types";
>
> const client = axios.create({ timeout: 15_000 });
>
> export const externalService = {
>   async searchApps(query: string): Promise<AppDirResult[]> {
>     const { data } = await client.get<AppDirResult[]>(API_CONFIG.external.appDir.search(query));
>     return Array.isArray(data) ? data : [];
>   },
>
>   async searchADGroups(query: string): Promise<ADGroupResult[]> {
>     const { data } = await client.get<ADGroupResult[]>(API_CONFIG.external.adGroups.search(query));
>     return Array.isArray(data) ? data : [];
>   },
>
>   async listModels(domain: string): Promise<ModelResult[]> {
>     const { data } = await client.get<ModelResult[]>(API_CONFIG.external.models.list(domain));
>     return Array.isArray(data) ? data : [];
>   },
> };
> ```

---

## TASK 5 · `src/mocks/mockData.ts` · MODIFY (ADD new arrays)

**Prompt to Claude Code:**

> Modify ONLY `src/mocks/mockData.ts`. Do NOT touch any other file. KEEP all existing mock data (MOCK_METAQ_DATASETS etc). ADD these new arrays at the bottom:
>
> Add `MOCK_APP_DIR_RESULTS` — array of 5 AppDirResult objects:
> - { app_id: "APP-001", app_name: "WMA Core Banking", description: "Core banking system for WMA", owner_team: "WMA-Platform" }
> - { app_id: "APP-002", app_name: "WMA Portfolio Mgmt", description: "Portfolio management and analytics", owner_team: "WMA-Investments" }
> - { app_id: "APP-003", app_name: "FX Trading Platform", description: "Foreign exchange trading system", owner_team: "WMA-Trading" }
> - { app_id: "APP-004", app_name: "Risk Analytics Engine", description: "Risk calculation and reporting", owner_team: "WMA-Risk" }
> - { app_id: "APP-005", app_name: "Client Onboarding System", description: "New client onboarding workflow", owner_team: "WMA-Operations" }
>
> Add `MOCK_AD_GROUPS` — array of 5 ADGroupResult objects:
> - { group_id: "GRP-001", group_name: "WMA-Data-Maintainers", description: "Data product maintainers", member_count: 12 }
> - { group_id: "GRP-002", group_name: "WMA-Data-Consumers", description: "Data product consumers", member_count: 45 }
> - { group_id: "GRP-003", group_name: "WMA-Risk-Analysts", description: "Risk analysis team", member_count: 8 }
> - { group_id: "GRP-004", group_name: "WMA-Compliance-Team", description: "Compliance and audit team", member_count: 6 }
> - { group_id: "GRP-005", group_name: "WMA-Data-Engineers", description: "Data engineering team", member_count: 15 }
>
> Add `MOCK_MODELS` — array of 3 ModelResult objects:
> - { model_id: "model-001", model_name: "Account Master Model", domain: "Account", description: "Canonical account data model" }
> - { model_id: "model-002", model_name: "Portfolio Holdings Model", domain: "Portfolio", description: "Investment holdings schema" }
> - { model_id: "model-003", model_name: "Transaction Model", domain: "Trading", description: "Trade execution data model" }
>
> Add `MOCK_KDE_COLUMNS` — array of 6 KDEColumn objects:
> - { column_name: "account_id", column_type: "VARCHAR(50)", ontology_element: "Account Identifier", business_definition: "Unique client account number", confidence_score: 0.95, key_data_element: true }
> - { column_name: "client_name", column_type: "VARCHAR(255)", ontology_element: "Client Name", business_definition: "Legal name of the client", confidence_score: 0.92, key_data_element: true }
> - { column_name: "market_value", column_type: "DECIMAL(18,2)", ontology_element: "Market Value", business_definition: "Current market value of position", confidence_score: 0.88, key_data_element: true }
> - { column_name: "currency_code", column_type: "CHAR(3)", ontology_element: "Currency", business_definition: "ISO 4217 currency code", confidence_score: 0.97, key_data_element: false }
> - { column_name: "last_updated", column_type: "TIMESTAMP", ontology_element: "Update Timestamp", business_definition: "Last modification timestamp", confidence_score: 0.78, key_data_element: false }
> - { column_name: "status_flag", column_type: "CHAR(1)", ontology_element: "Status Indicator", business_definition: "Active/Inactive status flag", confidence_score: 0.75, key_data_element: false }
>
> Import the types from "../api/types" as needed. Export all new arrays.

---

## TASK 6 · `src/mocks/handlers.ts` · REPLACE ENTIRE FILE

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md. REPLACE the entire file `src/mocks/handlers.ts`. Do NOT touch any other file.
>
> Import from msw: `http`, `HttpResponse`. Import mock data from `./mockData`. Import types from `../api/types`.
>
> Create an in-memory store: `let products: DataProduct[] = [];`
>
> Define these 13 handlers in the `handlers` array:
>
> 1. `http.post("*/v1/dataproducts", ...)` — Read JSON body. Generate UUIDs for resource_id, resource_iri, dataproduct_id. Push to products array. Return 201 with full DataProduct shape. lifecycle_status = "build".
>
> 2. `http.patch("*/v1/dataproducts/:id", ...)` — Read JSON body. Read `section` field. Return 200 with `{ resource_id: id, section, saved: true }`. For section="datasets", also return mock dataset_ids and port_ids. For section="publish", return lifecycle_status: "deploy" + dpf_yaml_data object.
>
> 3. `http.get("*/v1/dataproducts/:id", ...)` — Find product by id in array. Return 200 or 404.
>
> 4. `http.get("*/v1/dataproducts", ...)` — Return `{ items: products, total: products.length, page: 1, pageSize: 25 }`.
>
> 5. `http.delete("*/v1/dataproducts/:id", ...)` — Set lifecycle_status="retire". Return product.
>
> 6. `http.get("*/v1/dataproducts/:id/kde", ...)` — Return KDEResponse with 1 mock dataset containing MOCK_KDE_COLUMNS.
>
> 7. `http.get("*/v1/dataproducts/:id/lineage", ...)` — Return LineageResponse with 2 mock inputs and 1 mock output.
>
> 8. `http.get("*/v1/dataproducts/:id/dpf", ...)` — Return a mock DPF JSON object.
>
> 9. `http.get("*/metaq-api/datasets*", ...)` — Return MOCK_METAQ_DATASETS (keep existing behavior).
>
> 10. `http.get("*/metaq-api/dataset/:title", ...)` — Return single dataset filtered by title.
>
> 11. `http.get("*/appdir/search*", ...)` — Filter MOCK_APP_DIR_RESULTS by query param `q`. Return matching results.
>
> 12. `http.get("*/adgroups/search*", ...)` — Filter MOCK_AD_GROUPS by query param `q`. Return matching results.
>
> 13. `http.get("*/models/list*", ...)` — Filter MOCK_MODELS by query param `domain`. Return matching results.
>
> Use crypto.randomUUID() for generating IDs. Export `handlers` array as default.

---

## TASK 7 · `src/stores/useRegistrationStore.ts` · REPLACE ENTIRE FILE

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md. REPLACE the entire file `src/stores/useRegistrationStore.ts`. Do NOT touch any other file.
>
> Import `create` from "zustand". Import `DatasetConfig, PortConfig, MetaqDataset` from "../api/types".
>
> Define the store with these exact fields and defaults (copy-paste, do not improvise):
>
> **Metadata fields:**
> - dpasResourceId: string | null = null
> - dpasResourceIri: string | null = null
> - dpasDataproductId: string | null = null
> - dpasEtag: string | null = null
> - lastSavedAt: string | null = null
> - metaqSource: string | null = null
>
> **Step 1 fields:**
> - ownerEmail: string = ""
> - ownerGpn: string = ""
> - displayName: string = ""
> - name: string = ""
> - description: string = ""
> - productType: string = ""
> - sourceApplicationId: string = ""
> - sourceApplicationName: string = ""
> - owningBusinessDivision: string = ""
> - domain: string = ""
> - subDomain: string = ""
> - maintainersGroupProd: string[] = []
> - allowedPlatforms: string[] = []
> - tags: Array<{key: string; value: string}> = []
>
> **Step 2 fields:**
> - datasets: DatasetConfig[] = []
>
> **Step 3 fields:**
> - ownEntitlementProcess: boolean = false
> - bbsEntitlementInfo: string = ""
> - janusBbsMaintainerGroup: string = ""
> - alignToDpasModel: boolean = true
> - alignExistingStructure: boolean = false
> - eligibleRoles: string[] = []
> - approvalRules: string = ""
> - fieldLevelRestrictions: string = ""
> - crossDomainSharing: boolean = false
>
> **Step 5 fields:**
> - dataScope: string = "Global"
> - refreshFrequency: string = "Daily"
> - qualityRules: string = ""
> - completenessTarget: string = ""
> - accuracyTarget: string = ""
> - timelinessTarget: string = ""
> - consistencyRules: string = ""
> - uniquenessRules: string = ""
> - validityRules: string = ""
> - fitnessScore: string = ""
>
> **Step 6 fields:**
> - dataRetention: string = "7 years"
> - slaUptime: string = "99.9%"
> - supportHours: string = "24/7"
> - changeNotificationDays: string = "30"
> - accessAgreement: string = ""
> - breakingChangesPolicy: string = ""
> - consumerOnboarding: string = ""
> - escalationPath: string = ""
>
> **Step 8 fields:**
> - dataClassification: string = "Confidential"
> - containsPii: boolean = true
> - regulatoryRequirements: string[] = []
> - piiHandlingProcedure: string = ""
> - dpaRequired: boolean = false
> - policies: string[] = []
>
> **Step 9 fields:**
> - mlFeatureStore: string = "Not Applicable"
> - aiModelUsage: string = "Not Applicable"
> - embeddingSupport: string = "Not Applicable"
> - biasAssessment: string = "Not Applicable"
> - explainabilityRequirement: string = "Not Applicable"
> - modelGovernance: string = "Not Applicable"
>
> **Step 10 fields:**
> - attestAccuracy: boolean = false
> - attestOwnership: boolean = false
>
> **Actions:**
> - updateFields: (fields) => set(fields) — shallow merge
> - addDataset: (dataset) => set(state => ({ datasets: [...state.datasets, dataset] }))
> - updateDataset: (index, updates) => set(state => { const d = [...state.datasets]; d[index] = { ...d[index], ...updates }; return { datasets: d }; })
> - removeDataset: (index) => set(state => ({ datasets: state.datasets.filter((_, i) => i !== index) }))
> - addPort: (dsIdx, port) => set(state => { const d = [...state.datasets]; d[dsIdx] = { ...d[dsIdx], ports: [...d[dsIdx].ports, port] }; return { datasets: d }; })
> - removePort: (dsIdx, portIdx) => set(state => { const d = [...state.datasets]; d[dsIdx] = { ...d[dsIdx], ports: d[dsIdx].ports.filter((_, i) => i !== portIdx) }; return { datasets: d }; })
> - prefillFromMetaq: (dataset: MetaqDataset) => set({ displayName: dataset.dataset_title || "", description: dataset.description || "", metaqSource: dataset.dataset_title })
> - markSavedToDpas: (resourceId, resourceIri, dataproductId, etag) => set({ dpasResourceId: resourceId, dpasResourceIri: resourceIri, dpasDataproductId: dataproductId, dpasEtag: etag, lastSavedAt: new Date().toISOString() })
> - reset: () => set(DEFAULTS)
> - getStepCompletion: () => { /* compute per-step fill counts for 10 steps using STEP_FIELDS array */ }
>
> Define STEP_FIELDS as array of 10 arrays, each listing the field keys for that step. Fields with non-empty defaults (dataScope, refreshFrequency, dataRetention, slaUptime, supportHours, changeNotificationDays, dataClassification, containsPii) always count as filled. Boolean fields count as filled when true. String fields count as filled when trimmed non-empty. Array fields count as filled when length > 0. Steps 4 and 7 have no fields (they are read-only) — return { filled: 0, total: 0 } for those.
>
> Export the store hook as `useRegistrationStore` and the type as `RegistrationStore`.

---

## TASK 8 · `src/hooks/useDataProducts.ts` · REPLACE ENTIRE FILE

**Prompt to Claude Code:**

> REPLACE the entire file `src/hooks/useDataProducts.ts`. Do NOT touch any other file.
>
> Keep the same structure: useQuery + useMutation hooks wrapping dpasService. Update all type imports to use the new types from "../api/types".
>
> Keep existing hooks: useDataProductList, useDataProduct, useCreateDataProduct, useUpdateDataProduct, useDeleteDataProduct.
>
> Update useCreateDataProduct's onSuccess to call `markSavedToDpas(data.resource_id, data.resource_iri, data.dataproduct_id, null)` — note the new 4-arg signature.
>
> ADD 3 new hooks:
> - `useKDE(id: string | null)` — useQuery calling dpasService.getKDE(id!), enabled when !!id
> - `useLineage(id: string | null)` — useQuery calling dpasService.getLineage(id!), enabled when !!id
> - `useDPF(id: string | null)` — useQuery calling dpasService.getDPF(id!), enabled when !!id

---

## TASK 9 · `src/hooks/useFormPersistence.ts` · MODIFY

**Prompt to Claude Code:**

> Modify ONLY `src/hooks/useFormPersistence.ts`. Do NOT touch any other file.
>
> Replace the STATE_KEYS array with ALL the new field names from the v2 store. The list is:
>
> Step 1: "ownerEmail", "ownerGpn", "displayName", "name", "description", "productType", "sourceApplicationId", "sourceApplicationName", "owningBusinessDivision", "domain", "subDomain", "maintainersGroupProd", "allowedPlatforms", "tags"
> Step 2: "datasets"
> Step 3: "ownEntitlementProcess", "bbsEntitlementInfo", "janusBbsMaintainerGroup", "alignToDpasModel", "alignExistingStructure", "eligibleRoles", "approvalRules", "fieldLevelRestrictions", "crossDomainSharing"
> Step 5: "dataScope", "refreshFrequency", "qualityRules", "completenessTarget", "accuracyTarget", "timelinessTarget", "consistencyRules", "uniquenessRules", "validityRules", "fitnessScore"
> Step 6: "dataRetention", "slaUptime", "supportHours", "changeNotificationDays", "accessAgreement", "breakingChangesPolicy", "consumerOnboarding", "escalationPath"
> Step 8: "dataClassification", "containsPii", "regulatoryRequirements", "piiHandlingProcedure", "dpaRequired", "policies"
> Step 9: "mlFeatureStore", "aiModelUsage", "embeddingSupport", "biasAssessment", "explainabilityRequirement", "modelGovernance"
> Step 10: "attestAccuracy", "attestOwnership"
> Metadata: "dpasResourceId", "dpasResourceIri", "dpasDataproductId", "dpasEtag", "metaqSource", "lastSavedAt"
>
> Update the WizardFormState type import — it no longer exists. Instead, just type STATE_KEYS as string[] and extractState as returning Record<string, unknown>.
>
> Keep all other logic (debounce, restore, clearDraft) exactly the same.

---

## TASK 10 · `src/utils/fieldMapping.ts` · REPLACE ENTIRE FILE

**Prompt to Claude Code:**

> REPLACE the entire file `src/utils/fieldMapping.ts`. Do NOT touch any other file.
>
> Create two functions:
>
> `toDpasCreatePayload(store)` — maps v2 store fields to DataProductCreatePayload:
> - store.displayName → title
> - store.description → description
> - store.productType → product_type (only if non-empty)
> - store.description (first sentence) → purpose
> - store.domain → domain (only if non-empty)
> - store.owningBusinessDivision → business_division (only if non-empty)
> - store.sourceApplicationId → source_application_id
> - store.sourceApplicationName → source_application_name
> - store.subDomain → sub_domain (only if non-empty)
> - store.ownerEmail → owner_email
> - store.ownerGpn → owner_gpn (only if non-empty)
> - store.maintainersGroupProd → maintainers_group: { PROD: store.maintainersGroupProd }
> - store.allowedPlatforms → allowed_platforms
> - store.tags → tags
>
> `fromMetaq(dataset: MetaqDataset)` — maps MetaQ dataset to partial store fields for prefill:
> - dataset.dataset_title → displayName
> - dataset.description → description
> - dataset.ontologyname → domain (if available)
> - dataset.cidcategory → (no direct mapping, but store as metaqSource)
>
> Import types from "../api/types".

---

## TASK 11 · `src/App.tsx` · MODIFY

**Prompt to Claude Code:**

> Modify ONLY `src/App.tsx`. Do NOT touch any other file.
>
> Change the `stepNames` array from 8 items to 10 items:
> ```
> const stepNames = [
>   'Product Identity',
>   'Datasets & Ports',
>   'Entitlements',
>   'KDE Semantics',
>   'Scope & Data Quality',
>   'Data Contract',
>   'Data Lineage',
>   'Sensitivity & Compliance',
>   'AI Readiness',
>   'Attestation & Submit',
> ];
> ```
>
> No other changes needed. The Sidebar, Header, and RegistrationForm components receive step info via props.

---

## TASK 12 · `src/components/Sidebar.tsx` · MODIFY

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md first. Modify ONLY `src/components/Sidebar.tsx`. Do NOT touch any other file.
>
> Changes:
> 1. Update STEP_META from 8 to 10 entries:
>    - { title: "Product Identity", sub: "Name, type, owner & platforms", required: true }
>    - { title: "Datasets & Ports", sub: "Create datasets, configure I/O", required: true }
>    - { title: "Entitlements", sub: "Access control & permissions", required: true }
>    - { title: "KDE Semantics", sub: "Column metadata (read-only)", required: false }
>    - { title: "Scope & Data Quality", sub: "DQ targets & fitness rules", required: true }
>    - { title: "Data Contract", sub: "SLA, retention & onboarding", required: true }
>    - { title: "Data Lineage", sub: "Visual flow map (read-only)", required: false }
>    - { title: "Sensitivity", sub: "Classification, PII & policies", required: true }
>    - { title: "AI Readiness", sub: "ML features & governance", required: false }
>    - { title: "Submit", sub: "Attestation & publish", required: true }
>
> 2. REMOVE the `getStepAccessibility` function entirely.
> 3. REMOVE the `isAccessible` check from handleStepClick — every step is always clickable:
>    ```
>    const handleStepClick = (stepId: number) => { onStepChange(stepId); };
>    ```
> 4. REMOVE `cursor-not-allowed opacity-40` conditional — all steps are always fully visible.
> 5. Add read-only step styling: Steps 4 and 7 (where required=false) get blue theme: circle `bg-[#0284c7] text-white`, background `bg-[#f0f9ff]`, border `border border-[#e0f2fe]`. Import `Eye` from lucide-react and show it instead of the step number for read-only steps.
> 6. Update the bottom status: "Step {currentStep} of 10 · {10 - currentStep} remaining"
> 7. Keep the circular progress ring, step completion tracking, and all existing visual patterns.

---

## TASK 13 · `src/components/Header.tsx` · MINOR MODIFY

**Prompt to Claude Code:**

> Modify ONLY `src/components/Header.tsx`. Do NOT touch any other file.
>
> The ONLY change: there is NO hardcoded "8" in this file — but double-check. If you find any reference to 8 steps, change it to 10. Otherwise, no changes needed. The Header receives currentStep and stepName as props and doesn't hardcode step count.

---

## **CHECKPOINT: Run `npm run dev`. The app MUST compile. Sidebar shows 10 steps. Free navigation works. All mock endpoints work.**

---
---

# PHASE 3 — STEP UIs (all inside RegistrationForm.tsx or new component files)

---

## TASK 14 · `src/components/RegistrationForm.tsx` · SCAFFOLD

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md FIRST — follow it exactly.
>
> REPLACE the entire file `src/components/RegistrationForm.tsx`. Do NOT touch any other file.
>
> Create the SCAFFOLD version with:
> 1. ALL the shared helpers at the top (COPY from current file): SectionHeader, HelpTip, InfoTip, StepProps, inputBase, inputOk, inputErr, fieldClass, labelClass, FieldError — EXACT same code, do not modify these helpers.
> 2. The main RegistrationForm component with the same outer structure: step header with icon, progress bar, form card wrapper, navigation buttons, step animation, keyboard shortcuts. Update stepInfo to 10 entries with correct icons: Package, Database, Lock, BookOpen, BarChart3, FileText, GitBranch, Shield, Cpu, Send. Update all "8" references to "10". Import new icons from lucide-react.
> 3. For steps 4 and 7 (read-only), show "Continue to Next Step" button instead of "Save & Continue".
> 4. 10 placeholder step functions (Step1 through Step10), each rendering:
>    ```tsx
>    function StepN({ errors, onChange }: StepProps) {
>      return (
>        <div>
>          <SectionHeader title="Step N Title" description="Coming soon." />
>          <div className="flex flex-col items-center justify-center py-16 text-[#9ca3af]">
>            <StepIcon className="w-10 h-10 mb-3 text-[#d1d5db]" />
>            <p className="text-[13px] font-medium">Step content coming soon</p>
>          </div>
>        </div>
>      );
>    }
>    ```
> 5. Import statements for useRegistrationStore (new store), useCreateDataProduct, useUpdateDataProduct, toDpasCreatePayload from new locations.
> 6. The handleContinue and handleSubmit functions should work with the new store fields. handleSubmit is now on step 10 (not step 8).
> 7. The Import from Catalog button and DatasetBrowser modal — KEEP on step 1.
>
> This is the scaffold. Individual steps will be filled in Tasks 15-21.

---

## TASK 15 · `src/components/RegistrationForm.tsx` · FILL Step 1

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md FIRST. Modify ONLY the Step1 function inside `src/components/RegistrationForm.tsx`. Do NOT touch any other step function or the main component.
>
> Replace the placeholder Step1 with the full Product Identity form. Follow STYLE_BIBLE.md exactly:
> - Use `const s = useRegistrationStore();`
> - Use `fieldClass(errors, 'fieldName')` for every input
> - Use `labelClass` for every label
> - Use `<FieldError message={errors.fieldName} />` after every field
> - Use `grid grid-cols-2 gap-x-6 gap-y-5` for layout
>
> Fields (in order):
> 1. Owner — `<input>` pre-filled with s.ownerEmail, `onChange('ownerEmail', ...)`
> 2. Display Name — `<input>` with placeholder "e.g. WMA Account Master"
> 3. Name (slug) — `<input>` with placeholder "auto-generated", font-mono class, `col-span-2` ← full width since slug can be long
> 4. Description — `<textarea rows={3}>` with `col-span-2`, char counter `{s.description.length}/2000`
> 5. Type — Radio card group, `col-span-2`, 3 cards in `grid grid-cols-3 gap-4`: Source Aligned (Database icon), Derived (GitBranch icon), Consumer Aligned (Users icon). Selected: `border-2 border-[#e60028] bg-[#fff5f5]`. Unselected: `border border-[#e5e7eb] bg-white hover:border-[#c0c0c0]`. Each card: `rounded-xl p-4 cursor-pointer transition-all duration-150 text-center`. Icon `w-5 h-5 mx-auto mb-2 text-[#9ca3af]`. Title `text-[13px] font-semibold text-[#111]`. Desc `text-[11px] text-[#6b7280] mt-1`.
> 6. Source Application — `<input>` with placeholder "Search applications..." (typeahead will be wired in Task 26, for now just a plain input storing sourceApplicationName)
> 7. Owning Business Division — `<select>` with options: "", "Wealth Management Americas", "Global Wealth Management", "Personal & Corporate Banking", "Asset Management", "Investment Bank"
> 8. Domain — `<select>` with options: "", "Account", "Client", "Portfolio", "Trading", "Risk", "Reference Data", "Compliance", "Operations"
> 9. Sub Domain — `<select>` disabled when domain is empty, with `opacity-50 cursor-not-allowed bg-[#f9fafb]` when disabled. Domain-specific options (hardcode a simple map).
> 10. Maintainers Group — `<input>` with placeholder "Search AD groups..." (typeahead wired later, for now plain input storing janusBbsMaintainerGroup as text)
> 11. Allowed Platforms — `col-span-2`, horizontal row of clickable pill-buttons: PowerBI, DevPod, Denodo, Databricks, Kafka. Each pill: `px-3 py-1.5 rounded-lg text-[12px] font-medium border cursor-pointer transition-all`. Selected: `bg-[#fef2f2] text-[#e60028] border-[#fecaca]`. Unselected: `bg-white text-[#6b7280] border-[#e5e7eb] hover:border-[#c0c0c0]`. Use s.allowedPlatforms array, toggle on click.
> 12. Tags — `col-span-2`, "+ Add Tag" button. Each tag row: key input + value input + X remove button in a flex row. Store in s.tags array.

---

## TASK 16-21 · Same pattern for Steps 3, 5, 6, 8, 9, 10

**Each task follows the exact same prompt template:**

> Read STYLE_BIBLE.md FIRST. Modify ONLY the Step{N} function inside `src/components/RegistrationForm.tsx`. Do NOT touch any other step function or the main component.
>
> Replace the placeholder with the full form. Follow STYLE_BIBLE exactly. Use fieldClass, labelClass, FieldError, SectionHeader, grid grid-cols-2 gap-x-6 gap-y-5.
>
> [Then list the specific fields for that step from the task plan]

I won't repeat all the fields here — they're already in the task plan. The key difference is: **each prompt touches ONLY ONE function inside ONE file**.

---

## TASK 22 · `src/components/DatasetBuilder.tsx` · CREATE NEW FILE

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md FIRST. Create a NEW file `src/components/DatasetBuilder.tsx`. Do NOT touch any other file.
>
> This component handles Step 2 (Datasets & Ports). It will be imported into RegistrationForm.tsx's Step2 function.
>
> [Include the full dataset builder spec from the task plan — empty state, dataset cards, Add Dataset dialog, Add Port dialog, platform-specific fields, CID badges, port mini-cards]
>
> Use the STYLE_BIBLE modal pattern (copy DatasetBrowser.tsx structure). Use fieldClass/labelClass/FieldError for all form fields inside dialogs.

Then a small follow-up task:

> Modify ONLY the Step2 function inside `src/components/RegistrationForm.tsx`. Import DatasetBuilder from './DatasetBuilder' and render `<DatasetBuilder />`. Pass the store and error props.

---

## TASK 23 · `src/components/ReadOnlySteps.tsx` · CREATE NEW FILE

**Prompt to Claude Code:**

> Read STYLE_BIBLE.md FIRST. Create a NEW file `src/components/ReadOnlySteps.tsx`. Do NOT touch any other file.
>
> Export two components: `KDESemanticsView` and `DataLineageView`.
>
> [Include the full KDE table spec and Lineage flow spec from the task plan]
>
> These are read-only — no form fields, no save. Just data display.

Then a small follow-up:

> Modify ONLY the Step4 and Step7 functions inside `src/components/RegistrationForm.tsx`. Import KDESemanticsView and DataLineageView from './ReadOnlySteps'. Render them.

---
---

# PHASE 4 — POLISH

---

## TASK 24 · `src/components/SummaryPanel.tsx` · MODIFY

> Modify ONLY `src/components/SummaryPanel.tsx`. Update the `sections` array to reference the new store field names for 10 steps.

---

## TASK 25 · `src/validation/schemas.ts` · REPLACE ENTIRE FILE

> REPLACE `src/validation/schemas.ts` with 10 Zod schemas matching v2 fields. [Include schema specs from task plan]

---

## TASK 26 · `src/hooks/useExternalSearch.ts` · CREATE NEW FILE

> Create new hooks: useAppDirSearch, useADGroupSearch, useModelList. Then wire them into Step1's Source Application and Maintainers Group fields.

---

## TASK 27 · `vite.config.ts` · MODIFY

> Add 3 proxy entries for /appdir, /adgroups, /models. Extend OPTIONS handler.

---
---

# SUMMARY

| # | File | Action | Phase |
|---|------|--------|-------|
| 1 | src/api/types.ts | REPLACE | 1 |
| 2 | src/config/apiConfig.ts | MODIFY | 1 |
| 3 | src/api/dpasService.ts | REPLACE | 1 |
| 4 | src/api/externalService.ts | CREATE | 1 |
| 5 | src/mocks/mockData.ts | MODIFY | 1 |
| 6 | src/mocks/handlers.ts | REPLACE | 1 |
| 7 | src/stores/useRegistrationStore.ts | REPLACE | 1 |
| 8 | src/hooks/useDataProducts.ts | REPLACE | 1 |
| 9 | src/hooks/useFormPersistence.ts | MODIFY | 2 |
| 10 | src/utils/fieldMapping.ts | REPLACE | 2 |
| 11 | src/App.tsx | MODIFY | 2 |
| 12 | src/components/Sidebar.tsx | MODIFY | 2 |
| 13 | src/components/Header.tsx | MINOR | 2 |
| 14 | src/components/RegistrationForm.tsx | SCAFFOLD | 3 |
| 15 | src/components/RegistrationForm.tsx | Step 1 | 3 |
| 16 | src/components/RegistrationForm.tsx | Step 3 | 3 |
| 17 | src/components/RegistrationForm.tsx | Step 5 | 3 |
| 18 | src/components/RegistrationForm.tsx | Step 6 | 3 |
| 19 | src/components/RegistrationForm.tsx | Step 8 | 3 |
| 20 | src/components/RegistrationForm.tsx | Step 9 | 3 |
| 21 | src/components/RegistrationForm.tsx | Step 10 | 3 |
| 22 | src/components/DatasetBuilder.tsx | CREATE | 3 |
| 23 | src/components/ReadOnlySteps.tsx | CREATE | 3 |
| 24 | src/components/SummaryPanel.tsx | MODIFY | 4 |
| 25 | src/validation/schemas.ts | REPLACE | 4 |
| 26 | src/hooks/useExternalSearch.ts | CREATE | 4 |
| 27 | vite.config.ts | MODIFY | 4 |

**27 atomic tasks. Each one = one file = one prompt. Execute in order. Run `npm run dev` after each.**
