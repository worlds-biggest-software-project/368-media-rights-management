# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Media Rights Management · Created: 2026-05-26

## Philosophy

Every concept in the media rights domain — organisations, parties, titles, contracts, rights grants, royalty rules, sales data, royalty calculations, and statements — gets its own table with strict foreign-key relationships. This mirrors the dimensional structure of rights management where a single grant spans territory × platform × format × language × time window × exclusivity, and royalty calculations chain from sales data through rules to statement lines.

Media rights management has two dominant access patterns: (1) "show me what rights are available for Title X in territory Y on platform Z after date D" (avails query) and (2) "calculate royalties for Contract C for period Q, applying escalators, splits, advances, and recoupment." The avails query joins titles → rights_grants with multi-dimensional filtering and conflict detection. Royalty calculation joins sales_data → royalty_rules → royalty_calculations with waterfall logic. Both patterns are well-served by indexed relational queries.

The domain is inherently multi-dimensional and relational: a title has many contributors with different split percentages, a contract grants rights across multiple territories and platforms, and a single sales report from a DSP references thousands of titles. Normalisation prevents data duplication across these many-to-many relationships and ensures referential integrity across the rights chain.

**Best for:** Teams building a production rights platform where multi-dimensional avails queries, complex royalty waterfall calculations, cross-catalogue analytics, and industry-standard interoperability (DDEX, ONIX, EIDR) are priorities.

**Trade-offs:**
- Pro: Multi-dimensional rights queries with standard SQL joins and indexes
- Pro: Referential integrity across the rights chain (title → grant → rule → calculation → statement)
- Pro: Industry identifiers (ISRC, ISWC, ISAN, EIDR, ISBN) as indexed columns for interoperability
- Pro: Sales data partitioned for high-volume DSP ingestion (millions of rows)
- Con: 14 tables to maintain and migrate
- Con: Avails query with conflict detection requires multi-join with temporal overlap logic
- Con: Adding new rights dimensions requires schema migration
- Con: Royalty waterfall calculation spans 4+ tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| W3C ODRL 2.2 | Rights grants modeled with ODRL-aligned permission/prohibition semantics |
| ISO 15706 (ISAN) | titles.isan for audiovisual works |
| ISO 3901 (ISRC) | titles.isrc for sound recordings |
| ISO 15707 (ISWC) | titles.iswc for musical compositions |
| ISO 21047 (ISTC) | titles.istc for textual works |
| ISO 27729 (ISNI) | parties.isni for contributor/rights-holder identity |
| ISO 3166 | rights_grants.territory uses ISO 3166-1 alpha-2 codes |
| ISO 4217 | Multi-currency fields use ISO 4217 codes |
| ISO 639 | rights_grants.language uses ISO 639-1 codes |
| DDEX ERN/DSR | Sales data ingestion from DSP reports; title delivery metadata |
| CWR | Musical work registration with CMOs |
| ONIX 3.0 | Book catalogue import/export with territory rights |
| EIDR | titles.eidr_id for audiovisual content registry |
| OpenAPI 3.1 | REST API for avails and rights CRUD |
| JSON Schema 2020-12 | Validation of rights and royalty payloads |
| OAuth 2.0 / OIDC | Authentication; licensee/payee portal access |
| GDPR | Personal data of payees and contributors |
| SOC 2 | Audit trail for financial data |
| MCP | AI assistant integration |

---

## Organisations

```sql
CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    org_type            TEXT NOT NULL CHECK (org_type IN (
                            'publisher','label','studio','distributor',
                            'licensor','licensee','agency','cmo','other'
                        )),
    registration_number TEXT,
    tax_id              TEXT,
    country             TEXT NOT NULL,
    currency_default    TEXT NOT NULL DEFAULT 'USD',
    settings_json       JSONB NOT NULL DEFAULT '{}',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    role                TEXT NOT NULL CHECK (role IN (
                            'admin','rights_manager','royalty_accountant',
                            'licensee','payee','auditor','viewer'
                        )),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_org ON users (org_id, role);
```

---

## Parties

```sql
CREATE TABLE parties (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
    party_type          TEXT NOT NULL CHECK (party_type IN (
                            'individual','company','estate','trust','cmo'
                        )),
    name                TEXT NOT NULL,
    isni                TEXT,
    ipi_number          TEXT,
    tax_id              TEXT,
    country             TEXT,
    currency_preferred  TEXT DEFAULT 'USD',
    payment_method      TEXT CHECK (payment_method IN (
                            'bank_transfer','cheque','paypal','wire','held'
                        )),
    payment_details_json JSONB,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_parties_org ON parties (org_id);
CREATE INDEX idx_parties_isni ON parties (isni) WHERE isni IS NOT NULL;
CREATE INDEX idx_parties_ipi ON parties (ipi_number) WHERE ipi_number IS NOT NULL;
```

---

## Titles

```sql
CREATE TABLE titles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
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
    isan                TEXT,
    isrc                TEXT,
    iswc                TEXT,
    istc                TEXT,
    isbn                TEXT,
    eidr_id             TEXT,
    doi                 TEXT,
    upc                 TEXT,
    catalogue_number    TEXT,
    release_date        DATE,
    duration_seconds    INTEGER,
    language_original   TEXT,
    genre               TEXT,
    status              TEXT NOT NULL CHECK (status IN (
                            'active','archived','withdrawn','pending'
                        )) DEFAULT 'active',
    metadata_json       JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_titles_org ON titles (org_id, vertical);
CREATE INDEX idx_titles_type ON titles (title_type, status);
CREATE INDEX idx_titles_isrc ON titles (isrc) WHERE isrc IS NOT NULL;
CREATE INDEX idx_titles_iswc ON titles (iswc) WHERE iswc IS NOT NULL;
CREATE INDEX idx_titles_isan ON titles (isan) WHERE isan IS NOT NULL;
CREATE INDEX idx_titles_isbn ON titles (isbn) WHERE isbn IS NOT NULL;
CREATE INDEX idx_titles_eidr ON titles (eidr_id) WHERE eidr_id IS NOT NULL;
CREATE INDEX idx_titles_parent ON titles (parent_title_id) WHERE parent_title_id IS NOT NULL;
CREATE INDEX idx_titles_name ON titles USING GIN (to_tsvector('english', title_name));
```

---

## Title Contributors

```sql
CREATE TABLE title_contributors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title_id            UUID NOT NULL REFERENCES titles(id),
    party_id            UUID NOT NULL REFERENCES parties(id),
    contributor_role    TEXT NOT NULL CHECK (contributor_role IN (
                            'author','co_author','composer','lyricist',
                            'performer','producer','director','editor',
                            'translator','illustrator','photographer',
                            'arranger','adapter','rights_holder','other'
                        )),
    ownership_pct       NUMERIC(7,4),
    is_controlled       BOOLEAN NOT NULL DEFAULT TRUE,
    territory_scope     TEXT[] NOT NULL DEFAULT '{WW}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contributors_title ON title_contributors (title_id);
CREATE INDEX idx_contributors_party ON title_contributors (party_id);
```

---

## Contracts

```sql
CREATE TABLE contracts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
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
    licensor_party_id   UUID NOT NULL REFERENCES parties(id),
    licensee_party_id   UUID NOT NULL REFERENCES parties(id),
    effective_date      DATE NOT NULL,
    expiry_date         DATE,
    auto_renew          BOOLEAN NOT NULL DEFAULT FALSE,
    renewal_notice_days INTEGER,
    exclusivity         TEXT CHECK (exclusivity IN (
                            'exclusive','non_exclusive','sole'
                        )),
    currency            TEXT NOT NULL DEFAULT 'USD',
    advance_cents       BIGINT DEFAULT 0,
    advance_recouped_cents BIGINT DEFAULT 0,
    minimum_guarantee_cents BIGINT DEFAULT 0,
    document_url        TEXT,
    clause_extraction_json JSONB,
    obligations_json    JSONB NOT NULL DEFAULT '[]',
    signature_json      JSONB,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, contract_ref)
);
CREATE INDEX idx_contracts_org ON contracts (org_id, status);
CREATE INDEX idx_contracts_licensor ON contracts (licensor_party_id);
CREATE INDEX idx_contracts_licensee ON contracts (licensee_party_id);
CREATE INDEX idx_contracts_expiry ON contracts (expiry_date)
    WHERE status = 'active';
```

---

## Rights Grants

```sql
CREATE TABLE rights_grants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(id),
    title_id            UUID NOT NULL REFERENCES titles(id),
    right_type          TEXT NOT NULL CHECK (right_type IN (
                            'theatrical','home_video','svod','tvod','avod',
                            'est','linear_tv','free_tv','pay_tv',
                            'radio','streaming','download','physical',
                            'print','ebook','audiobook','translation',
                            'dramatisation','merchandise','synchronisation',
                            'public_performance','mechanical','broadcast',
                            'digital_distribution','ai_training',
                            'remake','sequel','all_rights','other'
                        )),
    territories         TEXT[] NOT NULL DEFAULT '{WW}',
    excluded_territories TEXT[] NOT NULL DEFAULT '{}',
    platforms           TEXT[] NOT NULL DEFAULT '{}',
    formats             TEXT[] NOT NULL DEFAULT '{}',
    languages           TEXT[] NOT NULL DEFAULT '{}',
    window_start        DATE NOT NULL,
    window_end          DATE,
    exclusivity         TEXT CHECK (exclusivity IN (
                            'exclusive','non_exclusive','sole'
                        )),
    holdback_days       INTEGER DEFAULT 0,
    is_sublicensable    BOOLEAN NOT NULL DEFAULT FALSE,
    status              TEXT NOT NULL CHECK (status IN (
                            'active','expired','terminated','pending','draft'
                        )) DEFAULT 'active',
    odrl_policy_json    JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_grants_contract ON rights_grants (contract_id);
CREATE INDEX idx_grants_title ON rights_grants (title_id, right_type);
CREATE INDEX idx_grants_avails ON rights_grants (title_id, status, window_start, window_end)
    WHERE status = 'active';
CREATE INDEX idx_grants_territories ON rights_grants USING GIN (territories);
CREATE INDEX idx_grants_window ON rights_grants (window_start, window_end)
    WHERE status = 'active';
```

---

## Royalty Rules

```sql
CREATE TABLE royalty_rules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id         UUID NOT NULL REFERENCES contracts(id),
    grant_id            UUID REFERENCES rights_grants(id),
    payee_party_id      UUID NOT NULL REFERENCES parties(id),
    rule_type           TEXT NOT NULL CHECK (rule_type IN (
                            'percentage_of_receipts','flat_per_unit',
                            'flat_per_period','escalating','tiered',
                            'minimum_guarantee','advance_recoupment',
                            'co_author_split','sub_publisher_split',
                            'override','custom_dsl'
                        )),
    rate_pct            NUMERIC(7,4),
    flat_amount_cents   BIGINT,
    currency            TEXT NOT NULL DEFAULT 'USD',
    escalation_json     JSONB,
    -- [{
    --   "threshold_units": 10000, "rate_pct": 10.0
    -- }, {
    --   "threshold_units": 50000, "rate_pct": 12.5
    -- }, {
    --   "threshold_units": 100000, "rate_pct": 15.0
    -- }]
    split_pct           NUMERIC(7,4),
    applies_to          TEXT CHECK (applies_to IN (
                            'gross_receipts','net_receipts',
                            'retail_price','wholesale_price',
                            'units_sold','streams','downloads'
                        )),
    recoup_from_advance BOOLEAN NOT NULL DEFAULT FALSE,
    priority_order      INTEGER NOT NULL DEFAULT 1,
    dsl_expression      TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rules_contract ON royalty_rules (contract_id);
CREATE INDEX idx_rules_payee ON royalty_rules (payee_party_id);
```

---

## Sales Data

```sql
CREATE TABLE sales_data (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
    title_id            UUID REFERENCES titles(id),
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
    source_line_number  INTEGER,
    is_matched          BOOLEAN NOT NULL DEFAULT FALSE,
    is_anomalous        BOOLEAN NOT NULL DEFAULT FALSE,
    anomaly_reason      TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (report_period_start);

CREATE INDEX idx_sales_title ON sales_data (title_id, report_period_start);
CREATE INDEX idx_sales_org ON sales_data (org_id, source_channel, report_period_start);
CREATE INDEX idx_sales_unmatched ON sales_data (org_id, is_matched)
    WHERE is_matched = FALSE;
CREATE INDEX idx_sales_anomalous ON sales_data (org_id, created_at)
    WHERE is_anomalous = TRUE;
CREATE INDEX idx_sales_batch ON sales_data (import_batch_id);
```

---

## Royalty Calculations

```sql
CREATE TABLE royalty_calculations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
    contract_id         UUID NOT NULL REFERENCES contracts(id),
    rule_id             UUID NOT NULL REFERENCES royalty_rules(id),
    payee_party_id      UUID NOT NULL REFERENCES parties(id),
    title_id            UUID REFERENCES titles(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    gross_revenue_cents BIGINT NOT NULL DEFAULT 0,
    net_revenue_cents   BIGINT NOT NULL DEFAULT 0,
    units               BIGINT NOT NULL DEFAULT 0,
    rate_applied_pct    NUMERIC(7,4),
    royalty_cents       BIGINT NOT NULL DEFAULT 0,
    advance_offset_cents BIGINT NOT NULL DEFAULT 0,
    net_payable_cents   BIGINT NOT NULL DEFAULT 0,
    currency            TEXT NOT NULL DEFAULT 'USD',
    calculation_notes   TEXT,
    status              TEXT NOT NULL CHECK (status IN (
                            'draft','approved','paid','disputed','void'
                        )) DEFAULT 'draft',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_calcs_payee ON royalty_calculations (payee_party_id, period_start);
CREATE INDEX idx_calcs_contract ON royalty_calculations (contract_id, period_start);
CREATE INDEX idx_calcs_status ON royalty_calculations (status, period_start);
```

---

## Royalty Statements

```sql
CREATE TABLE royalty_statements (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
    payee_party_id      UUID NOT NULL REFERENCES parties(id),
    statement_ref       TEXT NOT NULL,
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    currency            TEXT NOT NULL DEFAULT 'USD',
    total_royalty_cents  BIGINT NOT NULL DEFAULT 0,
    advance_offset_cents BIGINT NOT NULL DEFAULT 0,
    net_payable_cents   BIGINT NOT NULL DEFAULT 0,
    previous_balance_cents BIGINT NOT NULL DEFAULT 0,
    closing_balance_cents BIGINT NOT NULL DEFAULT 0,
    status              TEXT NOT NULL CHECK (status IN (
                            'draft','approved','sent','acknowledged',
                            'paid','disputed'
                        )) DEFAULT 'draft',
    sent_at             TIMESTAMPTZ,
    paid_at             TIMESTAMPTZ,
    document_url        TEXT,
    lines_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "title_name": "Song Title", "isrc": "USRC12345678",
    --   "territory": "US", "channel": "spotify",
    --   "units": 150000, "gross_cents": 60000, "rate_pct": 15.0,
    --   "royalty_cents": 9000
    -- }]
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, statement_ref)
);
CREATE INDEX idx_statements_payee ON royalty_statements (payee_party_id, period_start DESC);
CREATE INDEX idx_statements_status ON royalty_statements (status);
```

---

## AI Suggestions

```sql
CREATE TABLE ai_suggestions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organisations(id),
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
    org_id              UUID REFERENCES organisations(id),
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
CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
```

---

## Example Queries

### Avails query — what rights are available for a title in a territory?

```sql
SELECT t.title_name, rg.right_type, rg.exclusivity,
       rg.window_start, rg.window_end,
       rg.territories, rg.platforms, rg.languages,
       c.contract_ref, p.name AS licensor
FROM rights_grants rg
JOIN contracts c ON c.id = rg.contract_id
JOIN titles t ON t.id = rg.title_id
JOIN parties p ON p.id = c.licensor_party_id
WHERE rg.title_id = 'title-uuid'
  AND rg.status = 'active'
  AND 'DE' = ANY(rg.territories)
  AND rg.window_start <= '2027-01-01'
  AND (rg.window_end IS NULL OR rg.window_end >= '2027-01-01')
ORDER BY rg.right_type, rg.window_start;
```

### Conflict detection — overlapping exclusive grants

```sql
SELECT a.id AS grant_a, b.id AS grant_b,
       a.right_type, a.territories, a.window_start, a.window_end
FROM rights_grants a
JOIN rights_grants b ON a.title_id = b.title_id
    AND a.right_type = b.right_type
    AND a.id < b.id
    AND a.territories && b.territories
    AND a.window_start < COALESCE(b.window_end, '9999-12-31')
    AND b.window_start < COALESCE(a.window_end, '9999-12-31')
WHERE a.title_id = 'title-uuid'
  AND a.status = 'active' AND b.status = 'active'
  AND (a.exclusivity = 'exclusive' OR b.exclusivity = 'exclusive');
```

### Royalty calculation summary by payee

```sql
SELECT p.name AS payee,
       SUM(rc.gross_revenue_cents) AS total_gross,
       SUM(rc.royalty_cents) AS total_royalty,
       SUM(rc.advance_offset_cents) AS total_offset,
       SUM(rc.net_payable_cents) AS net_payable,
       rc.currency
FROM royalty_calculations rc
JOIN parties p ON p.id = rc.payee_party_id
WHERE rc.org_id = 'org-uuid'
  AND rc.period_start >= '2026-01-01'
  AND rc.period_end <= '2026-03-31'
  AND rc.status = 'approved'
GROUP BY p.id, p.name, rc.currency
ORDER BY net_payable DESC;
```

### Top-performing titles by revenue

```sql
SELECT t.title_name, t.title_type, t.vertical,
       SUM(sd.gross_revenue_cents) AS total_revenue,
       SUM(sd.units_sold) AS total_units,
       SUM(sd.streams) AS total_streams
FROM sales_data sd
JOIN titles t ON t.id = sd.title_id
WHERE sd.org_id = 'org-uuid'
  AND sd.report_period_start >= '2026-01-01'
GROUP BY t.id, t.title_name, t.title_type, t.vertical
ORDER BY total_revenue DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation | 1 | organisations |
| Users | 1 | users |
| Parties | 1 | parties (rights holders, licensees, payees with ISNI/IPI) |
| Catalogue | 2 | titles, title_contributors |
| Contracts | 2 | contracts, rights_grants (multi-dimensional) |
| Royalties | 3 | royalty_rules, royalty_calculations, royalty_statements |
| Sales | 1 | sales_data (partitioned) |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **13** | |

---

## Key Design Decisions

1. **`rights_grants` with array dimensions** — territories, platforms, formats, and languages are `TEXT[]` arrays with GIN indexes. This enables the avails query pattern: "find all active grants where territory contains 'DE' and window overlaps date range." Array overlap (`&&`) and containment (`@>`) operators make conflict detection expressible in SQL.

2. **`rights_grants.right_type` with `ai_training`** — AI-training-data licensing is modeled as a first-class right type alongside traditional rights (theatrical, SVOD, streaming, print). This addresses the emerging need for transparent rights provenance in AI training datasets.

3. **`royalty_rules` separate from contracts** — a single contract can have multiple royalty structures: different rates per territory, escalating rates by volume, co-author splits, sub-publisher shares. Separate rows per rule enable the waterfall calculation engine to process rules in priority order.

4. **`sales_data` partitioned by report period** — DSP reports can contain millions of rows per period. Partitioning by `report_period_start` enables efficient period-scoped queries and data retention management. The `is_matched` flag tracks whether each sales line has been linked to a title and contract for royalty calculation.

5. **`contracts.advance_recouped_cents`** — advance recoupment is a running balance: royalties earned reduce the unrecouped advance until it reaches zero. Storing the recouped amount on the contract row enables the royalty engine to check recoupment status before calculating net payable.

6. **`contracts.clause_extraction_json`** — AI-extracted clauses from PDF contracts are stored as JSONB, preserving the structured output (royalty rates, territory definitions, exclusivity, term dates) while the relational fields hold the canonical, human-verified values.

7. **`parties` with ISNI and IPI** — ISNI (ISO 27729) identifies contributors across systems; IPI (Interested Party Information) is the music-industry identifier used in CWR registrations. Both are indexed for interoperability with CMOs, DDEX messages, and industry registries.

8. **`titles.parent_title_id`** — enables hierarchical catalogues: a TV series contains episodes, a music album contains tracks, a book series contains volumes. The self-referencing FK supports recursive queries for catalogue browsing.

9. **`rights_grants.odrl_policy_json`** — stores the W3C ODRL machine-readable representation of the grant alongside the relational fields. This enables export of rights as ODRL policies for interoperability with content registries and AI-licensing frameworks.

10. **13 tables** — media rights management has many-to-many relationships at every level (titles ↔ contributors, contracts ↔ titles, rules ↔ payees). Normalisation prevents duplication and ensures the royalty calculation chain is traceable from sales line to statement.
