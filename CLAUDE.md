# SteelOS Ore — project guide for Claude Code

AI-first ERP for iron ore traders and exporters. Product of TheSteelOS (thesteelos.com). First customer: Ganesha Steel, Kolkata.

## Source of truth
- `docs/PRD.md` — full product spec: data model (Section 6), calculation engines (Section 7), modules, dashboard spec (Section 9), phasing (Section 14), seed data (Section 15). Read the relevant section before building any feature.
- `prototype/index.html` — working clickable prototype. Match its screens, copy, layout and design tokens. Its settlement engine (`settle()` function) is the reference implementation.

## Business rules that must never break
- Domestic trade is back-to-back: **no domestic stock**, with one confirmed, scoped exception — the trader's own warehouse (`WarehouseLot`, PRD §6.3, §2.1). A trip can go mine → warehouse and later warehouse → plant; this must stay the *exception* path (its own `Trip.destination_type`/`origin_type`), never the default. Every other domestic trip is still mine → plant direct. Export stock still exists only at port plots.
- The atomic unit of domestic trade is the **trip** (one truck, mine → plant).
- Settlement engine test case (PRD 7.2) must always return **₹4,29,600** deductions on ₹2,00,00,000 gross. Keep it as an automated test.
- Debit note test (R-2291): plant claims ₹6,30,400, contract allows ₹5,70,400 → ₹60,000 over-deducted.
- Export duty rule: Fe ≥ 58.00% → 30%, below → nil. Configurable, never hard-coded. Store both dry and as-received Fe.
- FEMA realisation deadline = shipping bill date + 15 months.
- Filings are prepared and checked by the system but approved by the CA before filing. E-way bills / e-invoices are generated in real time when checks pass.
- AI never posts money, confirms saudas, files returns or messages external parties without a human action.

## Stack
- Frontend: Vite + React + TypeScript, mobile-first PWA for field roles (offline queue for mine/plant staff).
- Backend: Postgres (Supabase: auth, storage, row-level security). Role-based redaction of rates/margins enforced server-side (RLS / views), not only in UI.
- AI: Claude API for document extraction, morning brief, Ask, debit-note and LC checks.
- Integrations: Tally Prime connector (XML over HTTP, small Node service on the customer's Tally PC), GST e-invoice/e-way bill via GSP API, WhatsApp Business API.

## Conventions
- Indian number formatting everywhere: ₹12,34,567, lakh (L) and crore (Cr). Dates as 18 Sep 2026.
- Every table has company_id (tenant), created/updated by/at. Every change written to audit_log.
- Money in integer paise; tonnes as numeric(12,3); assay values numeric(6,3).
- Design tokens: see PRD 13.1 (stone background, ink, hematite accent #A63D2A, Instrument Sans / Instrument Serif / IBM Plex Mono for numbers). Light and dark themes.
- Sentence-case UI copy; buttons say exactly what happens.

## Build order
Phase 0 (demo) → Phase 1 (domestic MVP) → Phase 2 (export). See PRD Section 14. Build one module at a time, with tests, and commit after each.

## Open questions (don't assume — ask the user)
See PRD Section 17: freight payer, private plant penalty terms, road vs rail share, FOB/CFR, port blending, Fe basis for duty, canalisation status, volumes, Tally setup, CA filings.
