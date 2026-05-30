# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Media Rights Management · Created: 2026-05-26

## Philosophy

Every state change in the rights platform — title catalogued, right granted, contract executed, sales imported, royalty calculated, statement issued, advance recouped — is recorded as an immutable event in a single append-only store. The event store is the sole source of truth; five materialised read models are projected from the event stream to serve the catalogue, avails, contract status, royalty dashboard, and sales reconciliation views.

Media rights management has a critical need for provenance and auditability. Rights disputes hinge on proving who granted what to whom, when, and whether a conflicting grant existed at the time. Royalty audits require reconstructing exactly how a calculation was derived from source sales data through rule application. With event sourcing, the complete chain of custody — from title registration through rights grants, sales ingestion, royalty calculation, to statement delivery — is an immutable ledger. Every state can be reconstructed at any point in time, which is precisely what auditors and legal teams need.

The CQRS pattern separates the high-throughput write path (DSP sales data ingestion at millions of rows per import, contract amendments, rights grants) from the read-optimised catalogue and dashboard views. Read models can be rebuilt from events when new analytics requirements emerge — essential in a domain where regulatory interpretations and royalty calculation methods evolve.

**Best for:** Teams building a rights platform where full audit trail provenance, rights-dispute resolution via temporal replay, royalty audit transparency, and high-throughput sales ingestion are priorities.

**Trade-offs:**
- Pro: Complete provenance chain for rights disputes — who granted what, when
- Pro: Royalty calculation audit trail is the event stream itself — every step reconstructable
- Pro: Temporal queries ("what rights were active on date X?") by replaying events
- Pro: Read models can be rebuilt when reporting requirements change
- Con: CQRS adds infrastructure complexity — event store + projections + read models
- Con: Eventual consistency between event writes and read model updates
- Con: High-volume sales events require careful partitioning and snapshot strategy
- Con: Conflict detection across events requires materialised avails view

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (ce_source, ce_type, ce_specversion, ce_time) |
| W3C ODRL 2.2 | right_granted events carry ODRL-aligned permission structure |
| W3C PROV-O | Event chain provides provenance ontology for rights-of-record |
| ISO 15706 (ISAN) | title_registered events carry ISAN for audiovisual works |
| ISO 3901 (ISRC) | title_registered events carry ISRC for recordings |
| ISO 15707 (ISWC) | title_registered events carry ISWC for compositions |
| ISO 27729 (ISNI) | Party identification in contributor and contract events |
| ISO 3166 | Territory codes in rights events |
| ISO 4217 | Currency codes in financial events |
| DDEX ERN/DSR | sales_imported events reference DDEX report format |
| ONIX 3.0 | title_registered events support ONIX metadata |
| OAuth 2.0 / OIDC | Actor identity on every event |
| SOC 2 | Immutable event log satisfies audit requirements |
| GDPR | Event stream tracks all personal data access |
| MCP | AI assistant integration via ai stream events |

---

## Event Store

```sql
CREATE TABLE event_store (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         TEXT NOT NULL CHECK (stream_type IN (
                            'org','user','title','contract',
                            'rights','royalty','sales','ai','config'
                        )),
    stream_id           UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    event_type          TEXT NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    actor_id            UUID,
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','import_job','api_client'
                        )),
    org_id              UUID,
    ce_source           TEXT NOT NULL DEFAULT '/media-rights',
    ce_specversion      TEXT NOT NULL DEFAULT '1.0',
    ce_type             TEXT NOT NULL,
    ce_time             TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_number)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_type, stream_id, sequence_number);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_org ON event_store (org_id, created_at);
```

### Key Event Types by Stream

**title stream:**
- `title_registered` — new title added to catalogue with identifiers (ISRC, ISWC, ISAN, ISBN, EIDR)
- `title_metadata_updated` — release date, genre, duration, language changed
- `contributor_added` — party linked to title with role and ownership percentage
- `contributor_removed` — party unlinked from title
- `contributor_split_changed` — ownership percentage adjusted
- `title_archived` — title removed from active catalogue
- `title_parent_linked` — episode linked to series, track linked to album

**contract stream:**
- `contract_drafted` — new contract created with parties and terms
- `contract_negotiation_updated` — terms revised during negotiation
- `contract_executed` — contract signed and activated (with signature_json)
- `contract_amended` — existing contract modified (captures before/after)
- `contract_renewed` — auto-renewal triggered or manual renewal executed
- `contract_terminated` — contract ended early
- `contract_expired` — contract reached end date
- `obligation_created` — new obligation added (royalty statement, renewal notice)
- `obligation_fulfilled` — obligation completed (statement sent, payment made)
- `clause_extracted` — AI extracted clauses from uploaded PDF

**rights stream:**
- `right_granted` — structured grant with territory, platform, format, language, window, exclusivity
- `right_revoked` — grant withdrawn or terminated
- `right_expired` — window end date passed
- `right_amended` — grant terms modified (territory added, window extended)
- `conflict_detected` — overlapping exclusive grants identified
- `conflict_resolved` — conflict cleared (grant amended or revoked)
- `holdback_applied` — holdback period activated
- `avails_queried` — natural-language or structured avails query logged

**sales stream:**
- `sales_batch_imported` — DSP/retail report ingested (batch metadata, row count, source)
- `sales_lines_matched` — sales lines linked to titles and contracts
- `sales_lines_unmatched` — unmatched lines flagged for review
- `anomaly_detected` — under-reporting, missed channel, or unexpected spike found
- `anomaly_resolved` — anomaly reviewed and classified (confirmed, false_positive, adjusted)

**royalty stream:**
- `royalty_calculated` — calculation run for contract/period (captures inputs, rules applied, output)
- `escalation_triggered` — volume threshold crossed, rate increased
- `advance_recouped` — advance balance reduced by earned royalties
- `minimum_guarantee_shortfall` — earned royalties below MG, top-up applied
- `statement_generated` — PDF/CSV statement created for payee
- `statement_sent` — statement delivered to payee (channel: email, portal, post)
- `statement_acknowledged` — payee confirmed receipt
- `payment_issued` — payment sent to payee (amount, currency, method)
- `payment_disputed` — payee disputed calculation

**ai stream:**
- `clause_extraction_completed` — AI extracted rights terms from PDF
- `rights_auto_classified` — AI classified grant dimensions from text
- `anomaly_flagged` — AI flagged suspicious sales data
- `avails_question_answered` — AI answered natural-language avails query
- `renewal_recommended` — AI suggested renewal based on performance
- `suggestion_applied` — user accepted AI suggestion
- `suggestion_dismissed` — user dismissed AI suggestion

---

## Stream Snapshot

```sql
CREATE TABLE stream_snapshot (
    stream_type         TEXT NOT NULL,
    stream_id           UUID NOT NULL,
    sequence_number     BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

---

## Projection Checkpoint

```sql
CREATE TABLE projection_checkpoint (
    projection_name     TEXT PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence       BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model: Catalogue

```sql
CREATE TABLE rm_catalogue (
    title_id            UUID PRIMARY KEY,
    org_id              UUID NOT NULL,
    title_name          TEXT NOT NULL,
    title_type          TEXT NOT NULL,
    vertical            TEXT NOT NULL,
    parent_title_id     UUID,
    status              TEXT NOT NULL,
    identifiers_json    JSONB NOT NULL DEFAULT '{}',
    -- {"isrc": "GBAYE0000001", "iswc": "T-345246800-1", "upc": "0602445790456"}
    contributors_json   JSONB NOT NULL DEFAULT '[]',
    -- [{"party_id": "uuid", "name": "Jane Doe", "role": "composer", "ownership_pct": 50.0}]
    active_grants_count INTEGER NOT NULL DEFAULT 0,
    active_contracts_count INTEGER NOT NULL DEFAULT 0,
    revenue_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "lifetime_gross_cents": 4500000,
    --   "lifetime_streams": 15000000,
    --   "by_channel": {"spotify": 2700000, "apple_music": 1200000},
    --   "current_period_cents": 350000
    -- }
    metadata_json       JSONB NOT NULL DEFAULT '{}',
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cat_org ON rm_catalogue (org_id, vertical, status);
CREATE INDEX idx_cat_identifiers ON rm_catalogue USING GIN (identifiers_json);
CREATE INDEX idx_cat_name ON rm_catalogue USING GIN (to_tsvector('english', title_name));
```

---

## Read Model: Avails

```sql
CREATE TABLE rm_avails (
    grant_id            UUID PRIMARY KEY,
    title_id            UUID NOT NULL,
    title_name          TEXT NOT NULL,
    contract_id         UUID NOT NULL,
    contract_ref        TEXT NOT NULL,
    right_type          TEXT NOT NULL,
    territories         TEXT[] NOT NULL DEFAULT '{}',
    excluded_territories TEXT[] NOT NULL DEFAULT '{}',
    platforms           TEXT[] NOT NULL DEFAULT '{}',
    formats             TEXT[] NOT NULL DEFAULT '{}',
    languages           TEXT[] NOT NULL DEFAULT '{}',
    window_start        DATE NOT NULL,
    window_end          DATE,
    exclusivity         TEXT,
    holdback_days       INTEGER DEFAULT 0,
    is_sublicensable    BOOLEAN NOT NULL DEFAULT FALSE,
    licensor_name       TEXT,
    licensee_name       TEXT,
    status              TEXT NOT NULL,
    has_conflict        BOOLEAN NOT NULL DEFAULT FALSE,
    conflict_details_json JSONB,
    -- [{
    --   "conflicting_grant_id": "uuid",
    --   "conflicting_contract_ref": "IR-2026-LIC-002",
    --   "overlap_territories": ["DE","FR"],
    --   "overlap_window": {"start": "2026-06-01", "end": "2027-12-31"},
    --   "detected_at": "2026-05-20T14:30:00Z"
    -- }]
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_avails_title ON rm_avails (title_id, right_type, status);
CREATE INDEX idx_avails_territories ON rm_avails USING GIN (territories);
CREATE INDEX idx_avails_window ON rm_avails (window_start, window_end)
    WHERE status = 'active';
CREATE INDEX idx_avails_conflicts ON rm_avails (title_id)
    WHERE has_conflict = TRUE;
```

---

## Read Model: Contract Status

```sql
CREATE TABLE rm_contract_status (
    contract_id         UUID PRIMARY KEY,
    org_id              UUID NOT NULL,
    contract_ref        TEXT NOT NULL,
    contract_type       TEXT NOT NULL,
    status              TEXT NOT NULL,
    effective_date      DATE NOT NULL,
    expiry_date         DATE,
    licensor_name       TEXT NOT NULL,
    licensee_name       TEXT NOT NULL,
    parties_json        JSONB NOT NULL DEFAULT '[]',
    grants_count        INTEGER NOT NULL DEFAULT 0,
    titles_json         JSONB NOT NULL DEFAULT '[]',
    -- [{"title_id": "uuid", "title_name": "Song Title", "right_type": "streaming"}]
    financial_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "advance_cents": 5000000, "recouped_cents": 3200000,
    --   "minimum_guarantee_cents": 1000000,
    --   "total_royalties_earned_cents": 4800000,
    --   "total_paid_cents": 3200000
    -- }
    royalty_rules_json  JSONB NOT NULL DEFAULT '[]',
    obligations_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "type": "royalty_statement", "next_due": "2026-07-15",
    --   "status": "pending", "last_fulfilled": "2026-04-15"
    -- }]
    next_obligation_date DATE,
    clause_extraction_json JSONB,
    timeline_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "event": "contract_drafted", "at": "2026-01-10", "actor": "Rights Manager"},
    --  {"event": "contract_executed", "at": "2026-01-20"},
    --  {"event": "right_granted", "at": "2026-01-20", "count": 5}
    -- ]
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cstatus_org ON rm_contract_status (org_id, status);
CREATE INDEX idx_cstatus_expiry ON rm_contract_status (expiry_date)
    WHERE status = 'active';
CREATE INDEX idx_cstatus_obligation ON rm_contract_status (next_obligation_date)
    WHERE next_obligation_date IS NOT NULL;
```

---

## Read Model: Royalty Dashboard

```sql
CREATE TABLE rm_royalty_dashboard (
    org_id              UUID NOT NULL,
    period_type         TEXT NOT NULL CHECK (period_type IN (
                            'monthly','quarterly','semi_annual','annual'
                        )),
    period_start        DATE NOT NULL,
    summary_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_gross_revenue_cents": 25000000,
    --   "total_royalties_cents": 3750000,
    --   "total_advances_recouped_cents": 800000,
    --   "total_net_payable_cents": 2950000,
    --   "statements_generated": 45,
    --   "statements_sent": 42,
    --   "payments_issued": 38,
    --   "disputes_open": 2
    -- }
    by_payee_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "party_id": "uuid", "name": "Jane Doe",
    --   "gross_cents": 1200000, "royalty_cents": 180000,
    --   "advance_remaining_cents": 1800000,
    --   "net_payable_cents": 0, "status": "recouping"
    -- }]
    by_title_json       JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "title_id": "uuid", "title_name": "Song Title",
    --   "gross_cents": 350000, "royalty_cents": 52500,
    --   "streams": 1500000
    -- }]
    by_channel_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "spotify": {"gross_cents": 12000000, "streams": 45000000},
    --   "apple_music": {"gross_cents": 8000000, "streams": 25000000}
    -- }
    by_territory_json   JSONB NOT NULL DEFAULT '{}',
    -- {"US": 15000000, "GB": 5000000, "DE": 3000000}
    currency_json       JSONB NOT NULL DEFAULT '{}',
    -- {"USD": 20000000, "GBP": 3000000, "EUR": 2000000}
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (org_id, period_type, period_start)
);
CREATE INDEX idx_royalty_period ON rm_royalty_dashboard (period_type, period_start DESC);
```

---

## Read Model: Sales Reconciliation

```sql
CREATE TABLE rm_sales_reconciliation (
    batch_id            UUID PRIMARY KEY,
    org_id              UUID NOT NULL,
    source_channel      TEXT NOT NULL,
    source_file         TEXT,
    report_period_start DATE NOT NULL,
    report_period_end   DATE NOT NULL,
    import_status       TEXT NOT NULL,
    total_lines         INTEGER NOT NULL DEFAULT 0,
    matched_lines       INTEGER NOT NULL DEFAULT 0,
    unmatched_lines     INTEGER NOT NULL DEFAULT 0,
    match_rate          NUMERIC(5,3),
    total_gross_cents   BIGINT NOT NULL DEFAULT 0,
    total_units         BIGINT NOT NULL DEFAULT 0,
    total_streams       BIGINT NOT NULL DEFAULT 0,
    anomalies_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "type": "under_reporting",
    --   "title_name": "Song Title",
    --   "expected_cents": 50000, "actual_cents": 12000,
    --   "confidence": 0.89, "status": "pending_review"
    -- }]
    unmatched_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "identifier": "GBAYE0000099",
    --   "title_text": "Unknown Track",
    --   "gross_cents": 2500, "streams": 15000
    -- }]
    imported_at         TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recon_org ON rm_sales_reconciliation (org_id, source_channel);
CREATE INDEX idx_recon_period ON rm_sales_reconciliation (report_period_start DESC);
```

---

## Example Event Replay: Rights Grant Lifecycle

```sql
-- Replay the full lifecycle of a rights grant
SELECT event_type, ce_time, actor_type,
       event_data, metadata
FROM event_store
WHERE stream_type = 'rights'
  AND stream_id = 'grant-uuid'
ORDER BY sequence_number;

-- Results show:
-- 1. right_granted (territories: [US,CA,GB], window: 2026-01-01 to 2028-12-31, exclusive)
-- 2. conflict_detected (overlapping grant found for GB, streaming rights)
-- 3. right_amended (territories changed to [US,CA], GB removed)
-- 4. conflict_resolved (GB conflict cleared)
-- 5. right_expired (window end reached 2028-12-31)
```

### Point-in-time avails reconstruction

```sql
-- What rights were active for a title on a specific date?
SELECT event_data->>'right_type' AS right_type,
       event_data->'territories' AS territories,
       event_data->>'exclusivity' AS exclusivity,
       event_data->>'window_start' AS window_start,
       event_data->>'window_end' AS window_end,
       ce_time AS granted_at
FROM event_store
WHERE stream_type = 'rights'
  AND event_type = 'right_granted'
  AND event_data->>'title_id' = 'title-uuid'
  AND ce_time <= '2027-06-01T00:00:00Z'
  AND NOT EXISTS (
      SELECT 1 FROM event_store e2
      WHERE e2.stream_type = 'rights'
        AND e2.stream_id = event_store.stream_id
        AND e2.event_type IN ('right_revoked', 'right_expired')
        AND e2.ce_time <= '2027-06-01T00:00:00Z'
  );
```

### Royalty calculation audit trail

```sql
-- Trace how a specific royalty amount was calculated
SELECT event_type, ce_time,
       event_data->>'contract_ref' AS contract,
       event_data->>'payee_name' AS payee,
       event_data->>'rule_type' AS rule_type,
       (event_data->>'gross_revenue_cents')::BIGINT AS gross,
       (event_data->>'rate_applied_pct')::NUMERIC AS rate,
       (event_data->>'royalty_cents')::BIGINT AS royalty,
       (event_data->>'advance_offset_cents')::BIGINT AS advance_offset,
       (event_data->>'net_payable_cents')::BIGINT AS net_payable
FROM event_store
WHERE stream_type = 'royalty'
  AND event_data->>'contract_id' = 'contract-uuid'
  AND event_data->>'period_start' = '2026-04-01'
ORDER BY ce_time;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshot, projection_checkpoint |
| Read Models | 5 | rm_catalogue, rm_avails, rm_contract_status, rm_royalty_dashboard, rm_sales_reconciliation |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Rights stream as provenance chain** — every rights grant, amendment, revocation, and conflict detection is an immutable event. This provides the legal-grade audit trail needed for rights-of-record disputes: "prove that this exclusive grant existed before the conflicting deal was signed."

2. **`conflict_detected` / `conflict_resolved` events** — conflict detection is a first-class event rather than a computed flag. This captures when a conflict was identified, by what logic (human review vs. AI), and how it was resolved — essential for dispute arbitration.

3. **`rm_avails` as materialised view** — the avails query is the most performance-critical operation. Materialising active grants with pre-computed conflict flags means avails search is a simple table scan with territorial array filtering, rather than a complex event replay.

4. **Royalty calculation events capture full inputs** — each `royalty_calculated` event stores the gross revenue, rule applied, rate, escalation threshold, advance offset, and net payable. Auditors can verify any royalty amount by examining the event without replaying upstream events.

5. **Sales events separate from royalty events** — `sales_batch_imported` events track raw DSP data ingestion, while `royalty_calculated` events track the business logic applied to that data. This separation enables re-running royalty calculations against the same sales data when rules change.

6. **`rm_sales_reconciliation` per batch** — DSP report ingestion is a batch operation. The read model surfaces match rates, unmatched lines, and anomalies per import batch, enabling the operations team to resolve data quality issues before royalty calculation.

7. **`rm_contract_status` with timeline** — the contract timeline (`contract_drafted` → `contract_executed` → `right_granted` → `obligation_fulfilled`) provides a visual audit trail for each deal. The `next_obligation_date` index enables the obligation calendar dashboard.

8. **`advance_recouped` events** — advance recoupment is a critical financial event that changes the net-payable status of a deal. Modeling it as an explicit event (rather than updating a balance) preserves the recoupment history and enables time-based financial reporting.

9. **CloudEvents envelope** — `ce_source`, `ce_specversion`, `ce_type`, and `ce_time` follow the CloudEvents 1.0 specification. This enables interoperability with enterprise event buses and message brokers used by larger content companies.

10. **8 tables** — three infrastructure tables (event store, snapshots, checkpoints) plus five read models covering the core views: catalogue browsing, avails search, contract management, royalty analytics, and sales reconciliation.
