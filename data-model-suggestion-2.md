# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Media Rights Management · Created: 2026-05-26

## Philosophy

The two central aggregates in media rights are the **title** (what is being licensed) and the **contract** (who licensed what, on what terms). This model makes each the centre of its own rich JSONB document: titles embed contributors, industry identifiers, and a rights summary; contracts embed parties, rights grants, royalty rules, obligations, and financial terms. Sales data remains relational and partitioned because DSP ingestion is high-volume (millions of rows) and requires batch processing.

Media rights has two dominant access patterns: (1) "show me everything about this title — who owns it, what rights are granted, in which territories, and what's available" and (2) "show me this contract with all its terms, parties, grants, rules, and obligation deadlines." Embedding rights grants on both the title (for avails queries) and the contract (for royalty calculation) means both views are single-row reads. The trade-off is denormalisation: a rights grant appears in two places and must be kept in sync.

For mid-market content companies managing 1,000-50,000 titles with 50-500 active contracts, the JSONB approach drastically reduces schema complexity and migration burden. Adding a new right type (e.g., "ai_training"), a new territory grouping, or a new royalty rule variant requires no migration — just a new field in the JSONB structure.

**Best for:** Teams building an MVP for mid-market content companies where rapid iteration on rights dimensions, minimal schema migrations, fast catalogue browsing, and quick addition of new verticals (music, film, publishing) are priorities.

**Trade-offs:**
- Pro: 6 tables — simple schema, fast to deploy
- Pro: Full title context (contributors, identifiers, rights) in one row read
- Pro: Full contract context (parties, grants, rules, obligations) in one row read
- Pro: New right types, territories, and royalty structures require no migration
- Con: Rights grants denormalised across titles and contracts (sync required)
- Con: Cross-catalogue analytics require JSONB extraction
- Con: No FK enforcement between grants, rules, and sales data within JSONB
- Con: Large catalogues with hundreds of grants per title can produce large JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C ODRL 2.2 | Rights grants stored with ODRL-aligned permission semantics in JSONB |
| ISO 15706 (ISAN) | identifiers_json on titles for audiovisual works |
| ISO 3901 (ISRC) | identifiers_json on titles for sound recordings |
| ISO 15707 (ISWC) | identifiers_json on titles for musical compositions |
| ISO 27729 (ISNI) | Party ISNI in contributors_json and parties_json |
| ISO 3166 | Territory codes in rights_json |
| ISO 4217 | Currency codes in financial_json |
| ISO 639 | Language codes in rights_json |
| DDEX ERN/DSR | Sales data ingestion; title metadata export |
| ONIX 3.0 | Book catalogue import/export |
| EIDR | identifiers_json for audiovisual registry |
| OpenAPI 3.1 | REST API |
| OAuth 2.0 / OIDC | Authentication |
| GDPR | Payee personal data |
| MCP | AI assistant integration |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    role                TEXT NOT NULL CHECK (role IN (
                            'admin','rights_manager','royalty_accountant',
                            'licensee','payee','auditor','viewer'
                        )),
    org_json            JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "id": "uuid", "name": "Indie Records Ltd",
    --   "org_type": "label", "country": "GB",
    --   "currency_default": "GBP",
    --   "registration_number": "12345678",
    --   "settings": {"royalty_cycle": "quarterly", "auto_approve_threshold_cents": 100000}
    -- }
    preferences_json    JSONB NOT NULL DEFAULT '{}',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_role ON users (role);
```

---

## Titles

```sql
CREATE TABLE titles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL,
    title_name          TEXT NOT NULL,
    title_type          TEXT NOT NULL CHECK (title_type IN (
                            'film','tv_series','tv_episode','music_recording',
                            'music_composition','book','ebook','audiobook',
                            'game','software','image','article',
                            'podcast','sports_event','brand','other'
                        )),
    vertical            TEXT NOT NULL CHECK (vertical IN (
                            'film_tv','music','publishing','games',
                            'sports','brand_licensing','other'
                        )),
    parent_title_id     UUID REFERENCES titles(id),
    status              TEXT NOT NULL CHECK (status IN (
                            'active','archived','withdrawn','pending'
                        )) DEFAULT 'active',
    identifiers_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "isrc": "GBAYE0000001",
    --   "iswc": "T-345246800-1",
    --   "isan": null,
    --   "isbn": null,
    --   "eidr_id": null,
    --   "upc": "0602445790456",
    --   "catalogue_number": "IR-2026-0042",
    --   "doi": null
    -- }
    contributors_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "party_id": "uuid", "name": "Jane Doe",
    --   "isni": "0000000121032683",
    --   "ipi_number": "00123456789",
    --   "role": "composer", "ownership_pct": 50.0,
    --   "is_controlled": true, "territory_scope": ["WW"]
    -- }, {
    --   "party_id": "uuid", "name": "John Smith",
    --   "role": "lyricist", "ownership_pct": 50.0,
    --   "is_controlled": true
    -- }]
    rights_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "grant_id": "uuid", "contract_id": "uuid",
    --   "contract_ref": "IR-2026-LIC-001",
    --   "right_type": "streaming",
    --   "territories": ["US","CA","GB","DE","FR"],
    --   "excluded_territories": [],
    --   "platforms": ["spotify","apple_music","amazon_music"],
    --   "languages": [],
    --   "window_start": "2026-01-01",
    --   "window_end": "2028-12-31",
    --   "exclusivity": "non_exclusive",
    --   "licensee": "StreamCo Distribution",
    --   "status": "active"
    -- }, {
    --   "grant_id": "uuid", "contract_id": "uuid",
    --   "right_type": "synchronisation",
    --   "territories": ["WW"],
    --   "window_start": "2026-06-01",
    --   "window_end": null,
    --   "exclusivity": "exclusive",
    --   "licensee": "FilmSync Ltd",
    --   "status": "active"
    -- }]
    metadata_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "release_date": "2026-03-15",
    --   "duration_seconds": 245,
    --   "language_original": "en",
    --   "genre": "electronic",
    --   "bpm": 128,
    --   "key": "Am"
    -- }
    revenue_summary_json JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "lifetime_gross_cents": 4500000,
    --   "lifetime_units": 2500000,
    --   "lifetime_streams": 15000000,
    --   "by_channel": {
    --     "spotify": {"streams": 9000000, "gross_cents": 2700000},
    --     "apple_music": {"streams": 4000000, "gross_cents": 1200000}
    --   },
    --   "by_territory": {
    --     "US": {"gross_cents": 2800000},
    --     "GB": {"gross_cents": 900000}
    --   },
    --   "last_updated": "2026-05-01"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_titles_org ON titles (org_id, vertical, status);
CREATE INDEX idx_titles_type ON titles (title_type, status);
CREATE INDEX idx_titles_identifiers ON titles USING GIN (identifiers_json);
CREATE INDEX idx_titles_rights ON titles USING GIN (rights_json);
CREATE INDEX idx_titles_contributors ON titles USING GIN (contributors_json);
CREATE INDEX idx_titles_name ON titles USING GIN (to_tsvector('english', title_name));
CREATE INDEX idx_titles_parent ON titles (parent_title_id) WHERE parent_title_id IS NOT NULL;
```

---

## Contracts

```sql
CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL,
    contract_ref        TEXT NOT NULL,
    contract_type       TEXT NOT NULL CHECK (contract_type IN (
                            'licence','sub_licence','distribution',
                            'publishing','co_publishing','sub_publishing',
                            'administration','synchronisation','master_use',
                            'option','assignment','work_for_hire','other'
                        )),
    status              TEXT NOT NULL CHECK (status IN (
                            'draft','negotiation','executed','active',
                            'expired','terminated','renewed'
                        )) DEFAULT 'draft',
    effective_date      DATE NOT NULL,
    expiry_date         DATE,
    currency            TEXT NOT NULL DEFAULT 'USD',
    parties_json        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "party_id": "uuid", "name": "Indie Records Ltd",
    --   "role": "licensor", "isni": "0000000121032683",
    --   "contact_email": "rights@indierecords.com"
    -- }, {
    --   "party_id": "uuid", "name": "StreamCo Distribution",
    --   "role": "licensee", "contact_email": "licensing@streamco.com"
    -- }]
    rights_grants_json  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "title_id": "uuid", "title_name": "Song Title",
    --   "right_type": "streaming",
    --   "territories": ["US","CA","GB","DE","FR"],
    --   "platforms": ["spotify","apple_music","amazon_music"],
    --   "window_start": "2026-01-01", "window_end": "2028-12-31",
    --   "exclusivity": "non_exclusive",
    --   "is_sublicensable": false
    -- }]
    royalty_rules_json  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "payee_party_id": "uuid", "payee_name": "Jane Doe",
    --   "rule_type": "percentage_of_receipts",
    --   "rate_pct": 15.0, "applies_to": "net_receipts",
    --   "recoup_from_advance": true,
    --   "escalation": [
    --     {"threshold_units": 100000, "rate_pct": 17.5},
    --     {"threshold_units": 500000, "rate_pct": 20.0}
    --   ]
    -- }, {
    --   "id": "uuid", "payee_party_id": "uuid", "payee_name": "John Smith",
    --   "rule_type": "co_author_split",
    --   "split_pct": 50.0, "applies_to": "net_receipts"
    -- }]
    financial_json      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "advance_cents": 5000000,
    --   "advance_recouped_cents": 3200000,
    --   "minimum_guarantee_cents": 1000000,
    --   "auto_renew": true,
    --   "renewal_notice_days": 90,
    --   "exclusivity": "non_exclusive"
    -- }
    obligations_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "royalty_statement",
    --   "description": "Quarterly royalty statement due",
    --   "frequency": "quarterly", "next_due": "2026-07-15",
    --   "status": "pending"
    -- }, {
    --   "id": "uuid", "type": "renewal_notice",
    --   "description": "Renewal decision deadline",
    --   "due_date": "2028-10-01", "status": "pending"
    -- }]
    clause_extraction_json JSONB,
    signature_json      JSONB,
    document_url        TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, contract_ref)
);
CREATE INDEX idx_contracts_org ON contracts (org_id, status);
CREATE INDEX idx_contracts_expiry ON contracts (expiry_date)
    WHERE status = 'active';
CREATE INDEX idx_contracts_parties ON contracts USING GIN (parties_json);
CREATE INDEX idx_contracts_grants ON contracts USING GIN (rights_grants_json);
CREATE INDEX idx_contracts_obligations ON contracts USING GIN (obligations_json);
```

---

## Sales Data

```sql
CREATE TABLE sales_data (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL,
    title_id            UUID REFERENCES titles(id),
    title_identifier    TEXT,
    source_channel      TEXT NOT NULL CHECK (source_channel IN (
                            'spotify','apple_music','amazon_music',
                            'youtube','tidal','deezer','bandcamp',
                            'beatport','amazon_retail','barnes_noble',
                            'kobo','google_play','itunes','netflix',
                            'hulu','disney_plus','hbo_max',
                            'theatrical_box_office','broadcaster',
                            'physical_retail','direct_sales',
                            'cmo_distribution','other'
                        )),
    report_period_start DATE NOT NULL,
    report_period_end   DATE NOT NULL,
    territory           TEXT NOT NULL,
    currency_reported   TEXT NOT NULL,
    units_sold          BIGINT NOT NULL DEFAULT 0,
    streams             BIGINT NOT NULL DEFAULT 0,
    gross_revenue_cents BIGINT NOT NULL DEFAULT 0,
    net_revenue_cents   BIGINT NOT NULL DEFAULT 0,
    exchange_rate       NUMERIC(12,6),
    converted_cents     BIGINT,
    import_batch_id     UUID,
    source_file         TEXT,
    is_matched          BOOLEAN NOT NULL DEFAULT FALSE,
    is_anomalous        BOOLEAN NOT NULL DEFAULT FALSE,
    anomaly_json        JSONB,
    -- {
    --   "reason": "under_reporting",
    --   "expected_range_cents": [50000, 80000],
    --   "actual_cents": 12000,
    --   "confidence": 0.89
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (report_period_start);

CREATE INDEX idx_sales_title ON sales_data (title_id, report_period_start);
CREATE INDEX idx_sales_org ON sales_data (org_id, source_channel, report_period_start);
CREATE INDEX idx_sales_unmatched ON sales_data (org_id, is_matched)
    WHERE is_matched = FALSE;
CREATE INDEX idx_sales_batch ON sales_data (import_batch_id);
```

---

## AI Suggestions

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL,
    user_id             UUID REFERENCES users(id),
    suggestion_type     TEXT NOT NULL CHECK (suggestion_type IN (
                            'clause_extraction','rights_classification',
                            'conflict_detection','anomaly_alert',
                            'avails_answer','renewal_recommendation',
                            'code_suggestion','query_response'
                        )),
    title               TEXT NOT NULL,
    body                TEXT NOT NULL,
    suggestion_data     JSONB,
    contract_id         UUID REFERENCES contracts(id),
    title_id            UUID REFERENCES titles(id),
    confidence          NUMERIC(4,3),
    is_applied          BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,
    llm_model           TEXT,
    tokens_used         INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_org ON ai_suggestions (org_id, created_at DESC);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID,
    user_id             UUID REFERENCES users(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','import_job','api_client'
                        )),
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB,
    ip_address          INET,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log (org_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Avails search — available rights for a title from JSONB

```sql
SELECT title_name, r->>'right_type' AS right_type,
       r->>'exclusivity' AS exclusivity,
       r->>'window_start' AS window_start,
       r->>'window_end' AS window_end,
       r->'territories' AS territories,
       r->>'licensee' AS licensee,
       r->>'status' AS grant_status
FROM titles t,
     jsonb_array_elements(t.rights_json) AS r
WHERE t.id = 'title-uuid'
  AND r->>'status' = 'active'
  AND r->'territories' @> '"DE"'
ORDER BY r->>'right_type';
```

### Full contract context — single row read

```sql
SELECT contract_ref, contract_type, status,
       effective_date, expiry_date,
       parties_json, rights_grants_json,
       royalty_rules_json, financial_json,
       obligations_json, clause_extraction_json
FROM contracts
WHERE id = 'contract-uuid';
```

### Upcoming obligations across all contracts

```sql
SELECT c.contract_ref, c.status,
       o->>'type' AS obligation_type,
       o->>'description' AS description,
       (o->>'next_due')::DATE AS due_date
FROM contracts c,
     jsonb_array_elements(c.obligations_json) AS o
WHERE c.org_id = 'org-uuid'
  AND c.status = 'active'
  AND (o->>'next_due')::DATE <= now() + INTERVAL '30 days'
  AND o->>'status' = 'pending'
ORDER BY (o->>'next_due')::DATE;
```

### Top titles by revenue from embedded summary

```sql
SELECT title_name, title_type, vertical,
       (revenue_summary_json->>'lifetime_gross_cents')::BIGINT AS lifetime_gross,
       (revenue_summary_json->>'lifetime_streams')::BIGINT AS lifetime_streams
FROM titles
WHERE org_id = 'org-uuid'
  AND status = 'active'
ORDER BY (revenue_summary_json->>'lifetime_gross_cents')::BIGINT DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds org context) |
| Catalogue | 1 | titles (embeds contributors, identifiers, rights grants, revenue summary) |
| Contracts | 1 | contracts (embeds parties, rights grants, royalty rules, obligations, financials) |
| Sales | 1 | sales_data (relational, partitioned — high-volume DSP ingestion) |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **6** | |

---

## Key Design Decisions

1. **Rights grants on both titles and contracts** — the avails query is title-centric ("what's available for this title?") while royalty calculation is contract-centric ("what are the terms of this deal?"). Embedding grants on both aggregates means each view is a single-row read. A background sync process keeps the two copies consistent.

2. **`identifiers_json` on titles** — industry identifiers (ISRC, ISWC, ISAN, ISBN, EIDR, UPC, DOI) vary by vertical: music titles have ISRC/ISWC, books have ISBN, films have ISAN/EIDR. JSONB accommodates all verticals without nullable columns. GIN index enables lookup by any identifier.

3. **`contributors_json` on titles** — a title typically has 1-10 contributors with ownership percentages and roles. Embedding them with ISNI and IPI numbers means the catalogue view and DDEX/CWR export data are a single-row read.

4. **`royalty_rules_json` on contracts** — a contract typically has 2-8 royalty rules (percentage rates, escalators, splits, advance recoupment). Embedding rules with payee references means the royalty calculation engine loads all rules from a single contract row.

5. **`obligations_json` on contracts** — renewal deadlines, statement due dates, and reporting obligations (typically 3-10 per contract) are embedded for the obligation calendar view. New obligation types require no migration.

6. **Sales data remains relational** — DSP reports can contain millions of rows per import. Keeping sales_data in its own partitioned table preserves batch processing performance, period-scoped queries, and anomaly detection indexing. The `is_matched` flag tracks title-matching progress.

7. **`revenue_summary_json` on titles** — pre-aggregated revenue by channel and territory eliminates the need to scan sales_data for catalogue browsing. Updated periodically from sales data.

8. **`clause_extraction_json` on contracts** — AI-extracted clauses from PDF contracts are stored alongside the human-verified JSONB fields (parties, grants, rules). This preserves the raw AI output for audit while the canonical fields drive business logic.

9. **`anomaly_json` on sales data** — AI-detected anomalies (under-reporting, missed channels, unexpected territory spikes) are stored per sales line with confidence scores, enabling anomaly review workflows without a separate findings table.

10. **6 tables** — the title and contract aggregates capture the two primary domain objects with their full context. Sales data stays relational for volume. This keeps the schema simple for mid-market deployments while supporting rapid iteration on rights dimensions and royalty structures.
