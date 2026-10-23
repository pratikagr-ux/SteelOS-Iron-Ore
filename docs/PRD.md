# PRD — SteelOS Ore
### AI-first ERP for iron ore traders and exporters
Product of TheSteelOS (thesteelos.com) · First customer: Ganesha Steel, Kolkata · Version 1.0

---

## 0. Instructions for the app builder

Build a multi-tenant web app (desktop) plus a mobile-first responsive/PWA experience for field staff. Use the data model in Section 6 as the source of truth. Every number shown on any screen must be traceable to underlying records (click a number → see the records behind it). Use Indian number formatting everywhere (₹12,34,567 · lakh · crore). Seed the app with the demo data in Section 15 so every screen looks alive on first load.

Build order: follow Section 14 (Phase 0 first). Do not build features marked "Later".

---

## 1. Product summary

SteelOS Ore runs the whole business of an iron ore trader and exporter on one system: buying from mines, selling to steel plants, tracking every truck from mine to plant, settling on assay results, exporting through ports, staying compliant, and syncing the books to Tally.

It replaces Excel sheets, WhatsApp photos and manual Tally entry. The product is "AI-first": AI reads documents (weighbridge slips, lab reports, transit passes, debit notes, LCs), drafts records, finds money leaks, and writes a daily decision brief for the owner.

**One-line pitch:** Know your real margin on every sauda, every truck and every shipment — the same day, not three months later.

---

## 2. How the business works (domain context the app must reflect)

### 2.1 Domestic trade (back-to-back, NO stock)
The trader is not permitted to stock ore domestically. Every domestic deal is back-to-back:

1. **Purchase sauda** with a mine (direct or via e-auction), often with an advance paid to the mine.
2. **Sale sauda** with a buyer plant (sponge iron / DRI plant, pellet plant, integrated plant) at a contract spec (Fe %, size, moisture, impurities).
3. **Trip:** each truck loads at the mine with a state transit pass. Mine weighbridge records gross/tare/net. Mine or load-point analysis is recorded.
4. Truck drives directly to the plant.
5. Plant weighs it and runs its own analysis. Plant settles on its own numbers and issues deductions (debit note) for moisture, Fe shortfall, impurities, size. Trucks can be rejected or diverted.
6. Trader pays the mine on the mine's numbers.

**The trader's margin lives in the gap between step 3 and step 5.** Transit shortage and assay disputes are the biggest leak.

Rail movement (rakes) is also possible: loading at a railway siding, weight from railway receipt (RR), sampling at loading by an appointed analyst.

**Exception — domestic warehouse holding (confirmed with customer).** A domestic trip's destination is not always the buyer plant directly. Ore is sometimes trucked mine → the trader's own warehouse, held there for a period, and later trucked warehouse → plant against a sale sauda that may be raised only after the material is already on the ground. This is a real, scoped exception to "no domestic stock" — not a modelling gap — because mines sometimes load faster than a plant can absorb, and some sales are booked opportunistically once material is already in hand. The warehouse leg still needs its own weighbridge/assay capture (in-warehouse gross/tare/net, moisture) because stored material can dry out, degrade or go missing. See §6.3 for the data model (`Trip` origin/destination generalised, new `WarehouseLot`). Warehouse stock is domestic and must not be confused with export port stock in §2.2.

### 2.2 Export trade (stock allowed at port)
Ore moves from mines to a port plot (e.g. Paradip, Haldia, Vizag, Gopalpur, Dhamra) and accumulates as port stock. It is then loaded on a vessel and sold, usually to Chinese buyers or Singapore/HK traders, on FOB or CFR terms. Price is usually index-linked with Fe adjustments and penalties. Payment by LC: provisional invoice (~90%) at loading, final settlement on discharge-port assay and weight.

### 2.3 Key regulatory facts (configurable rules, not hard-coded)
- **Export duty:** Fe below 58% → nil. Fe 58% and above → 30%. (A cargo at exactly 58.00% attracts duty.)
- **FEMA:** export proceeds must be realised within 15 months of export; goods against advance must ship within 3 years.
- **GST** on iron ore sales, e-invoice and e-way bill per movement, TCS on mineral sales, TDS on freight and brokerage.
- **State mineral rules** (Odisha rules 2025): trading licence, transit pass per truck, periodic returns. **IBM** registration and returns.

### 2.4 Sourcing channels
The mine side of the business is not just "the mine" — the trader sources ore through five distinct channels, each with its own lifecycle before it becomes a purchase sauda:

1. **E-auctions.** The trader tracks upcoming auctions (auctioneer, quantity, lot size, grade, location, date/time, EMD required, notes, uploaded auction notice) whether or not it plans to bid. Participating means paperwork submitted and EMD deposited. Outcome is win or loss:
   - **Lost:** EMD refund is requested and tracked through to refunded / not refunded, with a refund reference.
   - **Won:** becomes a purchase sauda (lot qty, rate, grade, lifting time window, conditions, payment status) and enters the same lifting/trip flow as any other purchase.
2. **Private mines** — direct negotiated purchase, same purchase-sauda terms as §6.2 (rate, payment condition, delivery condition, grade condition).
3. **Iron ore crushing units** — buys crushed/sized material; same purchase-sauda structure.
4. **Sponge iron units, as sellers of ore by-products** — some sponge iron plants also sell ore fines/rejects back into the market; same structure.
5. **Other traders** — back-to-back purchase from another trader; same structure, flagged `source_channel = other_trader` for margin-chain visibility, so the owner can see when they are buying through an intermediary and losing a margin layer.

All five channels converge into the same `PurchaseSauda` record (§6.2) — the channel is metadata (`source_channel`), not a different data shape. This is deliberate: module-per-source-type ERPs can't answer "what's my total open position" without a cross-module report; here it's always one query.

### 2.5 Sales pipeline (pre-sauda)
Sale saudas with sponge iron units and export buyers usually don't happen in one step — there is a pipeline before the contract: **Enquiry → Quotation → Negotiation → Acceptance → Sale sauda.** (Export enquiries can also go straight to negotiation before a formal quotation, and negotiation can recur after a quotation too — the pipeline supports looping, not a strict linear gate.) The pipeline is tracked so the owner can see the funnel — how many enquiries are open, how many quotations are out, what's stuck in negotiation — instead of the current state where each conversation lives only in someone's WhatsApp. Terms captured at each stage (sale date, qty, rate, payment condition, delivery condition, grade condition, and an internal note on which purchase lot is provisionally reserved against this potential order) carry forward automatically into the sale sauda on acceptance, with no re-typing. See §6.2b.

---

## 3. Goals and success metrics

| Goal | Metric |
|---|---|
| Real margin visibility | Realised margin per sauda visible within 24 h of plant receipt |
| Stop assay/weight leaks | Deductions as % of sales reduced (target < 1.0%) |
| Zero re-typing | ≥ 80% of trip data created from photos/documents by AI extraction |
| Faster decisions | Owner clears daily decision queue on phone in < 10 minutes |
| Compliance | 100% of trucks with valid transit pass + e-way bill before dispatch |
| Books | Zero manual voucher entry in Tally for trade transactions |

---

## 4. Users and roles

| Role | Where | Main device | What they do |
|---|---|---|---|
| Owner / Partner | Kolkata | Phone + desktop | Decisions, approvals, dashboard, Ask AI |
| Trading desk | Kolkata | Desktop | Create saudas, match buy/sell, penalty schedules, quotes |
| Mine-side supervisor | Barbil / Joda | Phone (offline often) | Create trips, capture loading slips, transit passes, mine analysis |
| Plant-end representative | At buyer plant | Phone | Receipt, plant weighment, witness sampling, raise disputes |
| Port coordinator | Port | Phone + desktop | Port plot stock, vessel loading, draft survey, export documents |
| Accounts | Kolkata | Desktop | Invoices, payments, receivables, Tally sync |
| CA (external) | Remote | Desktop | Review and approve filing packets only |
| Transporter / Driver | Road | WhatsApp link, no app | Confirm loading, share location, upload unloading slip |

### 4.1 Permissions (enforced server-side, not just hidden in UI)

| Data | Owner | Trading desk | Mine supervisor | Plant rep | Port coord | Accounts | CA | Transporter |
|---|---|---|---|---|---|---|---|---|
| Purchase/sale rates | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Margins | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Trips (own scope) | ✅ | ✅ | ✅ create | ✅ receive | ✅ | ✅ | ❌ | own trips only |
| Assays & weights | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ read | ❌ |
| Payments & ledgers | ✅ | read | ❌ | ❌ | ❌ | ✅ | ✅ read | own freight only |
| Filing packets | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ prepare | ✅ approve | ❌ |
| Approvals | ✅ | limited | ❌ | ❌ | ❌ | limited | ❌ | ❌ |

Approval limits are configurable per user (e.g. trading desk can confirm saudas up to ₹50 L; above that needs Owner).

---

## 5. Core concepts (glossary)

- **Sauda:** a deal/contract. Purchase sauda (with mine) or sale sauda (with plant or export buyer).
- **Match:** link between purchase sauda(s) and sale sauda(s). Unmatched tonnes = open position (risk).
- **Trip:** one truck movement from mine to plant (or mine to port). The atomic unit of the domestic business.
- **Rake:** one rail movement (many wagons) with one RR.
- **Assay:** a lab analysis result (Fe, SiO₂, Al₂O₃, S, P, moisture, size fractions, LOI).
- **Penalty schedule:** structured rules in a sale/purchase sauda that convert an assay into rupee deductions or bonuses.
- **Settlement:** the calculated amount payable for a trip/rake/lot after applying weights, assays and the penalty schedule. Two stages: provisional and final.
- **Umpire analysis:** third-party re-test when mine and plant assays differ by more than the contract threshold.
- **WMT / DMT:** wet metric tonne / dry metric tonne. DMT = WMT × (1 − moisture%/100).
- **Port lot:** a stack of export material at a port plot with its own assay.
- **EMD:** earnest money deposit paid to participate in an e-auction; refundable if the lot is lost, adjusted against payment if won.
- **Lot (auction):** a defined parcel offered at auction with its own qty, grade and location, independent of any Product master.
- **Enquiry / Quotation / Negotiation round:** the pre-contract pipeline stages for a sale, before it becomes a SaleSauda.
- **WarehouseLot:** domestic stock held at the trader's own warehouse — the one scoped exception to the "no domestic stock" rule (§2.1).
- **Blocker:** a structured, named reason a trip or a lifting schedule has stalled (payment shortfall, truck unavailability, loading unavailability) — a first-class Decision source, not a comment buried in a chat thread.

---

## 6. Data model

All tables have: `id`, `company_id` (tenant), `created_at`, `created_by`, `updated_at`, `updated_by`. All changes are written to `audit_log`.

### 6.1 Masters
**Party** — `name`, `type` (mine | plant | transporter | broker | export_buyer | auctioneer | trader | cha | bank | lab | other) — the primary role, `roles[]` (same enum, every role this party actually plays — e.g. a private mine that also trades ore on the side is `type: mine, roles: [mine, trader]`; a sponge iron unit that both buys ore and sells its own rejects is `type: plant, roles: [plant, trader]`), `gstin`, `pan`, `address`, `state`, `contacts[]` (name, phone, whatsapp, email), `bank_details`, `credit_limit`, `payment_terms_days`, `notes`.

**Mine** (extends Party) — `lease_number`, `lease_valid_till`, `ec_valid_till`, `ibm_registration_no`, `location_lat_lng`, `mineral_products[]`.

**Plant** (extends Party) — `plant_type` (DRI | pellet | integrated | other), `lab_name`, `weighbridge_name`, `default_penalty_schedule_id`.

**Product** — `name` (e.g. "CLO 10–30 mm", "CLO 5–18 mm", "Lumps 10–40 mm", "Fines 0–10 mm"), `hsn`, `typical_spec` (JSON).

**Licence** — `holder` (own company or party), `type` (state trading licence | IBM registration | IEC | GST | port permission | other), `number`, `issued_on`, `valid_till`, `document_id`, `reminder_days_before` (default 60, 30, 7).

**Vehicle** — `registration_no`, `transporter_id`, `capacity_t`, `owner_phone`.

### 6.2 Commercial
**PurchaseSauda** — `number` (P-###), `mine_id`, `product_id`, `quantity_t`, `tolerance_pct`, `rate_per_t`, `rate_basis` (WMT | DMT), `price_basis` (ex-mine | FOR siding | delivered), `includes_royalty` (bool), `spec` (JSON), `penalty_schedule_id` (optional), `advance_amount`, `delivery_from`, `delivery_to`, `status` (draft | confirmed | loading | completed | closed | cancelled), `approved_by`, `source_document_id`.

**SaleSauda** — `number` (S-###), `plant_id`, `product_id`, `quantity_t`, `tolerance_pct`, `base_rate_per_t`, `rate_basis` (WMT | DMT), `price_basis` (ex-plant delivered | FOR loading), `freight_by` (trader | buyer | mine), `penalty_schedule_id` (required), `weight_basis` (plant weighbridge | mine weighbridge | RR), `assay_basis` (plant lab | third party at unloading | load-point analyst), `umpire_threshold_pct` (default 0.5), `sample_retention_days` (default 90), `provisional_payment_pct` (default 90), `payment_terms_days`, `status`, `approved_by`.

**SaudaMatch** — `purchase_sauda_id`, `sale_sauda_id`, `quantity_t`. Position view computes: bought-unsold tonnes and sold-unsourced tonnes.

**PenaltySchedule** — `name`, `lines[]`:
```
{
  parameter: "Fe" | "SiO2" | "Al2O3" | "S" | "P" | "Moisture" | "Oversize" | "Undersize" | "LOI" | custom,
  direction: "min" | "max",            // Fe is min; impurities are max
  basis_value: number,                 // e.g. Fe 63.0, SiO2 3.5
  bands: [
    { from: number, to: number,
      rate_type: "pct_of_base_price" | "rs_per_t_per_step" | "rs_per_t_on_excess_qty" | "qty_deduction_at_actual",
      rate: number, step: number,      // e.g. step 0.1 for SiO2 at ₹2.30
      pro_rata: true }
  ],
  bonus: { rate_type, rate, step, cap } | null,
  rejection_limit: number | null,      // e.g. Fe < 61 → rejected
  rejection_price_per_t: number | null,// e.g. token Re 1/t
  applies_on_qty: "shipped" | "payable"
}
```
Also add to **PurchaseSauda**: `source_channel` (auction | private_mine | crushing_unit | sponge_iron_unit | other_trader), `source_auction_participation_id` (nullable, set when `source_channel = auction`).

### 6.2a Sourcing pipeline (pre-purchase)
**Auction** — `auctioneer_id`, `title`, `product_id`, `quantity_t`, `lot_size_t`, `grade_spec` (JSON), `location`, `auction_at`, `emd_required_amount`, `notes`, `document_ids[]` (auction notice), `status` (upcoming | participating | not_participating | won | lost | closed).

**AuctionParticipation** — `auction_id`, `paperwork_submitted_at`, `emd_amount`, `emd_paid_at`, `emd_document_id`, `outcome` (pending | won | lost), `won_lot` (qty, rate, grade, lifting_from, lifting_to, conditions, purchase_sauda_id — set once the win is converted to a purchase sauda), `emd_refund` (requested_at, refunded_at, amount, reference, status: pending | refunded | not_refunded).

### 6.2b Sales pipeline (pre-sauda)
**Enquiry** — `party_id`, `product_id`, `quantity_t`, `source` (whatsapp | call | email | visit), `notes`, `status` (open | quoted | negotiating | accepted | lost | closed).

**Quotation** — `enquiry_id`, `rate_per_t`, `payment_condition` (advance | credit | on_delivery), `delivery_condition` (EX | FOR), `grade_condition` (JSON: Fe%, deduction terms), `valid_till`, `revision_no`.

**NegotiationRound** — `enquiry_id`, `quotation_id` (optional — negotiation can precede a formal quotation, e.g. for export buyers), `at`, `notes`, `counter_rate`, `by` (us | buyer).

On acceptance, an Enquiry converts to a `SaleSauda` (§6.2), carrying forward rate, payment condition, delivery condition and grade condition. `SaleSauda` gains two fields: `source_enquiry_id` (link back for the full trail) and `internal_reserved_purchase_sauda_id` (which purchase lot the trading desk earmarked against this deal while it was still being negotiated).

### 6.3 Movement
**Trip** — `number`, `sale_sauda_id`, `purchase_sauda_id`, `vehicle_id`, `transporter_id`, `driver_name`, `driver_phone`, `transit_pass_no`, `transit_pass_valid_till`, `eway_bill_no`, `eway_bill_valid_till`, `mine_gross_t`, `mine_tare_t`, `mine_net_t`, `mine_weighbridge_slip_doc_id`, `loaded_at`, `loading_photo_doc_ids[]` (geo+time stamped), `dispatched_at`, `arrived_at`, `plant_gross_t`, `plant_tare_t`, `plant_net_t`, `plant_weighment_doc_id`, `load_assay_id`, `plant_assay_id`, `shortage_t` (computed), `shortage_pct` (computed), `freight_rate_per_t`, `freight_amount`, `status` (loading | dispatched | in_transit | arrived | received | settled | disputed | rejected | diverted), `delay_flag` (computed), `last_location` (from WhatsApp share), `origin_type` (mine | warehouse), `origin_warehouse_id` (nullable), `destination_type` (plant | warehouse | port), `destination_warehouse_id` (nullable).

The back-to-back mine→plant trip (§2.1) is `origin_type: mine, destination_type: plant` and remains the default, by far the most common row. A trip into the trader's own warehouse (`destination_type: warehouse`) creates or tops up a `WarehouseLot`; a trip out of the warehouse to a plant (`origin_type: warehouse`) draws down that lot and does not re-trigger mine-side transit-pass checks (it needs its own local movement permit per state rules — configurable). This keeps `Trip` the single atomic unit end to end instead of splitting "receipt" and "dispatch" into separate modules.

**WarehouseLot** (the domestic stock exception, §2.1) — `warehouse_location`, `product_id`, `source_purchase_sauda_ids[]`, `quantity_in_t`, `quantity_out_t`, `quantity_on_hand_t` (computed), `received_assay_id`, `moisture_loss_allowance_pct` (configurable, default 0), `held_since`, `max_hold_days` (alert if exceeded — configurable per company policy), `status` (open | depleted).

**Rake** — `number`, `siding`, `rr_number`, `rr_weight_t`, `wagons_count`, `indent_date`, `placement_at`, `loading_completed_at`, `demurrage_hours`, `trips[]` (feeder trucks), `sale_sauda_id`, `load_assay_id`, `plant_assay_id`, `status`.

**Assay** — `sample_ref`, `taken_at`, `taken_by` (mine | plant | third party | umpire), `lab_name`, `fe`, `sio2`, `al2o3`, `s`, `p`, `loi`, `moisture`, `oversize_pct`, `undersize_pct`, `basis` (dry | as_received), `seal_numbers[]`, `document_id`, `extracted_by_ai` (bool), `verified_by`.

### 6.4 Settlement and disputes
**Settlement** — `scope` (trip | rake | lot | shipment), `scope_id`, `stage` (provisional | final), `qty_shipped_t`, `qty_payable_t`, `base_rate`, `lines[]` (parameter, actual, limit, deduction_rs, working_text), `gross_amount`, `total_deductions`, `bonus`, `net_amount`, `status` (draft | sent | agreed | disputed | closed).

**DebitNote** (received from plant) — `plant_id`, `number`, `date`, `amount`, `lines[]`, `document_id`, `system_settlement_id`, `variance_rs` (computed), `verdict` (matches | over_deducted | under_deducted).

**Dispute** — `settlement_id`, `reason`, `mine_value`, `plant_value`, `variance`, `umpire_requested_at`, `umpire_lab`, `umpire_assay_id`, `outcome`, `cost_borne_by` (per contract rule: party more remote from umpire pays; equidistant → split), `sample_expiry_date`.

### 6.5 Export
**PortPlot** — `port`, `plot_no`, `permission_doc_id`, `valid_till`.

**PortLot** — `plot_id`, `quantity_wmt`, `quantity_dmt`, `assay_id`, `fe_dry`, `fe_as_received`, `source_trip_ids[]`, `duty_band` (computed).

**ExportContract** — `number` (E-####), `buyer_id`, `incoterm` (FOB | CFR), `load_port`, `discharge_port`, `quantity_dmt`, `tolerance_pct`, `price_formula` (JSON: index name, quotational period, Fe basis, per-unit adjustment, penalties, fixed price option), `currency`, `lc_id`, `provisional_pct`, `status`.

**Shipment** — `export_contract_id`, `vessel_name`, `eta`, `laycan_from`, `laycan_to`, `laytime_terms`, `demurrage_rate`, `despatch_rate`, `nor_tendered_at`, `loading_started_at`, `loading_completed_at`, `bl_number`, `bl_date`, `draft_survey_qty`, `load_port_assay_id`, `discharge_assay_id`, `discharge_qty`, `shipping_bill_no`, `shipping_bill_date`, `export_duty_amount`, `realisation_deadline` (computed = shipping_bill_date + 15 months).

**LC** — `number`, `issuing_bank`, `advising_bank`, `amount`, `currency`, `expiry_date`, `latest_shipment_date`, `presentation_days`, `required_documents[]` (name, originals, copies, special conditions), `terms_text`, `document_id`.

**ExportDocument** — `shipment_id`, `type` (commercial invoice | B/L | certificate of analysis | certificate of weight | certificate of origin | draft survey | packing list | insurance | other), `document_id`, `lc_check_status` (not_checked | ok | discrepancy), `discrepancies[]`.

### 6.6 Money
**Invoice** (sale), **PurchaseBill**, **Payment** (in/out, mode, bank ref), **Advance** (to mine, against purchase sauda, adjusted per trip), **FreightBill** (transporter, TDS), **BrokerageBill**, **BankRealisation** (FIRC/e-BRC reference, amount, date, linked shipping bill).

### 6.7 Compliance, AI, system
**ComplianceRule** — `code`, `name`, `jurisdiction` (India | Odisha | Jharkhand | West Bengal | port | FEMA | destination), `applies_to` (trip | sauda | shipment | filing | licence), `condition` (JSON expression), `action` (block | warn | prepare), `message`, `source_reference`, `last_verified_on`, `verified_by`, `active`.

**FilingPacket** — `type` (GSTR-1 | GSTR-3B | TDS | TCS | mining return | IBM return | other), `period`, `data_snapshot`, `reconciliation_issues[]`, `status` (preparing | ready | with_CA | returned | approved | filed), `ca_comments`, `approved_by`, `filed_at`, `acknowledgement_doc_id`.

**Document** — `file_url`, `type`, `uploaded_by`, `uploaded_via` (app | WhatsApp | email | bulk import), `geo`, `captured_at`, `ai_extraction` (JSON with field-level confidence), `linked_records[]`.

**Decision** — `type`, `title`, `context_text`, `money_at_stake_rs`, `severity` (high | medium), `primary_action` (label, handler), `secondary_action` (label, handler), `source_record`, `status` (open | done | snoozed), `due_at`.

**Alert** — `type`, `record`, `message`, `severity`, `channel` (in-app | push | WhatsApp), `read_at`.

**Blocker** — a structured reason something in the lifting/dispatch pipeline has stalled, so it becomes a Decision source instead of a comment buried in a chat thread. `scope` (trip | auction_participation | purchase_sauda), `scope_id`, `type` (payment_shortfall | truck_unavailable | loading_unavailable | other), `detail_text` (e.g. "need to deposit ₹X for more dispatch", "trucks not available with the transporter", "loading not available today / since 15 Sep due to ___"), `raised_by`, `raised_at`, `resolved_at`, `money_to_unblock_rs` (nullable, for payment_shortfall).

**AuditLog** — `table`, `record_id`, `action`, `before`, `after`, `user_id`, `at`, `device`, `geo`.

---

## 7. Calculation engines (exact rules)

### 7.1 Trip shortage
```
shortage_t   = mine_net_t − plant_net_t
shortage_pct = shortage_t / mine_net_t × 100
```
Alert if `shortage_pct` > company threshold (default 0.75%).

### 7.2 Settlement engine
For each line in the penalty schedule:
1. Read the actual value from the governing assay (per `assay_basis`).
2. If direction = min (e.g. Fe) and actual < basis → penalty; if actual > basis and bonus exists → bonus.
3. If direction = max (impurities, moisture, size) and actual > basis → penalty.
4. Apply bands pro-rata. Steps are fractional when `pro_rata = true`.
5. Rejection: if actual crosses `rejection_limit` → mark rejected, price = `rejection_price_per_t`.
6. Moisture with `qty_deduction_at_actual`: `excess_qty = qty_shipped × (moisture − limit)/100`; `qty_payable = qty_shipped − excess_qty`; value lost = `excess_qty × base_rate`.
7. Price deductions are applied on `applies_on_qty` (shipped or payable).

Output: a settlement with one line per parameter showing actual, limit, working (in plain words), and rupees.

**Mandatory test case (must produce exactly these numbers):**
Base rate ₹5,000/WMT · shipped 4,000 WMT · penalties applied on shipped quantity.
Schedule: Fe basis 63%, 63→62: 2% of base per 1% pro-rata; 62→61: 4% of base per 1%; < 61 rejected. SiO₂ max 3.5%, ₹2.30/t per 0.1%. Al₂O₃ max 2.5%, ₹2.30/t per 0.1%. S max 0.02%, ₹5/t per 0.01%. P max 0.05%, ₹5/t per 0.01%. Moisture max 3.5%, quantity deduction at actual. Oversize +40 mm max 10%, ₹23/t on excess qty. Undersize −10 mm max 10%, ₹30/t on excess qty.
Assay: Fe 62.4, SiO₂ 3.8, Al₂O₃ 2.7, S 0.02, P 0.05, moisture 4.2, oversize 8, undersize 13.

| Line | Working | Deduction |
|---|---|---|
| Fe | 0.6 × 2% = 1.2% of ₹5,000 = ₹60/t × 4,000 | ₹2,40,000 |
| SiO₂ | 0.3% over = 3 steps × ₹2.30 = ₹6.90/t × 4,000 | ₹27,600 |
| Al₂O₃ | 0.2% over = 2 × ₹2.30 = ₹4.60/t × 4,000 | ₹18,400 |
| Undersize | 3% excess × 4,000 = 120 t × ₹30 | ₹3,600 |
| Moisture | 0.7% × 4,000 = 28 t × ₹5,000 | ₹1,40,000 |
| **Total** | | **₹4,29,600** |

Gross ₹2,00,00,000 → Net ₹1,95,70,400 (2.15% deducted).

### 7.3 Debit note checker
Compare the plant's debit note line by line with the system settlement. Verdict per line: matches (±₹1), over-deducted, under-deducted. Over-deduction creates a Decision: "Dispute ₹X with [plant]".

### 7.4 Umpire trigger
If `|plant_value − load_value| ≥ umpire_threshold_pct` for Fe (or any parameter flagged in the sauda) → create Decision "Request umpire analysis" with money at stake = deduction attributable to the variance, and deadline = sample expiry date.

### 7.5 Margin
```
Estimated margin per t (at sauda match) = sale_base_rate − purchase_rate − freight_per_t − expected_deduction_per_t − finance_cost_per_t
Realised margin per t (after settlement) = net_settlement_per_t − purchase_cost_per_t − actual_freight_per_t − finance_cost_per_t
finance_cost_per_t = capital_blocked × annual_rate × days_blocked / 365 / tonnes
```
`expected_deduction_per_t` = rolling average for that plant + product (last 90 days). `annual_rate` configurable (default 12%).

### 7.6 Export duty
```
if fe_value >= 58.00 → duty_rate = 30% else 0%     (configurable rule)
```
Store both `fe_dry` and `fe_as_received` on every port lot; the rule uses the basis configured by the company (confirm with CHA). Warn when the lot's Fe is between 57.5% and 58.5%.

### 7.7 Export price formula (configurable)
```
price_per_dmt = index_price_avg(QP) × (actual_Fe / basis_Fe)  [or ± per-unit adjustment]
                − penalties (SiO2, Al2O3, P, moisture above limits)
provisional_invoice = provisional_pct × qty_BL_dmt × provisional_price
final_invoice       = final_price × discharge_qty_dmt − provisional_paid
```
Index values entered manually (Later: licensed feed).

### 7.8 Laytime / demurrage
Laytime clock from NOR + turn time, excluding excepted periods entered by the port coordinator. `demurrage = (time_used − laytime_allowed) × demurrage_rate`; despatch if negative (per contract).

### 7.9 Lifting countdown and EMD lifecycle
```
days_left    = lifting_deadline − today     // auction/purchase lifting window
days_excess  = today − lifting_deadline     // only when today > lifting_deadline
```
Both are shown per won auction lot / purchase sauda while material is still being lifted, alongside any open `Blocker`. `AuctionParticipation.emd_refund.status` starts `pending` the moment an auction is marked lost; alert if it stays `pending` for more than 30 days (configurable).

---

## 8. Modules and features

### 8.1 Owner dashboard (highest priority — see Section 9)

### 8.2 Saudas
- List with filters (type, party, product, status, month) and quick totals.
- Create sauda form + **"Paste WhatsApp / upload PO" → AI draft** (party, product, qty, rate, spec, penalty clauses pre-filled; user confirms).
- Penalty schedule builder with templates per plant; preview "what if assay = X" calculator.
- Match screen: drag purchase saudas to sale saudas; show unmatched tonnes both sides.
- Sauda detail: header, progress bar (dispatched / received / settled), trips table, settlements, debit notes, disputes, payments, margin est vs realised, documents, activity log.
- Approval flow above limits.
- **Acceptance:** creating a sale sauda without a penalty schedule is blocked; sauda above user's limit goes to Owner decision queue.

### 8.3 Trips
- Create trip (mobile, < 30 seconds): scan transit pass → AI fills TP no., quantity, validity; photo of weighbridge slip → AI fills vehicle, gross, tare, net; auto-attach geotagged loading photo; select sale sauda (only valid, open saudas shown).
- Block dispatch if transit pass or e-way bill missing/invalid/expired.
- Live list with statuses, delay flags (in transit > configurable hours), map pins from WhatsApp location.
- Plant receipt (mobile): plant weighment photo → AI extraction; shortage computed instantly; plant assay upload → settlement auto-calculated.
- Bulk actions: attach one assay to many trips (lot-wise sampling).
- **Acceptance:** after plant assay is entered, settlement appears within 5 seconds with per-line working.

### 8.4 Rakes (Later in Phase 1 if customer uses rail)
Indent, placement, loading, RR, demurrage, feeder trucks, rake-level assay and settlement.

### 8.5 Settlements, debit notes and disputes
- Settlement list by stage (provisional / final), status, plant.
- Upload plant debit note → AI extracts lines → checker verdict.
- Dispute workspace: evidence (photos, seal numbers, assays), umpire request letter auto-drafted, deadline countdown, outcome entry, cost allocation rule applied automatically.

### 8.6 Export desk
- Port plots and port lots (stock view by plot with Fe and duty band).
- Export contracts with price formula builder.
- Shipment tracker: laycan, NOR, loading progress, laytime/demurrage clock.
- LC module: upload LC → AI extracts required documents and conditions → checklist.
- **LC document checker:** each uploaded export document is cross-checked by AI against LC terms and other documents (names, quantities, dates, ports, descriptions). Discrepancies listed before bank presentation.
- Provisional and final invoicing; claims from discharge-port results.
- Realisation tracker per shipping bill with 15-month deadline.

### 8.7 Money
- Receivables aging by plant with one-tap WhatsApp payment reminder.
- Mine advances: balance per mine, adjusted per trip, alerts if mine is behind loading schedule.
- Freight: transporter-wise trips, rates, shortage recoveries, TDS, payable.
- Brokerage.
- Cash forecast (next 30/60/90 days from receivable terms, advances, freight, export realisation).

### 8.8 Compliance centre
- Rules engine (see ComplianceRule). Each rule shows source and last-verified date.
- Per-transaction checks: transit pass, e-way bill, e-invoice, mine lease/clearance validity, party GSTIN active, export duty band, sanctions screening of buyer/vessel (manual flag in Phase 1).
- Compliance calendar: filings due, licence expiries, LC expiries, FEMA deadlines.
- **Filing workflow: Prepare → Check → CA review → Approve → File → Acknowledge.** The system never files without CA approval. CA gets a restricted login (filings, reconciliations and supporting documents only; no rates or margins).
- Real-time documents (e-way bill, e-invoice) are generated by operations when all checks pass; periodic filings go through CA.

### 8.9 Documents & AI inbox
- Central inbox for everything uploaded (app, WhatsApp forward, email forward).
- AI classifies the document type, extracts fields with confidence per field, suggests the record to link.
- Low-confidence fields highlighted for human confirmation; source image shown side by side.

### 8.10 Import existing data
- Upload Excel/CSV (sauda registers, trip registers, lab sheets, party lists, outstanding).
- AI column-mapping wizard: detects columns, maps to fields, shows preview, flags bad rows, imports with a batch id (undo by batch).
- Upload old PDFs/photos in bulk → AI inbox.

### 8.11 Tally integration
- Two-way with Tally Prime (XML over HTTP via a small connector running on the customer's Tally machine).
- Push: party ledgers, sales invoices, purchase bills, debit/credit notes, receipts, payments, freight bills with TDS, journal entries.
- Pull: ledger balances for reconciliation.
- Sync log with status per voucher; mismatch report.
- SteelOS is the operational truth; Tally remains the statutory book.

### 8.12 Ask AI (natural language)
- Search bar on desktop, "Ask" tab on mobile, voice input. Hindi, English and Hinglish.
- Answers are computed from live data and always cite the records used (clickable).
- Examples it must handle:
  - "Is mahine kis plant ne sabse zyada deduction kiya?"
  - "Mine X ka average Fe last 3 months?"
  - "Kalinga DRI ka outstanding kitna hai aur kitna 30 din se purana hai?"
  - "Which transporter has the worst shortage this month?"
  - "Realised vs estimated margin on S-2031?"
  - "Which export bills are closest to the FEMA deadline?"

### 8.13 Settings
Company profile, users and roles, approval limits, thresholds (shortage %, delay hours, umpire variance default, near-duty band), finance cost rate, number series, penalty templates, compliance rules, notification preferences, Tally connection.

---

## 9. Dashboard specification (prime screen)

Design principle: **every section either asks for a decision or explains one.** Reads top to bottom in the order the owner thinks. A reference design exists (desktop 1440 px + phone 390 px) — match its structure and style.

### 9.1 Desktop layout (top to bottom)
1. **Top bar:** SteelOS **Ore** logo · company name · Ask AI search bar (with ⌘K) · date · role/user menu.
2. **Morning brief:** one AI-generated sentence in large serif type, e.g. "₹4.3 lakh was deducted on yesterday's settlements — ₹3.1 lakh of it by one plant. Four decisions need you today." Buttons: "Read full brief", "Listen · 2 min".
3. **Row: Needs your decision (8/12 width) + Money map (4/12, dark card).**
   - Decision rows sorted by money at stake. Each row: severity dot, title, one-line context, amount + label, primary button (accent) + secondary button (outline). Actions work in place.
   - Decision sources: unmatched position above threshold; umpire trigger; over-deducted debit note; LC expiring ≤ 10 days with pending documents; advance approval; sauda above limit; trip rejection risk (assay near rejection limit); filing returned by CA; licence expiring ≤ 30 days.
   - Money map: working capital deployed (big number), stacked bar, rows for receivables (with % older than 30 days), advances with mines, domestic warehouse stock value at cost (only shown if the company holds any — the scoped exception to no-stock, see §2.1), export proceeds to realise (with nearest FEMA deadline), freight payable.
4. **Five vital numbers** (tiles with sparkline): dispatched today (t, trucks, vs plan) · in transit (t, trucks, delayed count) · realised margin MTD ₹/t vs estimated · deductions MTD % of sales vs target · average transit shortage % and worst transporter.
5. **Row: Sauda health (8/12) + Where money is leaking (4/12).**
   - Sauda health table: sauda + party + product/qty · layered progress bar (dispatched / received / settled) · margin est → realised · status pill (On track / Deductions high / Final settlement due / Loading ready…).
   - Leak panel: deductions by plant MTD (bars), transit shortage by transporter (bars), one **AI finding** box (pattern detected, e.g. plant lab reads low vs load-point on 9 of 11 rakes → suggest joint-sampling clause).
6. **Row: Trips needing attention (7/12) + Compliance (5/12).**
   - Only exceptions: delayed, shortage above threshold, rejection risk. Link to all trips.
   - Compliance: 4 status tiles (transit passes & e-way bills; current GST filing status; licence renewal; export realisation), each green/amber/red with one line.
7. **Row: Sourcing pipeline (6/12) + Sales pipeline funnel (6/12).**
   - Sourcing pipeline: upcoming auctions (auctioneer, lot, date, EMD required), auctions currently participating (total EMD at risk), and EMD refunds pending beyond 30 days. Each links straight to the Auction / AuctionParticipation record.
   - Sales pipeline funnel: counts at each stage (Enquiry → Quotation → Negotiation → Accepted this month), with the oldest open enquiry flagged so nothing goes stale in someone's WhatsApp.
8. **Row: Lifting board (12/12).**
   - One row per won auction lot / purchase sauda currently being lifted: material lifted so far vs total, days left / days excess (red once excess), and any open Blocker with its money-to-unblock amount. This is the truck-by-truck lifting view from the mine side — the same shape as "Trips needing attention" but scoped to lifting-schedule risk rather than in-transit risk.

### 9.2 Phone layout (owner)
Logo + date · "Suprabhat · Morning brief" + one-sentence brief (serif) · three number tiles (dispatched, margin MTD, deductions) · "Needs your decision" cards with two 44 px buttons each · dark card with working capital + compliance items needing attention · bottom nav: Home, Trips, Saudas, Ask, More.

### 9.3 Behaviour
- Every number is clickable → filtered list of underlying records.
- Dashboard refreshes live (new trip, settlement, payment updates within seconds).
- Role-aware: non-owners see a version without rates/margins/money map.
- Period switcher: Today / MTD / QTD / custom.

---

## 10. Mobile app (PWA) by role

**Owner:** morning brief, decisions with one-tap approve/hold, sauda health, money map, Ask by voice, push alerts (assay near rejection, truck delayed, shortage above threshold, debit note received, LC expiring, index moved sharply).

**Mine supervisor:** create trip in 30 s (photo → AI), mine analysis upload, today's plan vs loaded per sauda, mark a trip's destination as the own warehouse when the plant isn't ready to receive yet, and raise a Blocker (payment shortfall / truck unavailable / loading unavailable) instead of just texting the owner. **Offline-first:** queue records and photos locally, sync when network returns, show sync status.

**Plant representative:** mark arrival, plant weighment photo, sampling witness (photos, seal numbers, time), plant lab report upload → auto-penalty, raise dispute with evidence, umpire request if variance crosses threshold.

**Trading desk:** WhatsApp text → draft sauda, match buy/sell, quote check (mine rate + freight + expected deduction = net margin), penalty templates, track auction participation and EMD, log enquiries/quotations/negotiation rounds so a live conversation never lives only in someone's phone.

**Port coordinator:** port plot receipts and stock, draft survey and assay capture, loading progress, laytime clock, document checklist with photo capture checked against LC.

**Accounts:** payment approvals, WhatsApp reminders, Tally sync status.

**Transporter/driver (no app):** per-trip WhatsApp link: confirm loading, share location, upload unloading slip. Transporter link shows own trips and freight due only.

**All roles:** photo → AI extraction; GPS + time stamp on every photo; every edit logged; Hindi and English UI; voice input.

---

## 11. AI features (summary and guardrails)

| Feature | Input | Output |
|---|---|---|
| Document extraction | Photo/PDF of transit pass, weighbridge slip, lab report, debit note, invoice, LC, B/L, draft survey, CoA, CoO | Structured fields with confidence; linked record suggestion |
| WhatsApp → sauda | Pasted chat text | Draft sauda |
| Morning brief | Last 24 h data | One-sentence headline + full brief + audio |
| Decision queue | All open exceptions | Ranked decisions with money at stake |
| Debit note checker | Plant debit note | Line-by-line verdict |
| LC document checker | LC + export documents | Discrepancy list |
| Leak finder | Trips, assays, settlements | Patterns (plant lab bias, transporter shortage, mine grade drift) |
| Ask | Natural language (Hindi/English/Hinglish, voice) | Answer + cited records |
| Import mapper | Excel/CSV | Column mapping + validation |

**Guardrails:** AI never posts money, confirms saudas, files returns or sends messages to external parties without a human action. Every AI-extracted value shows its source image and confidence. Every AI answer shows the records it used. AI suggestions are logged.

---

## 12. Integrations

| System | Purpose | Phase |
|---|---|---|
| Tally Prime (XML/HTTP connector) | Push vouchers/masters, pull balances | 1 |
| GST e-invoice & e-way bill (via GSP API) | Generate per trip/invoice | 1 |
| WhatsApp Business API | Driver links, reminders, document forwarding in, alerts out | 1 |
| Email ingestion | Forward documents into AI inbox | 1 |
| State mineral portal (i3MS etc.) | Transit pass data | Manual entry/photo in 1; API Later |
| Bank statement import (CSV/PDF) | Payment reconciliation | 2 |
| Index prices (licensed feed) | Export price formula | Later |
| ICEGATE / CHA systems | Shipping bill data | Later |

---

## 13. Non-functional requirements

- **Indian formatting:** ₹ with lakh/crore grouping; dates DD MMM YYYY; tonnes with 2–3 decimals as configured.
- **Offline-first mobile** for mine and plant roles.
- **Security:** role-based access enforced on the server; field-level redaction of rates/margins; tenant isolation.
- **Audit:** immutable audit log for every create/update/delete with user, time, device, geo.
- **Performance:** dashboard loads < 2 s with 12 months of data; trip creation works on 2G/3G.
- **Languages:** English and Hindi UI; AI understands Hinglish; Odia labels Later.
- **Accessibility:** touch targets ≥ 44 px; text contrast ≥ 4.5:1.
- **Backups** daily; export of all data to Excel anytime.

### 13.1 Design system (match the reference dashboard)
- Background: warm stone `#F4F1EC`; cards white `#FFFFFF` with border `#E4DFD6`, radius 14–16 px.
- Ink: `#16181D`; secondary text `#5B5A55`; muted `#6B6A63`.
- Accent (hematite red): `#A63D2A` for primary buttons and key highlights.
- Status: good `#1F5E51` on `#DDEEE8`; warning `#6A4A00` on `#F6E7C4`; bad `#8E2A1B` on `#F7E1DC`.
- Dark card (money map): `#16181D` background, `#F4F1EC` text.
- Fonts: Instrument Serif (brief headline), Instrument Sans (UI), IBM Plex Mono (all numbers, so figures align).
- No gradients, no emoji, generous whitespace, one accent colour.

---

## 14. Phasing

### Phase 0 — Demo build (build this first)
Seeded, clickable, realistic:
1. Owner dashboard (desktop) exactly per Section 9.
2. Owner phone brief.
3. Mine-side trip creation on phone (with simulated AI extraction from a sample slip image).
4. Plant-end receipt on phone with auto-settlement using the Section 7.2 test case.
5. Sauda list + sauda detail (S-2031).
6. Settlement detail showing per-line working.
7. Ask AI answering the example questions from seed data.

### Phase 1 — MVP (domestic)
Masters, saudas, penalty schedules, matching, trips, settlements, debit notes, disputes, advances, receivables, freight, compliance checks per trip, CA filing workflow, Tally push, WhatsApp links, AI inbox, import wizard, dashboard live.

### Phase 2 — Export
Port plots/lots, export contracts, price formula, shipments, laytime, LC module and document checker, realisation tracker, export duty rule, provisional/final invoicing.

### Later
Rakes (if needed earlier, move to Phase 1), index feeds, portal APIs, bank reconciliation automation, sanctions screening API, Odia language, hedging/FX tools.

---

## 15. Seed data (use for demo; all names fictional)

**Company:** Ganesha Steel, Kolkata.

**Mines:** Barbil Minerals (Barbil), Mahanadi Mines (Joda), Keonjhar Ores (Koira).
**Plants:** Kalinga DRI (Jajpur, DRI), Utkal Pellets (Barbil, pellet), Shree Sponge (Rourkela, DRI).
**Transporters:** Maa Tarini Roadways, Barbil Carriers, Jajpur Logistics.
**Export buyer:** Pacific Ore Trading Pte Ltd (Singapore). **Vessel:** MV Ocean Grace. **Port:** Paradip.

**Saudas:**
| Sauda | Party | Product | Qty | Rates | Progress | Margin est → real | Status |
|---|---|---|---|---|---|---|---|
| S-2031 | Kalinga DRI | CLO 10–30 mm | 12,000 t | base ₹5,000/WMT | 78% dispatched / 64% received / 40% settled | ₹260 → ₹231 /t | Deductions high |
| S-2027 | Utkal Pellets | Fines 62% | 20,000 t | — | 55 / 51 / 38 | ₹180 → ₹186 /t | On track |
| S-2024 | Shree Sponge | CLO 5–18 mm | 6,000 t | — | 100 / 100 / 72 | ₹310 → ₹294 /t | Final settlement due |
| E-0412 | Pacific Ore (CFR China) | Fines 56.5% | 55,000 t | nil duty | 41,200 t at Paradip plot | $6.10/t est | Loading ready |
| P-118 | Barbil Minerals | CLO | 4,500 t unmatched | ₹4,850/t | — | — | Open position |
| P-121 | Mahanadi Mines | CLO 10–30 mm | 3,000 t | advance ₹25 L | — | — | Awaiting approval |

**Dashboard figures:** deducted yesterday ₹4.3 L (₹3.1 L by Kalinga DRI) · dispatched today 2,640 t / 94 trucks / plan 3,000 t · in transit 1,180 t / 41 trucks / 3 delayed · realised margin MTD ₹212/t vs ₹248/t · deductions MTD 1.9% · avg shortage 0.74%, worst 1.8% (Maa Tarini) · working capital ₹11.9 Cr: receivables ₹6.42 Cr (18% > 30 days), mine advances ₹3.15 Cr, export proceeds $1.86 M (next FEMA deadline 214 days), freight payable ₹0.84 Cr.

**Decisions:** (1) 4,500 t unsold, P-118, exposure ₹2.18 Cr — [Show matching buyers] [Set price alert]; (2) Umpire R-2291 Kalinga DRI, Fe 61.9 vs 62.8, ₹1.84 L at stake, sample expires in 81 days — [Request umpire] [Accept debit note]; (3) LC MV Ocean Grace expires in 6 days, 5 of 7 documents checked, $3.92 M — [Open document check] [Draft extension request]; (4) Advance ₹25 L to Mahanadi Mines, P-121 — [Approve] [Hold].

**Trips needing attention:** OD09 AB 4412 Barbil → Kalinga DRI, in transit 11 h 20 m (Delayed); OD14 K 7730 Joda → Utkal Pellets, short 2.4% (Shortage); OD09 Q 1185 Barbil → Shree Sponge, Fe 60.8% below 61% (Rejection risk).

**Compliance:** 94/94 transit passes & e-way bills valid · GSTR-1 September with CA, 2 mismatches · trading licence expires in 41 days · 0 export bills near 15-month limit.

**AI finding:** Kalinga's lab read Fe lower than the load-point on 9 of the last 11 rakes, by 0.6% on average — suggest a joint-sampling clause in the next sauda.

---

## 16. Out of scope (v1)
Mining operations, crushing/screening/blending at domestic depots, production modules, payroll/HR, full statutory accounting (Tally remains the book), automatic filing without CA approval, trading on exchanges.

## 17. Open questions (to confirm with customer after demo)
1. Who arranges and pays truck freight — trader, mine or plant?
2. Do private plant contracts use full penalty schedules like PSU tenders, or only Fe + moisture + size?
3. Road vs rail share of domestic volume?
4. Export terms: FOB or CFR? Do they charter vessels? Which ports?
5. Is blending done at the port plot?
6. For the 58% duty test, dry or as-received Fe (confirm with CHA)?
7. Current high-grade (≥ 64% Fe) export canalisation status (confirm with CHA).
8. Monthly volume, number of active parties, number of users by role.
9. Tally version and where it runs.
10. Which filings the CA currently handles and in what format.
11. How long does domestic warehouse stock typically sit before moving to a plant — hours, days, weeks? Does it need its own moisture/shrinkage allowance (§6.3, `WarehouseLot.moisture_loss_allowance_pct`)?
12. Should EMD-at-risk (deposited, auction outcome not yet known) count against working capital in the money map, or only once a loss is confirmed and refund is pending?
