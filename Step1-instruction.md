# Data Product Registration — Complete Change Log

## New Step 1: “Data Agreement & Access”

> **Date:** March 5, 2026
> **Scope:** Add a new 9th step (as Step 1) to the wizard, shifting all existing steps from 1–8 to 2–9
> **Source:** Screenshot requirements spreadsheet showing “Data Agreement & Access” page with 3 sections

-----

## What Changed (High-Level)

The wizard was expanded from **8 steps** to **9 steps** by inserting a brand-new **“Data Agreement & Access”** step as the first step. This step has 3 sections with **20 new fields**, including **conditional show/hide logic** in the Entitlements section. All existing steps shifted by +1 (old Step 1 became Step 2, etc.).

**Files modified:** 8 files total across types, store, validation, persistence, app layout, sidebar, registration form, and summary panel.

-----

## Task 1 — Add new fields to `src/api/types.ts`

### What changed

Added 20 new field definitions to the `WizardFormState` interface as the new Step 1. Renumbered all existing step comments from Steps 1–8 to Steps 2–9.

### Exact change

**Find this (around line 71):**

```ts
export interface WizardFormState {
  // Step 1 – Product Identity & Purpose (8 fields)
  productName: string;
```

**Replace with:**

```ts
export interface WizardFormState {
  // Step 1 – Data Agreement & Access (NEW)
  // Section A: Data Agreement & Functionality
  refreshFrequencyAgreement: string;
  availabilitySLA: string;
  ownedEndUserConsumptionPattern: string;
  usageGuidance: string;
  deprecationPolicy: string;
  highPerformantAgreement: string;
  sourceManagedConsumptionPatterns: string;
  // Section B: Sensitivity, Policy & Compliance
  overallClassification: string;
  retentionRequirement: string;
  regulatoryDisclaimers: string;
  dataProductCID: string;
  // Section C: Entitlements
  ownEntitlementProcessAgreement: string;
  entitlementInfoBBS: string;
  alignToDPASEntitlementModel: string;
  alignToExistingEntitlementStructure: string;
  eligibleRolesGroups: string;
  approvalRules: string;
  fieldLevelRestrictions: string;
  crossDomainSharing: string;
  janusBBSMaintainerGroup: string;

  // Step 2 – Product Identity & Purpose (8 fields)
  productName: string;
```

**Then rename all step comments below it:**

- `// Step 2 – Ownership & Support` → `// Step 3 – Ownership & Support`
- `// Step 3 – Source & Entitlements` → `// Step 4 – Source & Entitlements`
- `// Step 4 – Semantics & Data Quality` → `// Step 5 – Semantics & Data Quality`
- `// Step 5 – Data Contract & Access` → `// Step 6 – Data Contract & Access`
- `// Step 6 – Lineage & Transformations` → `// Step 7 – Lineage & Transformations`
- `// Step 7 – Compliance & AI Readiness` → `// Step 8 – Compliance & AI Readiness`
- `// Step 8 – Attestation & Submit` → `// Step 9 – Attestation & Submit`

### New fields added (20 total)

**Section A: Data Agreement & Functionality (7 fields)**

|Field Key                         |Type  |Description                                  |
|----------------------------------|------|---------------------------------------------|
|`refreshFrequencyAgreement`       |string|How frequently the data is updated (dropdown)|
|`availabilitySLA`                 |string|Availability SLA percentage (dropdown)       |
|`ownedEndUserConsumptionPattern`  |string|Consumer pattern description (textarea)      |
|`usageGuidance`                   |string|In House vs Central Access (text)            |
|`deprecationPolicy`               |string|Record management policy (text)              |
|`highPerformantAgreement`         |string|Yes/No select                                |
|`sourceManagedConsumptionPatterns`|string|Source-managed patterns (textarea)           |

**Section B: Sensitivity, Policy & Compliance (4 fields)**

|Field Key              |Type  |Description                                       |
|-----------------------|------|--------------------------------------------------|
|`overallClassification`|string|Public/Internal/Confidential/Restricted (dropdown)|
|`retentionRequirement` |string|7/5/3/1 years (dropdown)                          |
|`regulatoryDisclaimers`|string|Regulatory frameworks like GDPR, SOX (textarea)   |
|`dataProductCID`       |string|Data Product CID identifier (text)                |

**Section C: Entitlements (9 fields)**

|Field Key                            |Type           |Show When                |
|-------------------------------------|---------------|-------------------------|
|`ownEntitlementProcessAgreement`     |string (Yes/No)|Always visible           |
|`entitlementInfoBBS`                 |string         |Only when answer is “Yes”|
|`alignToDPASEntitlementModel`        |string         |Only when answer is “No” |
|`alignToExistingEntitlementStructure`|string         |Only when answer is “No” |
|`eligibleRolesGroups`                |string         |Only when answer is “No” |
|`approvalRules`                      |string         |Only when answer is “No” |
|`fieldLevelRestrictions`             |string         |Only when answer is “No” |
|`crossDomainSharing`                 |string         |Only when answer is “No” |
|`janusBBSMaintainerGroup`            |string         |Only when answer is “No” |

-----

## Task 2 — Add defaults for new fields in `src/stores/useRegistrationStore.ts`

### What changed

Added default values for all 20 new fields in the `DEFAULTS` object, added the new step’s field list to the `STEP_FIELDS` array, added 6 entries to `FIELDS_WITH_DEFAULTS`, and renumbered all step comments.

### Change 1 — In `DEFAULTS` object

**Find:**

```ts
const DEFAULTS: WizardFormState = {
  // Step 1 – Product Identity & Purpose
  productName: "",
```

**Replace with:**

```ts
const DEFAULTS: WizardFormState = {
  // Step 1 – Data Agreement & Access (NEW)
  // Section A: Data Agreement & Functionality
  refreshFrequencyAgreement: "Daily",
  availabilitySLA: "99.9%",
  ownedEndUserConsumptionPattern: "",
  usageGuidance: "",
  deprecationPolicy: "",
  highPerformantAgreement: "No",
  sourceManagedConsumptionPatterns: "",
  // Section B: Sensitivity, Policy & Compliance
  overallClassification: "Internal",
  retentionRequirement: "7 years",
  regulatoryDisclaimers: "",
  dataProductCID: "",
  // Section C: Entitlements
  ownEntitlementProcessAgreement: "No",
  entitlementInfoBBS: "",
  alignToDPASEntitlementModel: "",
  alignToExistingEntitlementStructure: "",
  eligibleRolesGroups: "",
  approvalRules: "",
  fieldLevelRestrictions: "",
  crossDomainSharing: "",
  janusBBSMaintainerGroup: "",

  // Step 2 – Product Identity & Purpose
  productName: "",
```

**Then rename all step comments below (same pattern as Task 1: Step 2→3, 3→4, … 8→9).**

### Change 2 — In `STEP_FIELDS` array

**Find:**

```ts
const STEP_FIELDS: (keyof WizardFormState)[][] = [
  // Step 1 – Product Identity & Purpose
```

**Replace with:**

```ts
const STEP_FIELDS: (keyof WizardFormState)[][] = [
  // Step 1 – Data Agreement & Access (NEW)
  ["refreshFrequencyAgreement", "availabilitySLA", "ownedEndUserConsumptionPattern", "usageGuidance", "deprecationPolicy", "highPerformantAgreement", "sourceManagedConsumptionPatterns", "overallClassification", "retentionRequirement", "regulatoryDisclaimers", "dataProductCID", "ownEntitlementProcessAgreement", "entitlementInfoBBS", "alignToDPASEntitlementModel", "alignToExistingEntitlementStructure", "eligibleRolesGroups", "approvalRules", "fieldLevelRestrictions", "crossDomainSharing", "janusBBSMaintainerGroup"],
  // Step 2 – Product Identity & Purpose
```

**(And rename the remaining step comments in the array to 3–9.)**

### Change 3 — In `FIELDS_WITH_DEFAULTS`

**Find:**

```ts
const FIELDS_WITH_DEFAULTS = new Set<keyof WizardFormState>([
  "environment",
```

**Replace with:**

```ts
const FIELDS_WITH_DEFAULTS = new Set<keyof WizardFormState>([
  // Step 1 – Data Agreement & Access
  "refreshFrequencyAgreement",
  "availabilitySLA",
  "highPerformantAgreement",
  "overallClassification",
  "retentionRequirement",
  "ownEntitlementProcessAgreement",
  // Steps 2–9
  "environment",
```

-----

## Task 3 — Add validation schema for new Step 1 in `src/validation/schemas.ts`

### What changed

Added a new Zod `step1Schema` that validates 12 required fields (Section A + B + main entitlement question). Renamed all existing schemas from step1–step8 to step2–step9. Updated STEP_KEYS, SCHEMAS arrays, and JSDoc.

### Change 1 — Rename all existing schema variables (do in reverse order to avoid conflicts):

- `const step8Schema` → `const step9Schema`
- `const step7Schema` → `const step8Schema`
- `const step6Schema` → `const step7Schema`
- `const step5Schema` → `const step6Schema`
- `const step4Schema` → `const step5Schema`
- `const step3Schema` → `const step4Schema`
- `const step2Schema` → `const step3Schema`
- `const step1Schema` → `const step2Schema`

### Change 2 — Change section header comment:

`// ── Step Schemas (8 Steps)` → `// ── Step Schemas (9 Steps)`

### Change 3 — Add new `step1Schema` right ABOVE `const step2Schema`:

```ts
// Step 1: Section A & B are required; Section C entitlement sub-fields are conditional
const step1Schema = z.object({
  refreshFrequencyAgreement: requiredString,
  availabilitySLA: requiredString,
  ownedEndUserConsumptionPattern: requiredString,
  usageGuidance: requiredString,
  deprecationPolicy: requiredString,
  highPerformantAgreement: requiredString,
  sourceManagedConsumptionPatterns: requiredString,
  overallClassification: requiredString,
  retentionRequirement: requiredString,
  regulatoryDisclaimers: requiredString,
  dataProductCID: requiredString,
  ownEntitlementProcessAgreement: requiredString,
});
```

### Change 4 — In `STEP_KEYS` array, add as first entry (before the productName array):

```ts
  // Step 1 – Data Agreement & Access (only Section A + B validated; Section C sub-fields are conditional)
  ["refreshFrequencyAgreement", "availabilitySLA", "ownedEndUserConsumptionPattern", "usageGuidance", "deprecationPolicy", "highPerformantAgreement", "sourceManagedConsumptionPatterns", "overallClassification", "retentionRequirement", "regulatoryDisclaimers", "dataProductCID", "ownEntitlementProcessAgreement"],
```

### Change 5 — Update comment: `// Only Section A keys are validated for Step 7` → `Step 8`

### Change 6 — In `SCHEMAS` array, add `step9Schema,` at the end (9 entries total: step1Schema through step9Schema)

### Change 7 — Update JSDoc: `(1–8)` → `(1–9)`

**Note:** Only 12 of the 20 fields are validated. The 8 conditional entitlement sub-fields (Section C) are NOT in the schema since they show/hide based on the Yes/No answer.

-----

## Task 4 — Add new fields to `src/hooks/useFormPersistence.ts`

### What changed

Added 20 new Step 1 field keys to the `STATE_KEYS` array so they get saved to and restored from localStorage. Renumbered all step comments.

### Exact change — Replace the entire `STATE_KEYS` array

**Find (the whole array from `const STATE_KEYS` to closing `];`).**

**Replace with:**

```ts
const STATE_KEYS: (keyof WizardFormState)[] = [
  // Step 1 – Data Agreement & Access (NEW)
  "refreshFrequencyAgreement", "availabilitySLA", "ownedEndUserConsumptionPattern",
  "usageGuidance", "deprecationPolicy", "highPerformantAgreement",
  "sourceManagedConsumptionPatterns", "overallClassification", "retentionRequirement",
  "regulatoryDisclaimers", "dataProductCID", "ownEntitlementProcessAgreement",
  "entitlementInfoBBS", "alignToDPASEntitlementModel", "alignToExistingEntitlementStructure",
  "eligibleRolesGroups", "approvalRules", "fieldLevelRestrictions",
  "crossDomainSharing", "janusBBSMaintainerGroup",
  // Step 2 – Product Identity & Purpose
  "productName", "productType", "businessDomain", "shortDescription", "longDescription",
  "productVersion", "dataCategory", "businessPurpose",
  // Step 3 – Ownership & Support
  "owner", "backupOwner", "itLead", "dataSteward", "supportEmail",
  "businessUnit", "costCenter",
  // Step 4 – Source & Entitlements
  "sourceUrl", "environment", "sourceSystem", "sourceDatabase", "sourceSchema",
  "connectionType", "refreshSchedule", "cidClassification", "accessControl",
  "highPerformant", "ownEntitlementProcess", "entitlementGroup", "entitlementApprover",
  // Step 5 – Semantics & Data Quality
  "businessEntity", "glossaryTerms", "ontologyMapping", "semanticTags", "domainModel",
  "dataScope", "refreshFrequency", "qualityRules", "completenessTarget", "accuracyTarget",
  "timelinessTarget", "consistencyRules", "uniquenessRules", "validityRules", "fitnessScore",
  // Step 6 – Data Contract & Access
  "dataRetention", "accessAgreement", "slaUptime", "supportHours", "responseTime",
  "escalationPath", "changeNotificationDays", "breakingChangesPolicy", "consumerOnboarding",
  "endOfLifeDate", "deprecationPlan",
  // Step 7 – Lineage & Transformations
  "upstreamSources", "transformationLogic", "upstreamOwner", "dataFlowDiagram",
  "processingEngine", "latencyRequirement", "errorHandling", "monitoringEndpoint", "alertingRules",
  // Step 8 – Compliance & AI Readiness
  "dataClassification", "containsPII", "regulatoryRequirements", "piiHandlingProcedure",
  "dpaRequired", "mlFeatureStore", "aiModelUsage", "embeddingSupport",
  "biasAssessment", "explainabilityRequirement", "modelGovernance",
  // Step 9 – Attestation & Submit
  "attestAccuracy", "attestOwnership", "attestPolicyCompliance",
  // Metadata
  "dpasResourceId", "dpasResourceIri", "dpasEtag", "metaqSource", "lastSavedAt",
];
```

-----

## Task 5 — Update `src/App.tsx` (step names array from 8 → 9)

### What changed

Added `'Data Agreement & Access'` as the first entry in the `stepNames` array.

### Exact change

**Find:**

```ts
const stepNames = [
  'Product Identity & Purpose',
  'Ownership & Support',
  'Source & Entitlements',
  'Semantics & Data Quality',
  'Data Contract & Access',
  'Lineage & Transformations',
  'Compliance & AI Readiness',
  'Attestation & Submit',
];
```

**Replace with:**

```ts
const stepNames = [
  'Data Agreement & Access',
  'Product Identity & Purpose',
  'Ownership & Support',
  'Source & Entitlements',
  'Semantics & Data Quality',
  'Data Contract & Access',
  'Lineage & Transformations',
  'Compliance & AI Readiness',
  'Attestation & Submit',
];
```

-----

## Task 6 — Update `src/components/Sidebar.tsx` (step list from 8 → 9)

### What changed

Added new entry to `STEP_META` array and updated the bottom status text from `of 8` to `of 9`.

### Change 1 — In `STEP_META` array

**Find:**

```ts
const STEP_META = [
  { title: "Identity & Purpose", sub: "Name, type, domain, version", required: true },
```

**Replace with:**

```ts
const STEP_META = [
  { title: "Agreement & Access", sub: "SLA, classification & entitlements", required: true },
  { title: "Identity & Purpose", sub: "Name, type, domain, version", required: true },
```

### Change 2 — Near the bottom of the file

**Find:**

```
Step {currentStep} of 8 · {8 - currentStep} remaining
```

**Replace with:**

```
Step {currentStep} of 9 · {9 - currentStep} remaining
```

-----

## Task 7 — Update `src/components/RegistrationForm.tsx` — Step numbers & stepInfo array

### What changed

Updated 12 places where step count `8` was hardcoded to `9`. Added `ClipboardList` icon import. Added new entry to `stepInfo`. Updated `displayStep` rendering block. Moved “Import from Catalog” from step 1 to step 2. Updated attestation completion check.

### Change 1 — Icon import

**Find:**

```ts
  Lightbulb, Info, Loader2, Lock
} from 'lucide-react';
```

**Replace with:**

```ts
  Lightbulb, Info, Loader2, Lock, ClipboardList
} from 'lucide-react';
```

### Change 2 — `handleContinue` function

**Find:** `if (currentStep >= 8 || isSaving) return;`
**Replace with:** `if (currentStep >= 9 || isSaving) return;`

### Change 3 — `handleSubmit` function

**Find:**

```ts
    // Validate Step 8 (attestations)
    const validation = validateStep(8, store);
```

**Replace with:**

```ts
    // Validate Step 9 (attestations)
    const validation = validateStep(9, store);
```

### Change 4 — Keyboard handler

**Find:** `if (currentStep < 8) handleContinue();`
**Replace with:** `if (currentStep < 9) handleContinue();`

### Change 5 — Replace the entire `stepInfo` array

```ts
  const stepInfo = [
    { number: 1, title: 'Data Agreement & Access', icon: ClipboardList, description: 'Define data frequency, SLA, classification, retention, and entitlement model.' },
    { number: 2, title: 'Product Identity & Purpose', icon: Package, description: 'Define the core identity of your data product — name, type, domain, and business purpose.' },
    { number: 3, title: 'Ownership & Support', icon: Users, description: 'Assign ownership, stewardship, and organizational accountability.' },
    { number: 4, title: 'Source & Entitlements', icon: Link2, description: 'Configure source systems, connections, and access entitlements.' },
    { number: 5, title: 'Semantics & Data Quality', icon: Database, description: 'Map business metadata, quality rules, and fitness targets.' },
    { number: 6, title: 'Data Contract & Access', icon: FileText, description: 'Establish SLAs, retention policies, and consumer onboarding.' },
    { number: 7, title: 'Lineage & Transformations', icon: GitBranch, description: 'Document upstream sources, transformation logic, and data flows.' },
    { number: 8, title: 'Compliance & AI Readiness', icon: Shield, description: 'Classify sensitivity, map regulations, and configure AI capabilities.' },
    { number: 9, title: 'Attestation & Submit', icon: Send, description: 'Review all entries, attest, and submit for governance review.' },
  ];
```

### Change 6 — Step counter display

**Find:** `{currentStep} / 8`
**Replace with:** `{currentStep} / 9`

### Change 7 — Progress bar

**Find:** `(currentStep / 8) * 100`
**Replace with:** `(currentStep / 9) * 100`

### Change 8 — Import from Catalog condition

**Find:** `{currentStep === 1 && (`
**Replace with:** `{currentStep === 2 && (`

### Change 9 — Replace the displayStep rendering block

**Find:**

```ts
          {displayStep === 1 && <Step1 errors={errors} onChange={handleFieldChange} titleLocked={!!store.dpasResourceId} />}
          {displayStep === 2 && <Step2 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 3 && <Step3 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 4 && <Step4 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 5 && <Step5 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 6 && <Step6 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 7 && <Step7 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 8 && <Step8 errors={errors} onChange={handleFieldChange} />}
```

**Replace with:**

```ts
          {displayStep === 1 && <StepAgreement errors={errors} onChange={handleFieldChange} />}
          {displayStep === 2 && <Step1 errors={errors} onChange={handleFieldChange} titleLocked={!!store.dpasResourceId} />}
          {displayStep === 3 && <Step2 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 4 && <Step3 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 5 && <Step4 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 6 && <Step5 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 7 && <Step6 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 8 && <Step7 errors={errors} onChange={handleFieldChange} />}
          {displayStep === 9 && <Step8 errors={errors} onChange={handleFieldChange} />}
```

### Change 10 — Navigation

**Find:** `{currentStep < 8 ? (`
**Replace with:** `{currentStep < 9 ? (`

### Change 11 — In Step8 function (attestation)

**Find:**

```ts
  // Compute completion across steps 1-7
  const requiredSteps = stepCompletion.slice(0, 7);
```

**Replace with:**

```ts
  // Compute completion across steps 1-8
  const requiredSteps = stepCompletion.slice(0, 8);
```

### Change 12 — Same area text

**Find:** `steps 1–7. Go back to fill missing fields.`
**Replace with:** `steps 1–8. Go back to fill missing fields.`

-----

## Task 8 — Add the `StepAgreement` form component in `src/components/RegistrationForm.tsx`

### What changed

Added the entire `StepAgreement` function component (~170 lines) right ABOVE the existing `function Step1`. This is the UI for the new Step 1 with 3 sections and conditional entitlement logic.

### Exact change

**Find this comment:**

```ts
/* ─── Step 1: Product Identity & Purpose (8 fields) ──────── */
```

**Replace it with the entire block below (everything from the new comment down to and including the new Step 2 comment):**

```tsx
/* ─── NEW Step 1: Data Agreement & Access (20 fields, 3 sections) ─ */

function StepAgreement({ errors, onChange }: StepProps) {
  const s = useRegistrationStore();
  const showEntitlementYes = s.ownEntitlementProcessAgreement === 'Yes';
  const showEntitlementNo = s.ownEntitlementProcessAgreement === 'No';

  return (
    <div>
      {/* Section A: Data Agreement & Functionality */}
      <SectionHeader title="Section A: Data Agreement & Functionality" description="Consumer-facing data availability, SLA, and consumption configuration." />
      <HelpTip>These fields define how downstream consumers will discover and use this data product.</HelpTip>
      <div className="grid grid-cols-2 gap-x-6 gap-y-5 mb-8">
        <div>
          <label className={labelClass}>How Frequently is the Data Updated? <span className="text-[#e60028]">*</span></label>
          <select value={s.refreshFrequencyAgreement} onChange={e => onChange('refreshFrequencyAgreement', e.target.value)} className={fieldClass(errors, 'refreshFrequencyAgreement')}>
            <option value="Real-time">Real-time</option>
            <option value="Hourly">Hourly</option>
            <option value="Daily">Daily</option>
            <option value="Weekly">Weekly</option>
          </select>
          <FieldError message={errors.refreshFrequencyAgreement} />
        </div>
        <div>
          <label className={labelClass}>Availability SLA <span className="text-[#e60028]">*</span></label>
          <select value={s.availabilitySLA} onChange={e => onChange('availabilitySLA', e.target.value)} className={fieldClass(errors, 'availabilitySLA')}>
            <option value="99.9%">99.9%</option>
            <option value="99.5%">99.5%</option>
            <option value="99%">99%</option>
            <option value="95%">95%</option>
          </select>
          <FieldError message={errors.availabilitySLA} />
        </div>
        <div className="col-span-2">
          <label className={labelClass}>Owned End User Consumption Pattern <span className="text-[#e60028]">*</span></label>
          <textarea rows={2} value={s.ownedEndUserConsumptionPattern} onChange={e => onChange('ownedEndUserConsumptionPattern', e.target.value)} placeholder="Describe the consumer pattern in house..." className={`${fieldClass(errors, 'ownedEndUserConsumptionPattern')} resize-y`} />
          <FieldError message={errors.ownedEndUserConsumptionPattern} />
        </div>
        <div>
          <label className={labelClass}>Usage Guidance <span className="text-[#e60028]">*</span></label>
          <input type="text" value={s.usageGuidance} onChange={e => onChange('usageGuidance', e.target.value)} placeholder="e.g. In House vs Central Access" className={fieldClass(errors, 'usageGuidance')} />
          <FieldError message={errors.usageGuidance} />
        </div>
        <div>
          <label className={labelClass}>Deprecation Policy (Record Mgmt) <span className="text-[#e60028]">*</span></label>
          <input type="text" value={s.deprecationPolicy} onChange={e => onChange('deprecationPolicy', e.target.value)} placeholder="e.g. Archive after 5 years" className={fieldClass(errors, 'deprecationPolicy')} />
          <FieldError message={errors.deprecationPolicy} />
        </div>
        <div>
          <label className={labelClass}>High Performant <span className="text-[#e60028]">*</span></label>
          <select value={s.highPerformantAgreement} onChange={e => onChange('highPerformantAgreement', e.target.value)} className={fieldClass(errors, 'highPerformantAgreement')}>
            <option value="No">No</option>
            <option value="Yes">Yes</option>
          </select>
          <FieldError message={errors.highPerformantAgreement} />
        </div>
        <div className="col-span-2">
          <label className={labelClass}>Source Managed Consumption Patterns <span className="text-[#e60028]">*</span></label>
          <textarea rows={2} value={s.sourceManagedConsumptionPatterns} onChange={e => onChange('sourceManagedConsumptionPatterns', e.target.value)} placeholder="Describe source-managed consumption patterns..." className={`${fieldClass(errors, 'sourceManagedConsumptionPatterns')} resize-y`} />
          <FieldError message={errors.sourceManagedConsumptionPatterns} />
        </div>
      </div>

      {/* Section B: Sensitivity, Policy & Compliance */}
      <SectionHeader title="Section B: Sensitivity, Policy & Compliance" description="Classification, retention, and regulatory requirements." />
      <div className="grid grid-cols-2 gap-x-6 gap-y-5 mb-8">
        <div>
          <label className={labelClass}>Overall Classification <span className="text-[#e60028]">*</span></label>
          <select value={s.overallClassification} onChange={e => onChange('overallClassification', e.target.value)} className={fieldClass(errors, 'overallClassification')}>
            <option value="Public">Public</option>
            <option value="Internal">Internal</option>
            <option value="Confidential">Confidential</option>
            <option value="Restricted">Restricted</option>
          </select>
          <FieldError message={errors.overallClassification} />
        </div>
        <div>
          <label className={labelClass}>Retention Requirement <span className="text-[#e60028]">*</span></label>
          <select value={s.retentionRequirement} onChange={e => onChange('retentionRequirement', e.target.value)} className={fieldClass(errors, 'retentionRequirement')}>
            <option value="7 years">7 years</option>
            <option value="5 years">5 years</option>
            <option value="3 years">3 years</option>
            <option value="1 year">1 year</option>
          </select>
          <FieldError message={errors.retentionRequirement} />
        </div>
        <div className="col-span-2">
          <label className={labelClass}>Regulatory or Legal Disclaimers <span className="text-[#e60028]">*</span></label>
          <textarea rows={2} value={s.regulatoryDisclaimers} onChange={e => onChange('regulatoryDisclaimers', e.target.value)} placeholder="Capture regulatory frameworks (GDPR, SOX, etc.)..." className={`${fieldClass(errors, 'regulatoryDisclaimers')} resize-y`} />
          <FieldError message={errors.regulatoryDisclaimers} />
        </div>
        <div className="col-span-2">
          <label className={labelClass}>Data Product CID <span className="text-[#e60028]">*</span></label>
          <input type="text" value={s.dataProductCID} onChange={e => onChange('dataProductCID', e.target.value)} placeholder="e.g. CID-00123" className={fieldClass(errors, 'dataProductCID')} />
          <FieldError message={errors.dataProductCID} />
        </div>
      </div>

      {/* Section C: Entitlements */}
      <SectionHeader title="Section C: Entitlements" description="Define the access entitlement model for this data product." />
      <InfoTip>If you have your own entitlement process, enter existing BBS information. Otherwise, configure alignment to the DPAS entitlement model below.</InfoTip>
      <div className="grid grid-cols-2 gap-x-6 gap-y-5">
        <div className="col-span-2">
          <label className={labelClass}>Do you have your own entitlement process? <span className="text-[#e60028]">*</span></label>
          <select value={s.ownEntitlementProcessAgreement} onChange={e => onChange('ownEntitlementProcessAgreement', e.target.value)} className={fieldClass(errors, 'ownEntitlementProcessAgreement')}>
            <option value="No">No</option>
            <option value="Yes">Yes</option>
          </select>
          <FieldError message={errors.ownEntitlementProcessAgreement} />
        </div>

        {/* Show only if YES */}
        {showEntitlementYes && (
          <div className="col-span-2">
            <label className={labelClass}>Enter Entitlement Information (BBS Existing)</label>
            <textarea rows={2} value={s.entitlementInfoBBS} onChange={e => onChange('entitlementInfoBBS', e.target.value)} placeholder="Enter your existing BBS entitlement details..." className={`${inputOk} resize-y`} />
          </div>
        )}

        {/* Show only if NO */}
        {showEntitlementNo && (
          <>
            <div className="col-span-2">
              <label className={labelClass}>Align to DPAS Entitlement Model?</label>
              <select value={s.alignToDPASEntitlementModel} onChange={e => onChange('alignToDPASEntitlementModel', e.target.value)} className={inputOk}>
                <option value="">Select...</option>
                <option value="Yes">Yes</option>
                <option value="No">No</option>
              </select>
            </div>
            <div className="col-span-2">
              <label className={labelClass}>Align to Existing Entitlement Structure?</label>
              <select value={s.alignToExistingEntitlementStructure} onChange={e => onChange('alignToExistingEntitlementStructure', e.target.value)} className={inputOk}>
                <option value="">Select...</option>
                <option value="Yes">Yes</option>
                <option value="No">No</option>
              </select>
            </div>
            <div className="col-span-2">
              <label className={labelClass}>Eligible Roles / Groups</label>
              <input type="text" value={s.eligibleRolesGroups} onChange={e => onChange('eligibleRolesGroups', e.target.value)} placeholder="e.g. WMA-Data-Consumers, Analysts" className={inputOk} />
            </div>
            <div className="col-span-2">
              <label className={labelClass}>Approval Rules</label>
              <textarea rows={2} value={s.approvalRules} onChange={e => onChange('approvalRules', e.target.value)} placeholder="Describe approval workflow..." className={`${inputOk} resize-y`} />
            </div>
            <div className="col-span-2">
              <label className={labelClass}>Field-Level Restrictions / Masking</label>
              <textarea rows={2} value={s.fieldLevelRestrictions} onChange={e => onChange('fieldLevelRestrictions', e.target.value)} placeholder="Describe column-level masking or restrictions..." className={`${inputOk} resize-y`} />
            </div>
            <div>
              <label className={labelClass}>Cross-Domain Sharing</label>
              <select value={s.crossDomainSharing} onChange={e => onChange('crossDomainSharing', e.target.value)} className={inputOk}>
                <option value="">Select...</option>
                <option value="Allowed">Allowed</option>
                <option value="Restricted">Restricted</option>
                <option value="Not Allowed">Not Allowed</option>
              </select>
            </div>
            <div>
              <label className={labelClass}>Janus BBS Maintainer Group</label>
              <input type="text" value={s.janusBBSMaintainerGroup} onChange={e => onChange('janusBBSMaintainerGroup', e.target.value)} placeholder="e.g. Janus-WMA-Maintainers" className={inputOk} />
            </div>
          </>
        )}
      </div>
    </div>
  );
}

/* ─── Step 2: Product Identity & Purpose (8 fields) ──────── */
```

### Conditional logic explained

- `ownEntitlementProcessAgreement === 'Yes'` → Shows 1 field: `entitlementInfoBBS` (BBS textarea)
- `ownEntitlementProcessAgreement === 'No'` → Shows 8 fields: DPAS alignment, roles, approval rules, masking, cross-domain, Janus group
- Neither (shouldn’t happen with default “No”) → Only the main Yes/No question visible

-----

## Task 9 — Update `src/components/SummaryPanel.tsx` (add new Step 1 section)

### What changed

Added `ClipboardList` icon import and a new “Agreement & Access” section as the first entry in the `sections` array showing 12 key fields.

### Change 1 — Icon import

**Find:**

```ts
import { ChevronRight, ChevronLeft, Package, Users, Link2, Database, FileText, GitBranch, Shield, Send } from 'lucide-react';
```

**Replace with:**

```ts
import { ChevronRight, ChevronLeft, ClipboardList, Package, Users, Link2, Database, FileText, GitBranch, Shield, Send } from 'lucide-react';
```

### Change 2 — In the `sections` array

**Find:**

```ts
  const sections = [
    {
      icon: Package,
      title: 'Identity & Purpose',
```

**Replace with:**

```ts
  const sections = [
    {
      icon: ClipboardList,
      title: 'Agreement & Access',
      fields: [
        { label: 'Frequency', value: v(s.refreshFrequencyAgreement) },
        { label: 'Avail. SLA', value: v(s.availabilitySLA) },
        { label: 'Consumption', value: v(s.ownedEndUserConsumptionPattern) },
        { label: 'Usage Guide', value: v(s.usageGuidance) },
        { label: 'Deprecation', value: v(s.deprecationPolicy) },
        { label: 'High Perf.', value: v(s.highPerformantAgreement) },
        { label: 'Src Mgd Patterns', value: v(s.sourceManagedConsumptionPatterns) },
        { label: 'Classification', value: v(s.overallClassification) },
        { label: 'Retention', value: v(s.retentionRequirement) },
        { label: 'Reg. Disclaimers', value: v(s.regulatoryDisclaimers) },
        { label: 'Product CID', value: v(s.dataProductCID) },
        { label: 'Own Entitle.', value: v(s.ownEntitlementProcessAgreement) },
      ],
    },
    {
      icon: Package,
      title: 'Identity & Purpose',
```

-----

## Testing Checklist

After applying all changes, run `npm run dev` and verify:

1. **Clear old localStorage** — DevTools → Application → Local Storage → delete `wma-registration-draft`
1. **Step 1 loads as “Data Agreement & Access”** — 3 sections visible: Functionality, Sensitivity, Entitlements
1. **Section C conditional logic** — “Yes” shows BBS field, “No” shows 8 DPAS alignment fields
1. **Sidebar** — 9 steps, first is “Agreement & Access”, bottom says “Step 1 of 9”
1. **Progress bar** — fills based on /9
1. **Fill required fields → Save & Continue** — advances to Step 2 (Product Identity)
1. **“Import from Catalog”** — appears on Step 2 only
1. **Summary panel** — first section is “Agreement & Access” with 12 fields
1. **Step 9 (Attestation)** — shows “steps 1–8” completion text
1. **Ctrl+Enter** shortcut works throughout
