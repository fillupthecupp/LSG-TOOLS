# Repo A — Porting Guide for Merged Product
# Retail Deal Pipeline & Screener

> **Purpose:** Migration reference for porting Repo A's intelligence layer into the merged product (Repo B as structural base).
> **Scope:** Documentation only — no code changes.
> **Assumptions:** Repo B is React / Vercel / Supabase. Merged product is a single deployed app.
> **Authority:** Where this doc conflicts with Repo B conventions, escalate; do not resolve silently.
> **Last updated:** April 16, 2026

---

## 1. Executive Summary

Repo A is the **intelligence and persistence layer** of the merged product. It is not the application shell.

Its contribution is specifically:
- The two AI extraction prompts (Reader + Standardizer) and the schema v2.0 template embedded in the Standardizer prompt
- The AI screener prompt and deal packaging logic that feeds it
- The Supabase write/read/patch patterns and their safety guards
- The one-pager rendering logic (section-by-section deal presentation)
- A suite of formatting utilities for CRE financial data
- The schema validation and metadata override logic

Repo A's single-file HTML architecture, inline CSS, and DOM manipulation patterns are **not** part of its contribution. These were constraint-driven delivery vehicles, not design decisions to carry forward.

The merged product should treat Repo A as the source of truth for:
- What fields exist in a deal record
- How those fields are written to and read from Supabase
- How the AI pipeline is structured and what prompts it uses
- What the IC hurdle thresholds are and how the screener evaluates against them

---

## 2. Must-Port Assets

Each asset below is named by source file and function/section. Port these before anything else.

---

### 2.1 Two-Agent Extraction Prompts

**Source:** `lsg_ingest.html`, `READER_SYSTEM` and `STANDARDIZER_SYSTEM` constants (lines ~919–1161)

**What they are:**
- `READER_SYSTEM` — Agent 1 system prompt. Instructs the model to read all source documents (OM, T12, rent roll, Excel model) and produce a comprehensive narrative summary. Critically, it imposes **no schema** — it reads faithfully and reports everything it finds.
- `STANDARDIZER_SYSTEM` — Agent 2 system prompt. Takes the narrative from Agent 1 and maps it to the full LSG schema v2.0 JSON template. Detects conflicts between source documents (e.g., OM vs. T12 NOI discrepancy) and populates `conflict_flags[]`. Returns only valid JSON.

**Why they must be ported verbatim:**
The prompts have been tuned for retail CRE extraction accuracy. The schema template embedded in the Standardizer prompt is the single most important artifact in the entire repo — it defines ~200 fields across 25 top-level objects. Do not rewrite, summarize, or trim these prompts without testing.

**Target in merged product:** A constants/prompts file, e.g., `lib/prompts/extractionPrompts.ts`. Export both as string constants. Do not inline them in a component or API route.

---

### 2.2 AI Screener Prompt

**Source:** `lsg_ingest.html`, `SCREEN_SYSTEM` constant (lines 1872–1901)

**What it is:**
System prompt for the IC screener. Instructs the model to screen a deal against provided hurdle rates and return a structured JSON verdict: `PASS | WATCH | FAIL`, with metric-level cards (`cap rate`, `IRR`, `CoC`, `debt yield`, `DSCR`, `price/SF`), risk flags, TIF analysis, go/no-go variable, basis sensitivity, and next steps. Uses 25-year amortization by default. Explicitly handles TIF as non-operating income.

**Model:** `claude-sonnet-4-6`, `max_tokens: 3000` (smaller than extraction, intentional).

**Target in merged product:** Same constants file as extraction prompts, or a dedicated `lib/prompts/screenerPrompt.ts`.

---

### 2.3 Screener Deal Package Builder

**Source:** `lsg_ingest.html`, `runScreen()` lines 1903–1988 (the prompt construction block specifically)

**What it is:**
The logic that constructs the deal screening package — the text block passed to `SCREEN_SYSTEM`. It draws from `raw_data` fields across multiple sub-objects: `property`, `returns`, `sources_uses`, `capital_sources`, `cash_flows`, `market`, `top_tenants`, `investment_thesis`. It also accepts analyst overrides (price override, cap override, context note) that take precedence.

The NOI formatting block (`noiArr`), tenant summary block (`tenantSummary`), and the multi-section prompt template are all worth porting. The specific field paths (`r.returns.irr_levered_5`, `r.cash_flows.noi[]`, etc.) are authoritative references to the schema.

**Target in merged product:** A utility function, e.g., `lib/screener/buildScreenPackage.ts`. Takes a deal record and optional overrides; returns a prompt string.

---

### 2.4 Hurdle Configuration

**Source:** `lsg_ingest.html`, `HURDLE_DEFAULTS` and `getHurdles()` (lines 2279–2318)

**What it is:**
LSG investment hurdle thresholds:
- Cap rate: ≥ 8.5%
- IRR: ≥ 15.0%
- CoC: ≥ 10.0%
- Debt yield: ≥ 8.0%
- DSCR: ≥ 1.25
- Max LTV: 75%
- Assumed debt rate (when unknown): 6.5%
- Assumed LTV (when unknown): 65%

These are not arbitrary — they represent LSG's actual IC criteria. Port verbatim as defaults.

**Target in merged product:** A configuration constant, e.g., `lib/config/hurdles.ts`. In the merged product, if multi-user support exists, per-user hurdle overrides should be stored in Supabase, not localStorage.

---

### 2.5 Supabase Write Logic

**Source:** `lsg_ingest.html`, `writeToSupabase()` (lines 1388–1471)

**What it is:**
The function that persists an extracted deal to Supabase. Port all of the following behaviors exactly:

1. **Strip trailing slash** from Supabase URL before constructing endpoint — `url.replace(/\/+$/, '')`.
2. **Delete `_reader_narrative`** from the payload before write — `delete rawClean._reader_narrative`. The Reader agent narrative is large and must never be stored.
3. **Payload size guard at 900,000 bytes** — if `JSON.stringify(rawClean).length > 900000`, trim `lease_abstracts` to 20 entries and trim all `cash_flows` arrays to 10 years.
4. **Row construction** — scalar index columns (`deal_name`, `address`, `asset_type`, `purchase_price`, `going_in_cap`, `irr_levered_5`, `moic_5`, `status`, `source_broker`, `assignee`, `ingested_at`, `source_files`, `schema_version`) plus full `raw_data` JSONB. Scalar values are parsed from string formats using `parseAmt()` and `parsePct()`.
5. **HTTP error translations** — 401 = invalid API key, 403 = RLS not disabled, 404 = table doesn't exist, 409 = duplicate. These are user-facing messages, not just logs.
6. **UUID extraction from response** — `Array.isArray(saved) ? saved[0]?.id : saved?.id`. Must validate that a UUID was returned; throw if missing.
7. **`Prefer: return=representation` header** — required to get the UUID back from Supabase on POST.

**Target in merged product:** A server-side utility function, e.g., `lib/supabase/deals.ts → writeDeal(data)`. Use the Supabase server client (not the anon browser client) if running on Vercel.

---

### 2.6 Supabase Read Logic

**Source:** `lsg_ingest.html`, `loadDeals()` (lines 1550–1573)

**What it is:**
Fetches all deals from the `deals` table ordered by `ingested_at` descending, limit 100. The select field list is the canonical set of scalar columns for the pipeline view:
`id, deal_name, address, asset_type, purchase_price, going_in_cap, irr_levered_5, moic_5, status, source_broker, assignee, ingested_at, raw_data`

**Note:** The screener saves a `screen` column via PATCH (see §2.7 below), but this column is not in the current `loadDeals()` SELECT. In the merged product, add `screen` to this query if screen results should persist across page loads.

**Target in merged product:** A utility function or React hook, e.g., `lib/supabase/deals.ts → fetchDeals()`. In React, wrap in `useEffect` or use a server component.

---

### 2.7 Screener Result Persistence

**Source:** `lsg_ingest.html`, `runScreen()` lines 2022–2038

**What it is:**
After getting a screen result from the AI, PATCHes it to Supabase as `{ screen: result }` on the deals row:
`PATCH /rest/v1/deals?id=eq.{encodeURIComponent(dealId)}`

**Known gap in Repo A:** The `screen` column is not defined in the documented CREATE TABLE SQL. The PATCH may be failing silently (line 2035: `console.warn`). Screen results currently survive only in-memory (`allDeals[idx].screen = result`). In the merged product, decide explicitly: add a `screen JSONB` column to the Supabase table, or store screen results inside `raw_data.screen`. Document the decision.

**Target in merged product:** Add `screen JSONB` as an explicit column in the `deals` table. Add it to the `loadDeals()` SELECT. Update the CREATE TABLE reference SQL.

---

### 2.8 Inline Deal Edit / PATCH Logic

**Source:** `lsg_ingest.html`, `saveEdits()` (lines 2126–2273)

**What it is:**
PATCHes a deal record to Supabase with both scalar column updates and nested `raw_data` field updates in a single call. Scalar columns patched: `deal_name`, `address`, `asset_type`, `status`, `source_broker`, `purchase_price`, `going_in_cap`, `irr_levered_5`, `moic_5`, `coc_1`. Nested raw_data paths patched: `property.*`, `returns.*`, `sources_uses.*`, `capital_sources.*`, `market.name`, `cash_flows.noi[0]`.

The pattern of using Supabase's JSONB path update (`raw_data->>'field'` vs. top-level column) is worth porting as a model for any edit operation.

**Target in merged product:** A utility function `lib/supabase/deals.ts → patchDeal(id, updates)`. In React, called from a form's submit handler.

---

### 2.9 DB Sync Logic (One-Pager)

**Source:** `lsg_one_pager.html`, `syncToDB()` and `buildSyncRow()` (lines ~1007–1090)

**What it is:**
Routes to PATCH if `opData._deal_id` exists (update existing record), POST if not (new record). Saves returned UUID back to `_deal_id`. The `buildSyncRow()` function constructs the same scalar + JSONB payload as `writeToSupabase()` in the ingest tool. The two row-building functions must stay in sync in the merged product — extract them into a single shared utility.

**Target in merged product:** `lib/supabase/deals.ts → upsertDeal(data)`. This should be the same function used by both the ingest pipeline and the one-pager sync. Not two separate implementations.

---

### 2.10 File Preparation

**Source:** `lsg_ingest.html`, `prepareContent()` (lines 872–887); `lsg_one_pager.html`, `prepareFileContent()` (similar)

**What it is:**
- XLSX files → SheetJS → all sheets converted to CSV text blocks with `=== Sheet: [name] ===` headers. This is required because the Anthropic API does not accept XLSX as a native document type.
- PDF files → `FileReader.readAsDataURL()` → base64 encoded → `{ type: 'document', source: { type: 'base64', media_type: 'application/pdf', data: b64 } }`

This is a **browser-side operation** (uses File API and FileReader). It cannot be moved to a Vercel API route without first uploading the file. In a React app, this utility runs in the browser before making the API call.

**SheetJS dependency:** Currently loaded via CDN (`xlsx.full.min.js`). In React/Vercel, replace with npm package `xlsx`. Behavior is identical.

**Target in merged product:** `lib/files/prepareContent.ts → prepareContent(file: File)`. Client-side utility. Import `XLSX` from `'xlsx'` (npm) instead of CDN global.

---

### 2.11 Formatting Utilities

**Source:** `lsg_one_pager.html` (lines 396–413) and `lsg_ingest.html` (lines 2537–2542)

**Port verbatim:**

```
pct(v, dec=1)    — Smart % formatter. Handles: pre-formatted "10.2%", raw decimal ≤1.5 (multiply × 100), whole number. Do not change the ≤1.5 threshold — this is a deliberate decision for how the schema stores percentages.
pctRaw(v, dec=1) — % formatter for already-formatted strings only.
cur(v, dec=0)    — Currency formatter with locale comma separation.
curM(v)          — Millions formatter: "$65.1M" for values ≥ 1e6, falls back to cur() otherwise.
mul(v)           — Multiplier formatter: "2.35x".
v(obj, ...keys)  — Safe nested value getter with fallback to '—'. Essential for rendering raw_data where any field may be null.
parseAmt(v)      — Parses "$65.5M", "$65,500,000", or plain numbers to raw number. Handles 'M' suffix.
parsePct(v)      — Parses "8.5%", "8.5", or "0.085" to percentage number.
esc(v)           — HTML escape. Required wherever string values are interpolated into HTML.
```

**Target in merged product:** `lib/formatters.ts`. These are pure functions with no DOM dependencies. Port verbatim — the `≤1.5` threshold in `pct()` is especially non-obvious and must not be changed without testing against real schema output.

---

### 2.12 Schema Validation

**Source:** `lsg_ingest.html`, `validateSchema()` and `countPopulated()` (lines 1366–1386)

**What they are:**
- `validateSchema(obj)` — checks 5 required fields: `deal_name`, `address/city`, `sources_uses.purchase_price`, `cash_flows.noi[]`, `top_tenants[]`. Returns warning strings.
- `countPopulated(obj, depth=0)` — recursive field counter, max depth 4. Counts non-null, non-empty, non-`'—'` values. Used in the extraction summary display.

**Target in merged product:** `lib/schema/validateDeal.ts`. Pure functions, no DOM dependencies.

---

### 2.13 Metadata Override Logic

**Source:** `lsg_ingest.html`, `buildMetaContext()` and `applyMetadata()` (lines 1342–1364)

**What they are:**
- `buildMetaContext()` — builds a text block prepended to the Standardizer prompt with analyst-tagged fields (deal name, asset type, status, broker, date, assignee). The instruction "use these values, do not override from document" is load-bearing — it prevents the AI from overwriting analyst-specified values with less reliable document values.
- `applyMetadata()` — after extraction, merges analyst-tagged metadata into the extracted JSON. Sets `_assignee`, `_ingested_at`, `_source_files`.

**Target in merged product:** `lib/extraction/metadata.ts`. These two functions are tightly coupled to the extraction pipeline flow — keep them together.

---

### 2.14 File Type Auto-Detection

**Source:** `lsg_ingest.html`, `addFiles()` (lines 835–848)

**What it is:**
Filename-based heuristics for auto-tagging uploaded documents:
- `t12` or `trailing` → T12
- `rent` or `roll` or `rr` → Rent Roll
- `.xlsx`/`.xls` without `om` in name → Excel Model
- default → OM

These heuristics set the document type label prefixed to each file block in the Reader agent prompt (`--- Document: [name] (Type: [type]) ---`). The type tag influences how Agent 1 reads the document.

**Target in merged product:** `lib/files/detectFileType.ts → detectFileType(filename: string)`. Pure utility.

---

### 2.15 One-Pager Rendering Logic

**Source:** `lsg_one_pager.html`, `buildOP()`, `renderSection()`, and the 17 named section renderers (lines ~777–1180)

**What it is:**
The full rendering engine that converts a schema v2.0 JSON object into a structured deal one-pager. Sections: Property Summary, Transaction, Returns, Valuation, Sources & Uses, Capital Stack, Top Tenants, Lease Abstracts, Cash Flows, Market, Sensitivity, Demographics, Expenses, Tenant Mix, Risks, Scorecard, Opportunity Description banner.

The rendering logic is currently DOM manipulation (generating HTML strings). In React, this becomes JSX. The **data access patterns** — how the rendering logic accesses schema fields — are authoritative and should be ported faithfully. The HTML structure can change.

**Target in merged product:** `/components/OnePager/` directory with one component per section (e.g., `ReturnsSummary.tsx`, `CapitalStack.tsx`, `TopTenants.tsx`). Each section component takes `data: DealRecord` as a prop.

---

### 2.16 Gap Detection

**Source:** `lsg_one_pager.html`, `renderGapPanel()` and the 21-field tracked list (lines ~specific in the schema section)

**What it is:**
The 21 fields tracked for completeness:
`deal_name, address, property.type, property.size_sf, property.occupancy, returns.going_in_cap, returns.irr_levered_5, returns.moic_5, property.walt, transaction.ask_price, capital_sources.ltc, capital_sources.ltv, cash_flows.noi[0], scorecard.overall_grade, investment_thesis.broker_headline, deal_risks, valuation.stable_value_psf_today, returns.coc_pre_refi_5, property.vintage, property.tenant_count, market.submarket`

**Target in merged product:** `lib/schema/gapDetection.ts → detectGaps(data: DealRecord): string[]`. Returns array of dot-path strings for fields that are null/empty. The 21-field list is the authoritative completeness standard.

---

## 3. Port in Order

Recommended migration sequence. Each step is a prerequisite for the next.

```
Phase 1 — Schema Foundation
  1. TypeScript types for schema v2.0
     Source: master doc Section 4 + STANDARDIZER_SYSTEM JSON template
     Target: types/deal.ts
     Why first: every other utility depends on these types

  2. Supabase deals table verification
     Confirm: deals table in merged product's Supabase instance matches
     the CREATE TABLE SQL exactly (including the screen JSONB column decision)
     Why second: can't test persistence until schema is confirmed

Phase 2 — Persistence Layer
  3. writeDeal() / upsertDeal() utility
     Source: writeToSupabase() + syncToDB() / buildSyncRow()
     Target: lib/supabase/deals.ts
     Includes: payload guard, _reader_narrative deletion, scalar row construction
     Why: all pipeline output flows through this

  4. fetchDeals() utility
     Source: loadDeals() query
     Target: lib/supabase/deals.ts
     Includes: add 'screen' to SELECT field list
     Why: pipeline view depends on this

  5. patchDeal() utility
     Source: saveEdits() PATCH pattern
     Target: lib/supabase/deals.ts
     Why: screener result save and inline edit both use this

Phase 3 — Intelligence Layer
  6. AI prompts constants
     Source: READER_SYSTEM, STANDARDIZER_SYSTEM, SCREEN_SYSTEM
     Target: lib/prompts/extractionPrompts.ts + lib/prompts/screenerPrompt.ts
     Why: extraction and screener API routes depend on these

  7. File preparation utilities
     Source: prepareContent()
     Target: lib/files/prepareContent.ts (client-side)
     Includes: SheetJS npm import, PDF base64, type detection heuristics
     Why: prerequisite for extraction pipeline

  8. Metadata utilities
     Source: buildMetaContext() + applyMetadata()
     Target: lib/extraction/metadata.ts
     Why: prerequisite for extraction pipeline

  9. Extraction pipeline (two-agent orchestrator)
     Source: runIngest() orchestration + Agent 1 and Agent 2 fetch calls
     Target: See §6 for routing decision
     Includes: validateSchema(), countPopulated(), conflict surfacing
     Why: core product function

Phase 4 — Screener
  10. Hurdle configuration
      Source: HURDLE_DEFAULTS
      Target: lib/config/hurdles.ts

  11. Screen package builder
      Source: runScreen() prompt construction block
      Target: lib/screener/buildScreenPackage.ts

  12. Screen API call + result persistence
      Source: runScreen() API fetch + PATCH
      Target: screener API route or client utility

Phase 5 — Presentation
  13. Formatting utilities
      Source: pct(), cur(), curM(), mul(), v(), parseAmt(), parsePct(), esc()
      Target: lib/formatters.ts

  14. Gap detection
      Source: renderGapPanel() field list + logic
      Target: lib/schema/gapDetection.ts

  15. One-pager section components (JSX)
      Source: buildOP(), renderSection(), 17 section renderers
      Target: components/OnePager/
      Note: Port data access logic; rebuild rendering as JSX

Phase 6 — Validation
  16. End-to-end test: upload OM → extract → Supabase write → pipeline view → screen → one-pager render
      No new features until this passes on a real deal.
```

---

## 4. Authority Decisions

The following are not open questions in the merged product. They are settled decisions from Repo A that must carry forward.

**A. Schema v2.0 is the extraction schema.**
The `STANDARDIZER_SYSTEM` prompt contains the authoritative JSON template with ~200 fields across 25 objects. If Repo B uses a different or narrower schema, Repo A's schema wins unless there is explicit, documented evidence that Repo B's schema is more complete or more accurate. Do not merge the schemas without a field-by-field comparison.

**B. `raw_data` JSONB is the source of truth. Scalar columns are index-only.**
The `deals` table stores full extracted JSON in `raw_data`. Scalar columns (`deal_name`, `going_in_cap`, etc.) are populated for filtering and sorting only — they are not the authoritative values. Any feature that reads deal data must read from `raw_data`, not from scalar columns. Do not add new scalar columns for extracted fields.

**C. Two-agent pipeline architecture.**
Reader and Standardizer are separate agents with separate API calls. They must not be collapsed into a single prompt. Reason: the Reader reads faithfully without schema constraints; the Standardizer maps without re-reading PDFs. This separation is a correctness decision. The Standardizer can be re-run on a new schema version without re-ingesting source documents.

**D. `conflict_flags[]` and `extraction_meta` must be preserved.**
These are not cosmetic. `conflict_flags[]` surfaces cases where two source documents disagree (e.g., OM NOI vs. T12 NOI). `extraction_meta.confidence`, `fields_populated`, and `schema_version` are the quality record of an extraction. Do not drop these in a "simplified" schema.

**E. `_deal_id` as UUID linkage key.**
The extracted JSON carries `_deal_id` (the Supabase row UUID) after a successful write. This field is the bridge between extraction output and the pipeline record. Any operation that updates an existing deal — inline edit, DB sync from one-pager, screener result save — uses this UUID. The pattern `if(_deal_id exists → PATCH; else → POST)` must be preserved.

**F. Payload trimming at 900,000 bytes.**
Before any Supabase write: delete `_reader_narrative`, check payload size, trim `lease_abstracts` to 20 entries and `cash_flows` arrays to 10 years if over 900KB. This is a runtime guard against Supabase's body size limits. Must survive the port.

**G. `encodeURIComponent()` on all deal IDs in Supabase REST URLs.**
All Supabase PATCH/GET URLs that include a deal ID must use `encodeURIComponent(dealId)`. This is a security fix applied in the April 14 session. Do not use bare UUID interpolation.

**H. Agent 1 model: `claude-sonnet-4-6`, `max_tokens: 8000`.**
**Agent 2 model: `claude-sonnet-4-6`, `max_tokens: 16000`.**
**Screener model: `claude-sonnet-4-6`, `max_tokens: 3000`.**
Do not reduce `max_tokens` below these values. The Standardizer requires 16,000 tokens to produce a complete schema v2.0 JSON for a dense OM. Reducing this causes truncated output, which is a silent failure (valid JSON that ends early).

**I. Analyst metadata overrides extraction.**
`buildMetaContext()` prepends analyst-tagged values to the Standardizer prompt with explicit instruction to not override them with document values. `applyMetadata()` enforces this post-extraction. This order is intentional — the analyst's pre-tag is more reliable than extracted values for fields like deal name, status, and broker.

---

## 5. What Not to Port Directly

**Do not move these assets as-is into the merged product:**

**Single-file HTML architecture.**
`lsg_ingest.html` (2,558 lines) and `lsg_one_pager.html` (1,837 lines) mix HTML, CSS, and JS because there is no build step. In a React/Vercel app, extract logic as described in §2 and §6. Do not concatenate the HTML files' script blocks into a React app.

**Inline CSS.**
Both files have thousands of lines of CSS co-mingled with HTML. Extract the visual design language (color variables, spacing, typography) and re-implement in the merged product's styling system. Do not copy CSS blocks verbatim.

**Pipeline tab embedded inside the ingest tool.**
The pipeline browser tab in `lsg_ingest.html` is functional but architecturally wrong — a deal browser should not live inside a file upload tool. In the merged product, the pipeline view is a dedicated route.

**`anthropic-dangerous-direct-browser-access: true` header.**
This header allows Anthropic API calls directly from the browser, bypassing CORS restrictions. It is acceptable in a single-user internal tool where the user supplies their own API key. In a multi-user product on Vercel, the Anthropic key must be a server-side environment variable and calls must go through a Vercel API route. See §7 for the timeout constraint this creates.

**localStorage credential storage.**
Repo A stores Anthropic API key, Supabase URL, and Supabase anon key in `localStorage` because there is no backend. In the merged product on Vercel with Supabase auth, credentials are environment variables (server-side) or user session tokens (client-side). Do not replicate the `KEYS` localStorage pattern.

**`getToolUrl()` path resolution.**
`lsg_ingest.html` uses `getToolUrl(filename)` to construct sibling file URLs dynamically (works on GitHub Pages, localhost, Netlify). In a React app with React Router, sibling navigation is a route change, not a URL construction. This function has no equivalent use in the merged product.

**Demo / hardcoded OKC Outlets data.**
The `loadDemo()` function in `lsg_one_pager.html` loads a hardcoded OKC Outlets JSON object. This was a development aid. In the merged product, demo/test data should come from the database.

**Collapsible Supabase setup guide.**
The setup guide (CREATE TABLE SQL, RLS instructions, error translations) is embedded in the ingest tool's UI. In the merged product, this belongs in a setup document or onboarding flow, not in-app.

**Dated master doc filename convention.**
`LSG_TOOLS_MASTER_2026-04-14.md` — date-stamped filenames create confusion in a multi-contributor repo. Use a stable filename; record dates inside the document.

**Hurdles in localStorage.**
`HURDLES_KEY = 'lsg_hurdles_v1'` stores IC hurdle overrides in the browser. In a multi-user product, hurdles should be stored per-user in Supabase or in a team configuration. The default values are still authoritative (see §4.D), but the storage mechanism should change.

---

## 6. Translation Guidance for React / Vercel / Supabase

For each major preserved asset, where it should land and how.

| Repo A Asset | Source Location | Target in Merged Product | Type |
|---|---|---|---|
| `READER_SYSTEM` prompt | `lsg_ingest.html` ~line 919 | `lib/prompts/extractionPrompts.ts` | Server-side constant |
| `STANDARDIZER_SYSTEM` prompt + schema v2.0 template | `lsg_ingest.html` ~line 970 | `lib/prompts/extractionPrompts.ts` | Server-side constant |
| `SCREEN_SYSTEM` prompt | `lsg_ingest.html` line 1872 | `lib/prompts/screenerPrompt.ts` | Server-side constant |
| Extraction pipeline orchestration | `runIngest()` | `app/api/ingest/route.ts` (Vercel) **or** client-side hook — see §7 | API route **or** React hook |
| Screener API call | `runScreen()` fetch | `app/api/screen/route.ts` (Vercel) **or** client-side — see §7 | API route **or** React hook |
| Screen package builder | `runScreen()` prompt construction | `lib/screener/buildScreenPackage.ts` | Utility function (shared) |
| `writeToSupabase()` | `lsg_ingest.html` line 1388 | `lib/supabase/deals.ts → writeDeal()` | Server utility |
| `syncToDB()` + `buildSyncRow()` | `lsg_one_pager.html` ~line 1007 | `lib/supabase/deals.ts → upsertDeal()` | Server utility |
| `loadDeals()` | `lsg_ingest.html` line 1550 | `lib/supabase/deals.ts → fetchDeals()` + React hook or server component | Data fetch |
| `patchDeal()` pattern | `saveEdits()` | `lib/supabase/deals.ts → patchDeal()` | Utility function |
| `prepareContent()` | `lsg_ingest.html` line 872 | `lib/files/prepareContent.ts` | **Client-side** utility |
| File type auto-detection | `addFiles()` lines 835–848 | `lib/files/detectFileType.ts` | Client-side utility |
| `buildMetaContext()` + `applyMetadata()` | `lsg_ingest.html` lines 1342–1364 | `lib/extraction/metadata.ts` | Utility functions |
| `validateSchema()` + `countPopulated()` | `lsg_ingest.html` lines 1366–1386 | `lib/schema/validateDeal.ts` | Utility functions |
| `pct()`, `cur()`, `curM()`, `mul()`, `v()` | `lsg_one_pager.html` lines 396–413 | `lib/formatters.ts` | Utility functions |
| `parseAmt()`, `parsePct()` | `lsg_ingest.html` lines 2540–2542 | `lib/formatters.ts` | Utility functions |
| `esc()` | Both files | Not needed in React JSX (JSX escapes by default). Only needed if generating raw HTML strings. | Utility (conditional) |
| Gap detection (21 fields) | `lsg_one_pager.html` | `lib/schema/gapDetection.ts → detectGaps()` | Utility function |
| `HURDLE_DEFAULTS` | `lsg_ingest.html` line 2280 | `lib/config/hurdles.ts` | Configuration constant |
| One-pager section renderers | `buildOP()`, `renderSection()`, 17 sections | `components/OnePager/` — one component per section | React components |
| One-pager layout persistence | localStorage `lsg_op_layout_v1` | `components/OnePager/useLayout.ts` — keep localStorage for now; upgrade to Supabase later | React hook |
| Screener result rendering | `renderScreenResult()` | `components/Screener/ScreenResult.tsx` | React component |
| Screener metric cards | `renderScreenerSection()` metric grid | `components/Screener/MetricCard.tsx` | React component |
| Deal compare table | `initCompare()`, `ROWS` definition | `components/PipelineCompare/` | React component |
| Deal metadata form fields | `getMetadata()` field list | `components/Ingest/DealMetaForm.tsx` | React component |
| Pipeline table columns | `renderDeals()` column list | `components/Pipeline/PipelineTable.tsx` | React component |
| Schema v2.0 field reference | `STANDARDIZER_SYSTEM` JSON template | `types/deal.ts` TypeScript interfaces | Type definitions |

---

## 7. Risks and Watchouts

### CRITICAL: Vercel Serverless Function Timeout

The two-agent extraction pipeline takes **3–4 minutes** (Agent 1: ~60–90s, Agent 2: ~60–90s, sequential). Vercel serverless functions timeout at **10 seconds by default**, **60 seconds on Pro**, **300 seconds on Enterprise**.

This means the extraction pipeline **cannot be naively moved to a Vercel API route** on any plan below Enterprise.

**Options — decide before merge execution:**

| Option | How | Trade-off |
|---|---|---|
| Keep client-side | `anthropic-dangerous-direct-browser-access: true`, user supplies API key | Maintains Repo A behavior. Exposes API key in browser network requests. Not suitable if API key is team-shared. |
| Vercel Edge Function with streaming | Edge runtime + streaming response | Eliminates timeout. Requires streaming progress UI. More complex to implement. |
| Background job queue | Trigger job via API route, poll for completion | Decouples UI from execution time. Requires a job store (Supabase table works). More complex but most scalable. |
| Vercel Enterprise | No code change | Cost/plan decision for Lightstone. Not a technical solution. |

**Recommendation:** If the merged product's API key is team-managed (not user-supplied), background job queue is the right answer. If each user supplies their own API key (current Repo A model), keep client-side for now.

---

### Schema v1.0 Drift in One-Pager

`lsg_one_pager.html` still uses extraction schema v1.0 in its single-pass extraction prompt. The ingest tool uses v2.0. This drift exists today in Repo A and must be resolved **before merge**, not carried forward. The one-pager's extraction prompt is missing the `investment_thesis`, `lease_abstracts`, `deal_risks`, and several `returns` sub-objects present in v2.0.

If the one-pager port happens before this is fixed, the merged product will inherit schema drift.

---

### Payload Size Guard Must Not Be Lost in Translation

The 900KB guard and the `lease_abstracts`/`cash_flows` trimming logic is buried inside `writeToSupabase()`. When this function is refactored into a `writeDeal()` utility, the guard is easy to drop if not explicitly checked. On dense OMs with many tenants and 10-year cash flows, payloads can exceed 1MB. Without the guard, Supabase writes will silently fail on large deals.

---

### `screen` Column is Undocumented

The screener saves results via `PATCH { screen: result }` to Supabase. The `screen` column is not in the CREATE TABLE SQL documented in the master doc. One of three things is true:
1. The column exists in the live Supabase instance but was added ad-hoc and never documented
2. The PATCH is failing silently and screen results are not persisted (only in-memory)
3. The column doesn't exist and this is a known limitation

Verify against the live Supabase instance before porting. Add `screen JSONB` to the authoritative table schema and to the `fetchDeals()` SELECT if it should persist.

---

### Row Construction Divergence

`writeToSupabase()` in the ingest tool and `buildSyncRow()` in the one-pager both construct the scalar + JSONB row, but they are separate implementations that can drift. In the merged product, there must be exactly one `buildDealRow()` function. Verify both implementations agree on the `address` construction logic:
```
[data.address, data.city, data.state].filter(Boolean).join(', ')
```
and on how `going_in_cap` and `irr_levered_5` are parsed via `parsePct()`.

---

### `pct()` Threshold is Non-Obvious

The `pct()` formatter uses `Math.abs(n) <= 1.5` to detect whether a value is a raw decimal (e.g., `0.085`) or a whole number (e.g., `8.5`). This threshold was calibrated for how the schema stores percentages. If it is changed, any field with a value between 1.5% and 100% (e.g., a 2% cap rate) will be misformatted. Do not modify this threshold without testing against real extraction output.

---

### `esc()` Becomes Irrelevant in JSX — But Watch String Interpolation

In Repo A, `esc()` is used everywhere HTML strings are constructed via template literals. In React JSX, output is escaped by default. However, if any Repo B code generates HTML strings (e.g., for `dangerouslySetInnerHTML`, email templates, or clipboard copy), `esc()` or equivalent must still be used. The one-pager print export and the screen brief copy-to-clipboard are two surfaces to watch.

---

### Model Pinning

All three AI calls in Repo A use `claude-sonnet-4-6`. Do not change this to a newer or cheaper model without re-testing extraction quality. The schema v2.0 template in the Standardizer prompt was calibrated for this model's JSON output behavior. A different model may produce valid JSON that populates the schema differently.

---

### File Preparation is Browser-Side Only

`prepareContent()` uses the browser's `File` API and `FileReader`. It cannot run on a Node.js server without a file upload step first. If the extraction pipeline moves to a Vercel API route, file content must be uploaded (as multipart form data or base64 strings) before the API route processes it. This changes the ingest UX: the user uploads files, the server processes them, rather than the browser processing them directly.

---

### SheetJS CDN → NPM

Repo A loads SheetJS from CDN: `xlsx.full.min.js`. In a React/Vercel app, install as `npm install xlsx` and import `import * as XLSX from 'xlsx'`. The API is identical. Do not load from CDN in a Node.js context.

---

### No Tests in Either Repo

Neither repo has automated tests. The merged product inherits zero test coverage for the extraction pipeline, screener, or one-pager rendering. The first real-world risk surface is incorrect extraction or silent schema drift. Consider adding at minimum a schema validation test that runs `validateSchema()` against a known-good extraction fixture before any merge milestone is declared complete.

---

## Appendix: Assumptions

The following assumptions underlie this document. If any are wrong, specific sections above will need revision.

1. **Repo B is React.** If Repo B is also vanilla JS, some "do not port" items (inline CSS, HTML architecture) become relevant again.
2. **Merged product is a single deployed app.** If the two tools remain separate deployments that share a Supabase instance, the authority decisions in §4 still apply but the port strategy in §6 changes.
3. **Supabase instance is shared.** The live `deals` table from Repo A's Supabase project carries forward. The merged product does not start with an empty database.
4. **No multi-user auth yet.** When auth is added, localStorage credential storage, hurdle persistence, and layout persistence will all need to move to user-scoped storage in Supabase.
5. **`screen` column status is unknown.** §7 calls this out as a verification step before merge.
6. **Schema v1.0 drift in one-pager is not resolved at time of writing.** This must be fixed before or during porting, not after.
