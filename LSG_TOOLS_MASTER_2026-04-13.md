# LSG Tools — Master Project Document

> **Usage:** Paste this file (or relevant sections) at the top of every new Claude chat before uploading HTML files.
> **Maintained by:** Phil Jomphe — update after every work session.
> **Last updated:** April 13, 2026

---

## 1. Project Overview

Internal deal intelligence toolset for Lightstone Group acquisitions team. Two standalone HTML files hosted on GitHub Pages. No build tools, no frameworks, no backend. All AI via direct Anthropic API browser calls. All persistence via Supabase REST API.

**Live URL base:** `https://[yourusername].github.io/lsg-tools/`

**Anthropic model in use:** `claude-sonnet-4-6` (16k max_tokens)

**Stack:**
- Frontend: Vanilla HTML/CSS/JS — single-file, no dependencies except SheetJS (CDN) for XLSX parsing
- AI: Anthropic API called directly from browser (`anthropic-dangerous-direct-browser-access: true`)
- Database: Supabase (Postgres + JSONB) — free tier
- Hosting: GitHub Pages (public repo)
- Fonts: DM Sans (one-pager), IBM Plex Sans + IBM Plex Mono (ingest)

---

## 2. File Registry

### `lsg_one_pager.html`
**Purpose:** Renders a formatted LSG internal one-pager from extracted deal JSON. Mirrors the internal underwriting summary format (OKC Outlets is the reference design).

**Key capabilities:**
- Multi-file upload (up to 3 files: PDF + XLSX) with type tagging (OM / T12 / Rent Roll / Excel Model)
- Single-pass AI extraction via Anthropic API → renders ~80-field JSON into formatted one-pager
- Drag-and-drop Layout Editor: rearrange sections between 4 main columns + Bottom zone + Hidden
- Column width controls per column: Narrow (0.7fr) / Normal (1.0fr) / Wide (1.4fr) / X-Wide (1.8fr)
- Auto-collapse empty columns (renders `0fr` / `display:none`)
- Gap panel: surfaces missing fields as amber tags; click any tag to jump to that field in Edit form
- Structured Edit form: tabbed UI (Deal / Property / Transaction / Returns / Debt / Raw JSON)
- Demo mode: loads OKC Outlets hardcoded data
- Print/PDF export via `window.print()` (landscape, hides UI chrome)
- Session persistence via localStorage (`lsg_op_data_v1`)
- Layout config persistence via localStorage (`lsg_op_layout_v1`, `lsg_op_colwidths_v1`)
- Nav link to `lsg_ingest.html` (dynamic path via `getToolUrl()`)

**Key functions:**
- `runExtract()` — multi-file extraction pipeline
- `prepareFileContent(file)` — PDF → base64 document block; XLSX → SheetJS CSV text block
- `renderOnePager(d)` — builds HTML from JSON via `buildOP(d)`
- `buildOP(d)` — main one-pager renderer; reads layout config for dynamic grid
- `renderSection(id, d)` — renders individual sections (returns, capital, tenants, cash flows, etc.)
- `renderGapPanel(d)` — detects null/empty fields from 21 tracked fields; shows amber panel
- `editData(focusPath)` — opens tabbed structured edit form; auto-focuses field if path passed
- `saveEdits()` — merges form edits into opData; falls back to raw JSON tab
- `getLayout()` / `saveLayoutConfig()` — layout persistence
- `getColWidths()` / `saveColWidths()` — column width persistence
- `getToolUrl(filename)` — resolves sibling file path dynamically for any host
- `pct(v)` — smart % formatter: handles pre-formatted strings, raw decimals ≤1.5, whole numbers

**Known issues / not yet built:**
- No Supabase write — one-pager cannot save edits back to database (planned: DB Sync button)
- Extraction is single-pass (vs. two-agent pipeline in ingest tool)
- `opData._deal_id` will be set when loaded from ingest pipeline — needed for future DB sync

**Header fields:**
- Anthropic API key only (stored in localStorage as `lsg_api_key`)

---

### `lsg_ingest.html`
**Purpose:** Two-agent OM ingestion pipeline. Reads source documents, extracts full deal intelligence into LSG schema v2.0, writes to Supabase `deals` table. Also provides a pipeline browser tab.

**Key capabilities:**
- Multi-file upload (up to 3): drag-drop or click; auto-detects type from filename
- File type tagging buttons per card: OM / T12 / Rent Roll / Excel Model / Other
- Deal metadata form: deal name, asset type, status, broker, date, assignee
- Two-agent pipeline:
  - **Agent 1 (Reader):** Reads all docs faithfully, produces narrative summary (no schema imposed). ~60-90s.
  - **Agent 2 (Standardizer):** Maps narrative to LSG schema v2.0, detects conflicts, populates `conflict_flags[]`. ~60-90s.
- Schema validation (client-side, instant)
- Auto-save to Supabase on pipeline completion
- Conflict flags displayed in amber panel post-extraction
- JSON preview with syntax highlighting
- "Open in One Pager" — writes JSON to localStorage, navigates to one-pager
- Pipeline tab: loads all deals from Supabase, searchable/filterable table
- Deal detail modal with key metrics and "Open in One Pager" button
- Collapsible Supabase setup guide (SQL, RLS instructions, error translations)
- Elapsed timer during pipeline run

**Key functions:**
- `runIngest()` — orchestrates full pipeline: file prep → Agent 1 → Agent 2 → validate → Supabase write
- `prepareContent(file)` — PDF → base64; XLSX → SheetJS CSV text
- `writeToSupabase(data)` — writes scalar index columns + full JSONB `raw_data`; handles payload sizing, RLS errors, auth errors
- `loadDeals()` — fetches pipeline table from Supabase
- `renderDeals(deals)` — renders pipeline table with status tags and nav buttons
- `openDeal(id)` — fetches full row, shows detail modal
- `loadInOnePager(id)` — fetches `raw_data`, writes to localStorage, navigates to one-pager
- `getToolUrl(filename)` — dynamic path resolution (same as one-pager)
- `buildMetaContext()` — prepends analyst-tagged metadata to Agent 2 prompt
- `applyMetadata(obj)` — merges form metadata into extracted JSON
- `validateSchema(obj)` — checks 5 required fields; returns warning array
- `countPopulated(obj)` — counts non-null fields recursively

**Header fields (all persisted to localStorage):**
- Anthropic API key (`lsg_ant_key`)
- Supabase URL (`lsg_sb_url`) — format: `https://xxx.supabase.co` (no trailing slash)
- Supabase anon key (`lsg_sb_key`)

**Known issues:**
- Pipeline takes 3-4 minutes (2 sequential API calls on large PDFs — expected, by design)
- NetworkError on Supabase write = almost always CORS (need HTTPS host) or RLS/grants issue

---

## 3. Supabase Schema

```sql
CREATE TABLE deals (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ingested_at    TIMESTAMPTZ DEFAULT now(),
  deal_name      TEXT,
  address        TEXT,
  asset_type     TEXT,
  purchase_price NUMERIC,
  going_in_cap   NUMERIC,
  irr_levered_5  NUMERIC,
  moic_5         NUMERIC,
  status         TEXT DEFAULT 'Screening',
  source_broker  TEXT,
  assignee       TEXT,
  source_files   TEXT[],
  schema_version TEXT,
  raw_data       JSONB
);

ALTER TABLE deals DISABLE ROW LEVEL SECURITY;
GRANT ALL ON TABLE deals TO anon;
GRANT ALL ON TABLE deals TO authenticated;
GRANT USAGE ON SCHEMA public TO anon;
GRANT USAGE ON SCHEMA public TO authenticated;
```

**Design principle:** Scalar columns are index/filter fields only. `raw_data` JSONB is source of truth. Adding new extracted fields never requires a migration — they land in `raw_data` automatically.

---

## 4. Extraction Schema

**Schema version:** 2.0

**Top-level objects in extracted JSON:**
- `deal_name`, `address`, `city`, `state`, `mode`
- `property` — type, SF, acreage, occupancy, economic_occupancy, tenant_count, WALT, vintage, buildings, class, parking_ratio, zoning
- `transaction` — status, strategy, seller, sourcing, ask_price, ask_price_psf, bid_deadline
- `investment_thesis` — broker_headline, stated_upside[], repositioning_narrative, value_add_components[], comp_set_referenced[], disclosed_risks[], broker_NOI_adjustment_narrative
- `scorecard` — overall_grade, market_grade, property_grade, location_grade, business_plan_grade, description, kpis[]
- `returns` — going_in_yield_cost, going_in_cap, IRR (levered/unlevered, 5Y/10Y), net_profit, MOIC, CoC
- `valuation` — stable_value_psf (today/5yr/10yr), stable_cap_rate, exit caps
- `sources_uses` — purchase_price, closing_costs, reserves, debt/equity breakdown
- `capital_sources` — LTC, LTV, LTPP, rate details, term, IO period, debt yield, DSCR
- `top_tenants[]` — name, parent_company, SF, dates, rents, options, co-tenancy, guarantee, sales, health ratio, credit_watch
- `lease_abstracts[]` — full per-tenant detail (all tenants, not just top 5)
- `by_size[]` — small shop <5k, 5k-10k, major, big box breakdown
- `cash_flows` — 10-year arrays: GPR, vacancy, EGI, revenue, OpEx, NOI, capex, CFBDS, DS, CFADS, YoC, CoC
- `market` — submarket, GS grade, rent growth, inventory, vacancy, forward growth by year
- `income_assumptions` — vacancy, credit loss, escalations, MLA terms
- `sensitivity[]` — price sensitivity table
- `demographics` — pop/HH/income/home value at 30/50/70%, crime, traffic
- `expenses` — RET, CAM, mgmt, other, capex schedule
- `tenant_use_type[]` — use category breakdown
- `deal_risks` — anchor expiry, credit watch tenants, deferred maintenance, legal/environmental/title, market headwinds
- `conflict_flags[]` — field, source_a_value, source_b_value, resolution, note
- `extraction_meta` — schema_version, fields_populated, fields_null, confidence, source_docs, conflicts_detected
- `_deal_id` — Supabase row UUID (set after save; used for future DB sync)
- `_assignee`, `_ingested_at`, `_source_files` — analyst metadata

---

## 5. Architecture Decisions (Do Not Re-Litigate)

| Decision | Choice | Rationale |
|---|---|---|
| Single file vs. build system | Single HTML files | Zero infra, no IT dependency, works on any machine |
| One agent vs. two agents | Two agents (Reader + Standardizer) | Different cognitive jobs; standardizer can be re-run without re-ingesting PDF |
| JSONB vs. 80-column schema | JSONB + scalar index columns | Schema flexibility — new fields never require migrations |
| XLSX handling | SheetJS → CSV text block | Claude API doesn't accept XLSX as native document type |
| Hosting | GitHub Pages | Free, HTTPS (required for Supabase CORS), team-accessible |
| Max tokens | 16,000 | Required for full JSON output on dense OMs |
| Model | claude-sonnet-4-6 | Current production Sonnet; balance of quality and speed |
| Layout storage | localStorage | Zero infra; layout config persists per browser |
| Column widths | fr units with auto-collapse | Empty columns collapse to 0fr automatically |
| Path resolution | `getToolUrl()` dynamic | Works on GitHub Pages, localhost, Netlify — any host |

---

## 6. Active Work Log

### Session ending April 13, 2026
**lsg_one_pager.html changes:**
- Added multi-file upload with type tagging (replaces single file input)
- Added gap detection panel (21 tracked fields)
- Replaced raw JSON edit modal with tabbed structured form (5 tabs + Raw JSON)
- Fixed Hidden column in layout editor (was clamped to Bottom via `Math.min(s.col,4)`)
- Fixed layout editor grid to show all 6 columns including Hidden
- Added column width controls (Narrow/Normal/Wide/X-Wide) per column
- Auto-collapse empty columns (`0fr` + `display:none`)
- Added opportunity description to full-width banner above header boxes
- Added `getToolUrl()` for host-agnostic navigation
- Added `DOMContentLoaded` wrapper on init block (fixes null reference on session restore)
- Added "Deal Ingest →" nav link in header
- Fixed `pct()` helper — smart formatter handles pre-formatted strings vs raw decimals
- Fixed `le-cols` grid: `repeat(4,1fr) 0.8fr 0.6fr`
- Removed Underwriting Calculator (was broken, caused null reference errors)

**lsg_ingest.html — new file:**
- Built from scratch: two-agent pipeline, Supabase write, pipeline browser tab
- Agent 1 system prompt: faithful narrative reader, no schema
- Agent 2 system prompt: LSG schema v2.0 standardizer + conflict detector
- `writeToSupabase()`: payload size guard, trailing slash strip, specific error messages per HTTP status
- CORS fix: must be served from HTTPS (GitHub Pages) — file:// protocol blocks Supabase fetch
- RLS fix: `ALTER TABLE deals DISABLE ROW LEVEL SECURITY` + `GRANT ALL ON TABLE deals TO anon`
- Collapsible setup guide with exact SQL
- Dynamic path navigation via `getToolUrl()`

---

## 7. Open Items (Prioritized Backlog)

### Immediate / Next Session
- [ ] **DB Sync button in one-pager** — writes `opData` back to Supabase; UPDATE if `_deal_id` exists, INSERT if new; saves returned `_deal_id` to localStorage
- [ ] **Test full workflow end-to-end** — ingest OM → pipeline tab → open in one-pager → edit → sync back to Supabase

### Near Term
- [ ] **Pipeline Dashboard** (`lsg_pipeline.html`) — standalone deal browser; replaces the Pipeline tab in ingest tool; richer filtering, status updates, bulk actions
- [ ] **One-pager "Load from Database" mode** — search Supabase from within one-pager; select deal; render without re-extracting
- [ ] **Schema v2.0 alignment** — one-pager extraction prompt still uses v1.0 schema; should match ingest tool's v2.0 schema (investment_thesis, lease_abstracts, deal_risks objects)
- [ ] **Multi-pass extraction orchestrator** — add validation pass after extraction; targeted re-query for flagged fields; improves accuracy from ~70% to ~90%+

### Medium Term
- [ ] **pgvector / semantic search** — embed full OM text; store in Supabase pgvector; enable semantic queries across corpus
- [ ] **Comp engine** — on new deal ingest, auto-surface 3 most similar historical deals by vector similarity
- [ ] **IC Pre-Filter Agent** — screen extracted deal against LSG criteria (cap ≥ 8.5%, IRR targets, WALT floors); output Pass / Conditional Pass / Fail with specific flags
- [ ] **DealStorm Brief Generator** — takes extracted JSON + screen output; produces one-page DealStorm format (Minto Pyramid structure)
- [ ] **Distress Monitor** — periodic semantic query for distress signals across corpus; cross-reference CMBS data

### Long Term
- [ ] **Multi-user support** — Supabase auth; team members see shared pipeline; personal vs. team views
- [ ] **Status workflow** — Screening → Active → IC → Due Diligence → Pass/Dead; status changes logged with timestamps
- [ ] **Re-extraction** — button to re-run ingest on existing deal with current prompt version; schema version bump tracking

---

## 8. How to Start a New Chat

**Paste this at the top of the new chat:**
```
I'm building the LSG Deal Intelligence toolset. Here is the master project doc:
[paste this file]

Today I want to work on: [describe task]

Uploading current files: [list files you're attaching]
```

**Always upload the HTML files you're working on** — the master doc describes intent; the files are ground truth.

**Key context to mention if relevant:**
- Phil Jomphe, Acquisitions, Lightstone Group (299 Park Ave NYC)
- Working under Sanford (SVP Investments), retail vertical (Akiva) + structured finance (Irving)
- IT restrictions — no local installs; everything runs in the browser or GitHub Pages
- No terminal access — all deployment via GitHub web UI drag-and-drop
- Supabase project already configured; `deals` table created; RLS disabled; grants applied

---

## 9. Reference: Key File Locations

| Asset | Location |
|---|---|
| One Pager Generator | `lsg_one_pager.html` (repo root) |
| Deal Ingest Pipeline | `lsg_ingest.html` (repo root) |
| This master doc | `LSG_TOOLS_MASTER.md` (repo root) |
| Index redirect | `index.html` (repo root) → redirects to lsg_ingest.html |
| Supabase project | supabase.com → your project |
| GitHub repo | github.com/[yourusername]/lsg-tools |
| Live tools | https://[yourusername].github.io/lsg-tools/ |

---

*End of master document. Update Section 6 (Active Work Log) after every session.*
