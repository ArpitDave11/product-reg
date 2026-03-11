# WMA Data Product Registration — CODE STYLE BIBLE
## ⛔ MANDATORY — Read Before Writing ANY Code

**This document extracts the EXACT patterns from our existing codebase.**
**Every new line of code MUST use these patterns. No exceptions. No improvising.**

> If your code doesn't look like it belongs in the current `RegistrationForm.tsx`, it's wrong. Rewrite it.

---

# 1. CSS CLASS CONSTANTS — USE THESE, NEVER INVENT YOUR OWN

These are defined at the top of `RegistrationForm.tsx`. **Every new step MUST reuse them.**

```tsx
// ── THESE ARE ALREADY DEFINED — IMPORT/REUSE THEM ──

const inputBase = "w-full border rounded-lg px-3.5 py-2.5 text-[13px] text-[#111] bg-white focus:ring-2 focus:outline-none transition-all placeholder:text-[#c0c0c0]";

const inputOk = `${inputBase} border-[#e5e7eb] hover:border-[#c0c0c0] focus:border-[#e60028] focus:ring-[#e60028]/10`;

const inputErr = `${inputBase} border-[#ef4444] hover:border-[#dc2626] focus:border-[#ef4444] focus:ring-[#ef4444]/10`;

function fieldClass(errors: Record<string, string>, key: string) {
  return errors[key] ? inputErr : inputOk;
}

const labelClass = "block text-[13px] font-medium text-[#374151] mb-1.5";
```

### Rules:
- ✅ `className={fieldClass(errors, 'myField')}` — for ALL inputs, selects, textareas
- ✅ `className={labelClass}` — for ALL labels
- ❌ NEVER write `className="border border-gray-300 rounded p-2"` — this is wrong
- ❌ NEVER write `className="text-sm font-medium"` — use the exact constants
- ❌ NEVER use Tailwind prose classes like `text-sm`, `text-lg`, `font-bold` — use pixel sizes: `text-[13px]`, `text-[12px]`, `text-[11px]`

---

# 2. FIELD PATTERN — THE SACRED STRUCTURE

**Every single form field follows this EXACT template. No variation.**

### Text Input
```tsx
<div>
  <label className={labelClass}>
    Field Label <span className="text-[#e60028]">*</span>
  </label>
  <input
    type="text"
    value={s.fieldName}
    onChange={e => onChange('fieldName', e.target.value)}
    placeholder="e.g. Example value"
    className={fieldClass(errors, 'fieldName')}
  />
  <FieldError message={errors.fieldName} />
</div>
```

### Textarea
```tsx
<div className="col-span-2">
  <label className={labelClass}>
    Field Label <span className="text-[#e60028]">*</span>
  </label>
  <textarea
    rows={2}
    value={s.fieldName}
    onChange={e => onChange('fieldName', e.target.value)}
    placeholder="Describe something..."
    className={`${fieldClass(errors, 'fieldName')} resize-y`}
  />
  <FieldError message={errors.fieldName} />
</div>
```

### Select/Dropdown
```tsx
<div>
  <label className={labelClass}>
    Field Label <span className="text-[#e60028]">*</span>
  </label>
  <select
    value={s.fieldName}
    onChange={e => onChange('fieldName', e.target.value)}
    className={fieldClass(errors, 'fieldName')}
  >
    <option value="Option1">Option 1</option>
    <option value="Option2">Option 2</option>
  </select>
  <FieldError message={errors.fieldName} />
</div>
```

### Rules:
- ✅ Always wrap in `<div>` (or `<div className="col-span-2">` for full-width)
- ✅ Always `<label>` THEN `<input>` THEN `<FieldError>`  — this exact order
- ✅ Required asterisk: `<span className="text-[#e60028]">*</span>`
- ✅ Store access: `const s = useRegistrationStore();`
- ✅ onChange pattern: `onChange={e => onChange('fieldName', e.target.value)}`
- ❌ NEVER use `<Label>` from shadcn — use plain `<label>`
- ❌ NEVER use `<Input>` from shadcn — use plain `<input>`
- ❌ NEVER use `<Select>` from shadcn for basic dropdowns — use plain `<select>`
- ❌ NEVER skip `<FieldError>` — even if the field is optional

---

# 3. GRID LAYOUT — THE SACRED GRID

**Every step's form content uses this exact grid:**

```tsx
<div className="grid grid-cols-2 gap-x-6 gap-y-5">
  {/* fields go here */}
</div>
```

### Rules:
- ✅ `grid-cols-2` — always 2 columns
- ✅ `gap-x-6` — horizontal gap 24px
- ✅ `gap-y-5` — vertical gap 20px
- ✅ Full-width fields use `<div className="col-span-2">`
- ❌ NEVER use `grid-cols-1`, `grid-cols-3`, `flex`, or any other layout for form fields
- ❌ NEVER use `space-y-4`, `space-y-6`, `mb-4` between fields — the grid `gap` handles it
- ❌ NEVER add extra margin/padding between fields — the grid is the ONLY spacer

---

# 4. SECTION STRUCTURE — HOW A STEP LOOKS

**Every step function follows this exact structure:**

```tsx
function StepN({ errors, onChange }: StepProps) {
  const s = useRegistrationStore();
  return (
    <div>
      {/* Section header */}
      <SectionHeader
        title="Section Title"
        description="One line describing what this section captures."
      />

      {/* Optional tip */}
      <HelpTip>Helpful guidance text for the user.</HelpTip>
      {/* OR */}
      <InfoTip>Informational text in blue.</InfoTip>

      {/* Form grid */}
      <div className="grid grid-cols-2 gap-x-6 gap-y-5">
        {/* fields */}
      </div>
    </div>
  );
}
```

### If a step has TWO sections (like current Step 3):
```tsx
<div>
  <SectionHeader title="Section A: First Part" description="..." />
  <HelpTip>...</HelpTip>
  <div className="grid grid-cols-2 gap-x-6 gap-y-5 mb-8">
    {/* Section A fields */}
  </div>

  <SectionHeader title="Section B: Second Part" description="..." />
  <div className="grid grid-cols-2 gap-x-6 gap-y-5">
    {/* Section B fields */}
  </div>
</div>
```

### Rules:
- ✅ `mb-8` between sections (if there are multiple)
- ✅ Each section gets its own `SectionHeader`
- ❌ NEVER omit `SectionHeader` — every section needs one
- ❌ NEVER add `<h2>`, `<h3>` tags directly — use `SectionHeader` component

---

# 5. SHARED HELPER COMPONENTS — USE THESE

**These already exist. DO NOT recreate them. DO NOT modify them.**

### SectionHeader
```tsx
function SectionHeader({ title, description }: { title: string; description?: string }) {
  return (
    <div className="mb-5 pb-3 border-b border-[#f0f0f0]">
      <h3 className="text-[14px] font-semibold text-[#111]">{title}</h3>
      {description && <p className="text-[12px] text-[#9ca3af] mt-1">{description}</p>}
    </div>
  );
}
```

### HelpTip (amber/yellow, lightbulb icon)
```tsx
function HelpTip({ children }: { children: ReactNode }) {
  return (
    <div className="flex items-start gap-2.5 p-3.5 rounded-xl bg-[#f8f9fa] border border-[#f0f0f0] mb-6">
      <Lightbulb className="w-4 h-4 text-[#f59e0b] flex-shrink-0 mt-0.5" />
      <p className="text-[12px] text-[#6b7280] leading-relaxed">{children}</p>
    </div>
  );
}
```

### InfoTip (blue, info icon)
```tsx
function InfoTip({ children }: { children: ReactNode }) {
  return (
    <div className="flex items-start gap-2.5 p-3.5 rounded-xl bg-[#f0f9ff] border border-[#e0f2fe] mb-6">
      <Info className="w-4 h-4 text-[#0284c7] flex-shrink-0 mt-0.5" />
      <p className="text-[12px] text-[#0369a1] leading-relaxed">{children}</p>
    </div>
  );
}
```

### FieldError
```tsx
function FieldError({ message }: { message?: string }) {
  if (!message) return null;
  return <p className="text-[11px] text-[#ef4444] mt-1">{message}</p>;
}
```

---

# 6. OUTER WRAPPER — THE FORM CARD

**The RegistrationForm wraps ALL step content in this card. Steps do NOT create their own outer card.**

```tsx
{/* This is in RegistrationForm, NOT in individual steps */}
<div className="bg-white rounded-2xl border border-[#e5e7eb] shadow-[0_1px_2px_rgba(0,0,0,0.03),0_2px_8px_rgba(0,0,0,0.04)] p-8 lg:p-10">
  {/* Step content renders here */}
</div>
```

### Rules:
- ✅ Steps render FLAT content — just `<div>`, `<SectionHeader>`, `<div className="grid ...">`, fields
- ❌ NEVER add an outer card/border/shadow INSIDE a step — it's already wrapped
- ❌ NEVER add `p-6`, `p-8` padding inside a step — the wrapper handles it
- ❌ NEVER add `max-w-*` inside a step — the wrapper sets `max-w-[840px]`

---

# 7. STEP HEADER — ABOVE THE FORM CARD

**This is rendered by RegistrationForm, NOT by individual steps:**

```tsx
<div className="mb-6">
  <div className="flex items-start justify-between gap-4 mb-4">
    <div className="flex items-center gap-3.5">
      <div className="p-2.5 bg-[#fef2f2] rounded-xl border border-[#fecaca]">
        <StepIcon className="w-5 h-5 text-[#e60028]" />
      </div>
      <div>
        <h1 className="text-[20px] font-semibold text-[#111] tracking-tight">{current.title}</h1>
        <p className="text-[13px] text-[#6b7280] mt-1">{current.description}</p>
      </div>
    </div>
    <span className="text-[13px] text-[#b0b0b0] font-medium">{currentStep} / 10</span>
  </div>
  <div className="w-full bg-[#f0f0f0] rounded-full h-[5px] overflow-hidden">
    <div className="bg-[#e60028] h-full rounded-full transition-all duration-500 ease-out"
         style={{ width: `${(currentStep / 10) * 100}%` }} />
  </div>
</div>
```

### Rules:
- ✅ Step icon has red background: `bg-[#fef2f2] rounded-xl border border-[#fecaca]`
- ❌ NEVER put step title/icon INSIDE the step function — it's in the parent

---

# 8. NAVIGATION BAR — BOTTOM OF FORM CARD

**Already implemented. Do NOT touch.**

```tsx
<div className="flex justify-between items-center mt-10 pt-7 border-t border-[#f0f0f0]">
  {/* Back button */}
  <button className="px-4 py-2.5 rounded-lg text-[13px] font-medium ... border ...">
    <ChevronLeft className="w-3.5 h-3.5" /> Back
  </button>

  {/* Save & Continue OR Submit */}
  <button className="px-5 py-2.5 bg-[#e60028] text-white rounded-lg text-[13px] font-medium hover:bg-[#cc0024] ...">
    Save & Continue <ArrowRight className="w-3.5 h-3.5" />
  </button>
</div>
```

### Button Styles:
- **Primary (red):** `bg-[#e60028] text-white rounded-lg text-[13px] font-medium hover:bg-[#cc0024] shadow-[0_1px_3px_rgba(230,0,40,0.2)]`
- **Secondary (outline):** `bg-white text-[#374151] border border-[#e5e7eb] rounded-lg text-[13px] font-medium hover:border-[#9ca3af] hover:bg-[#f9fafb]`
- **Disabled:** add `disabled:opacity-60` and `cursor-not-allowed`
- **Loading:** `<Loader2 className="w-3.5 h-3.5 animate-spin" />` + "Saving..."

---

# 9. COLOR PALETTE — THE ONLY COLORS ALLOWED

```
PRIMARY RED:       #E60028  (buttons, accents, active states, required asterisks)
HOVER RED:         #CC0024  (button hover)
RED BACKGROUND:    #FEF2F2  (step icon bg, error bg)
RED BORDER:        #FECACA  (step icon border)
RED LIGHT BG:      #FFF5F5  (selected radio card)

BLACK:             #111111  (primary headings, titles)
DARK GRAY:         #374151  (label text, body text)
MEDIUM GRAY:       #6B7280  (descriptions, secondary text)
LIGHT GRAY:        #9CA3AF  (timestamps, hints, disabled text)
PLACEHOLDER GRAY:  #C0C0C0  (input placeholders, hover borders)
FAINT GRAY:        #B0B0B0  (step counter "3/10", very subtle text)

BORDER LIGHT:      #E8E8EC  (sidebar border, card borders)
BORDER NORMAL:     #E5E7EB  (input borders, card borders)
BORDER FAINT:      #F0F0F0  (section dividers, internal separators)

BG LIGHT:          #F9FAFB  (subtle background, section B cards)
BG LIGHTER:        #FAFAFA  (not-started step bg)
BG WHITE:          #FFFFFF  (cards, form background)
BG WARM:           #F8F9FA  (HelpTip background)

SUCCESS GREEN:     #22C55E  (completed state, submit button)
SUCCESS GREEN BG:  #F0FDF4  (completed step bg)
SUCCESS BORDER:    #DCFCE7  (completed step border)
SUCCESS TEXT:      #15803D  (completed text)
GREEN DARK:        #16A34A  (submit button hover)

WARNING AMBER:     #F59E0B  (in-progress state, lightbulb icon)
WARNING BG:        #FFFBEB  (warning banners)
WARNING BORDER:    #FEF3C7  (warning borders)
WARNING TEXT:      #92400E  (warning text)

INFO BLUE:         #0284C7  (info badges, read-only steps, catalog button)
INFO BG:           #F0F9FF  (InfoTip background, catalog button bg)
INFO BORDER:       #E0F2FE  (InfoTip border)
INFO TEXT:         #0369A1  (InfoTip body text)

ERROR RED:         #EF4444  (validation error borders)
ERROR HOVER:       #DC2626  (error border hover)
ERROR TEXT:        #EF4444  (error message text, at 11px)
```

### Rules:
- ✅ Use hex values: `text-[#374151]`, `border-[#e5e7eb]`, `bg-[#f9fafb]`
- ❌ NEVER use Tailwind color names: `text-gray-600`, `border-gray-200`, `bg-gray-50`
- ❌ NEVER use `red-600`, `green-500`, `blue-500` — use exact hex values
- ❌ NEVER invent new colors — if it's not in this list, it doesn't exist

---

# 10. TYPOGRAPHY — EXACT SIZES

```
Step Title:        text-[20px] font-semibold text-[#111] tracking-tight
Section Title:     text-[14px] font-semibold text-[#111]
Section Desc:      text-[12px] text-[#9ca3af]
Labels:            text-[13px] font-medium text-[#374151]
Body / Inputs:     text-[13px] text-[#111]
Descriptions:      text-[13px] text-[#6b7280]
Secondary text:    text-[12px] text-[#6b7280]
Hints / Help:      text-[12px] text-[#6b7280] leading-relaxed
Errors:            text-[11px] text-[#ef4444]
Timestamps:        text-[11px] text-[#9ca3af]
Badges/pills:      text-[10px] font-semibold uppercase tracking-wide
Step counter:      text-[13px] text-[#b0b0b0] font-medium
```

### Rules:
- ✅ Always use pixel sizes: `text-[13px]`, `text-[12px]`, `text-[11px]`, `text-[10px]`
- ❌ NEVER use `text-sm`, `text-base`, `text-lg`, `text-xs`
- ❌ NEVER use `font-bold` — use `font-semibold` for headings, `font-medium` for labels

---

# 11. ICON RULES

**All icons from `lucide-react`. Standard sizes:**

```
Step icon (in header):   w-5 h-5 text-[#e60028]
Icons inside buttons:    w-3.5 h-3.5
Icons inside tips:       w-4 h-4 text-[#f59e0b]  (HelpTip) or text-[#0284c7] (InfoTip)
Icons in sidebar steps:  w-3.5 h-3.5
Inline status icons:     w-3 h-3
```

### Rules:
- ✅ Icons are always Lucide React: `import { Package } from 'lucide-react'`
- ✅ Icon + text spacing: `flex items-center gap-2` or `gap-2.5`
- ❌ NEVER use `w-6 h-6` or larger icons inside form content
- ❌ NEVER import icons from other libraries

---

# 12. OPTIONAL SECTIONS — MUTED BACKGROUND PATTERN

**When a section is optional (like AI Readiness in current Step 7):**

```tsx
<div className="rounded-xl bg-[#f9fafb] border border-[#f0f0f0] p-6">
  <div className="flex items-center gap-2.5 mb-4 pb-3 border-b border-[#e8e8ec]">
    <h3 className="text-[14px] font-semibold text-[#111]">Section B: AI Readiness</h3>
    <span className="text-[10px] font-semibold text-[#0284c7] bg-[#f0f9ff] border border-[#e0f2fe] px-2 py-0.5 rounded-full uppercase tracking-wide">
      Optional
    </span>
  </div>
  <InfoTip>Configure only if this data product will be consumed by AI/ML pipelines.</InfoTip>
  <div className="grid grid-cols-2 gap-x-6 gap-y-5">
    {/* optional fields */}
  </div>
</div>
```

---

# 13. MODAL / DIALOG PATTERN

**For dataset dialog, port dialog, etc:**

```tsx
<div className="fixed inset-0 z-50 flex items-center justify-center">
  {/* Backdrop */}
  <div className="absolute inset-0 bg-black/40 backdrop-blur-sm" onClick={onClose} />

  {/* Modal */}
  <div className="relative w-full max-w-[560px] max-h-[85vh] bg-white rounded-2xl shadow-2xl flex flex-col overflow-hidden mx-4">
    {/* Header */}
    <div className="flex items-center justify-between px-6 py-4 border-b border-[#e8e8ec]">
      <div className="flex items-center gap-3">
        <div className="p-2 bg-[#f0f9ff] rounded-lg border border-[#e0f2fe]">
          <IconHere className="w-4 h-4 text-[#0284c7]" />
        </div>
        <div>
          <h2 className="text-[15px] font-semibold text-[#111]">Dialog Title</h2>
          <p className="text-[12px] text-[#9ca3af]">Subtitle</p>
        </div>
      </div>
      <button onClick={onClose} className="p-1.5 text-[#9ca3af] hover:text-[#374151] hover:bg-[#f3f4f6] rounded-lg transition-colors">
        <X className="w-4 h-4" />
      </button>
    </div>

    {/* Body — scrollable */}
    <div className="flex-1 overflow-y-auto px-6 py-4">
      <div className="grid grid-cols-2 gap-x-6 gap-y-5">
        {/* fields using same labelClass, fieldClass, FieldError pattern */}
      </div>
    </div>

    {/* Footer */}
    <div className="px-6 py-4 border-t border-[#f0f0f0] flex justify-end gap-3">
      <button onClick={onClose} className="px-4 py-2.5 bg-white text-[#374151] border border-[#e5e7eb] rounded-lg text-[13px] font-medium hover:bg-[#f9fafb]">
        Cancel
      </button>
      <button onClick={handleSave} className="px-5 py-2.5 bg-[#e60028] text-white rounded-lg text-[13px] font-medium hover:bg-[#cc0024] shadow-[0_1px_3px_rgba(230,0,40,0.2)]">
        Save
      </button>
    </div>
  </div>
</div>
```

This pattern is already used in `DatasetBrowser.tsx`. Copy its structure exactly.

---

# 14. BADGES & PILLS

```tsx
{/* CID Badge — Green */}
<span className="inline-flex items-center gap-1 text-[10px] font-semibold px-2 py-0.5 rounded-full bg-[#f0fdf4] text-[#15803d]">
  <span className="w-1.5 h-1.5 rounded-full bg-[#22c55e]" />
  Green
</span>

{/* Optional Badge — Blue */}
<span className="text-[10px] font-semibold text-[#0284c7] bg-[#f0f9ff] border border-[#e0f2fe] px-2 py-0.5 rounded-full uppercase tracking-wide">
  Optional
</span>

{/* Metadata Chip — Gray */}
<span className="text-[10px] px-2 py-0.5 rounded bg-[#f3f4f6] text-[#6b7280] border border-[#e5e7eb]">
  chip text
</span>
```

---

# 15. WHAT NEVER TO DO — COMMON MISTAKES

| ❌ WRONG | ✅ RIGHT | Why |
|----------|---------|-----|
| `className="text-sm text-gray-600"` | `className="text-[13px] text-[#6b7280]"` | We use pixel sizes and hex colors |
| `<Label>` from shadcn | `<label className={labelClass}>` | We use plain HTML elements |
| `<Input>` from shadcn | `<input className={fieldClass(errors, 'x')}>` | We use our own input styling |
| `className="border rounded p-2"` | `className={fieldClass(errors, 'x')}` | Always use the fieldClass helper |
| `className="space-y-4"` in form grids | `className="grid grid-cols-2 gap-x-6 gap-y-5"` | Grid is the only layout for fields |
| `<div className="bg-white rounded-xl p-6">` wrapping step | Nothing — parent handles the card | Steps render flat content |
| `className="font-bold text-lg"` | `className="text-[14px] font-semibold text-[#111]"` | Exact sizes, font-semibold not bold |
| Adding `<h2>Section</h2>` | `<SectionHeader title="Section" />` | Use the helper component |
| Using `red-600`, `green-500` colors | Using `#e60028`, `#22c55e` hex | We never use Tailwind named colors |
| Adding `mb-3` between fields | Let grid `gap-y-5` handle spacing | Grid handles all field spacing |
| Writing `style={{ ... }}` inline | Use Tailwind classes | Inline styles only for dynamic values like width% |
| Creating new wrapper divs with padding | Keep structure flat | Extra nesting ruins consistency |

---

# 16. CHECKLIST — BEFORE EVERY PR

Before submitting any code:

- [ ] Every input uses `fieldClass(errors, 'fieldName')` — NO custom input classes
- [ ] Every label uses `labelClass` — NO custom label styles
- [ ] Every field has `<FieldError message={errors.fieldName} />` after it
- [ ] Form layout uses `grid grid-cols-2 gap-x-6 gap-y-5` — NO flex/space-y
- [ ] Section starts with `<SectionHeader title="..." description="..." />`
- [ ] Colors are hex values `#E60028` — NO Tailwind color names like `red-600`
- [ ] Font sizes are pixel: `text-[13px]` — NO `text-sm`, `text-base`
- [ ] Icons are from `lucide-react` with `w-3.5 h-3.5` or `w-4 h-4`
- [ ] No outer card/border/padding inside step functions — parent handles it
- [ ] No shadcn `<Input>`, `<Label>`, `<Select>` for basic form fields
- [ ] Step function signature: `function StepN({ errors, onChange }: StepProps)`
- [ ] Store access: `const s = useRegistrationStore();`
- [ ] Required asterisk: `<span className="text-[#e60028]">*</span>`
- [ ] Modal follows `DatasetBrowser.tsx` structure exactly
- [ ] No new colors invented — only colors from Section 9
- [ ] Textarea has `resize-y` class
- [ ] Full-width fields use `col-span-2`

---

*If something isn't covered here, look at how the EXISTING code does it and copy that pattern exactly.*
