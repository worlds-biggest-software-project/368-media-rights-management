# Media Rights Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, vertical-agnostic platform for content licensing, rights tracking, and royalty calculation across music, film, publishing, games, and sports.

Media Rights Management is an open-source rights and royalties platform for content owners, publishers, distributors, and licensees. It targets the gap between spreadsheets and enterprise incumbents, giving mid-market content companies a single system to track rights grants across territories, formats, platforms, and time windows, and to compute royalties from reported sales data.

---

## Why Media Rights Management?

- Incumbents like Rightsline and Klopotek STREAM are enterprise-grade but carry opaque pricing and long, complex implementations that put them out of reach for mid-market teams.
- Most existing tools specialise in a single vertical: Mymediabox in consumer-products licensing, FilmTrack in film/TV, knk and Crealo in publishing, Curve and Reprtoir in music. Companies operating across verticals are forced to stitch multiple systems together.
- Mid-tier content companies are routinely left tracking rights manually in spreadsheets or legacy contract databases, leading to undetected rights conflicts, missed royalty payments, late renewals, and litigation.
- Public API documentation across the incumbent landscape is sparse, making developer-first integration into modern sales, CRM, and finance stacks difficult.
- Emerging needs — transparent rights provenance for AI-training-data licensing, real-time conflict detection during negotiation, AI-assisted contract review — are not well served by the existing market.

---

## Key Features

### Rights & Avails

- Structured rights catalogue per title with territory, platform, format, language, and time-window dimensions
- Avails search and conflict / holdback detection across windows and exclusivities
- Rights availability inquiry showing what is available, encumbered, or expired by title and territory
- REST API for avails queries and rights CRUD

### Contracts & Licensing Workflow

- Digital contract storage with structured metadata capture (parties, term, exclusivity, financial terms)
- NLP-based clause extraction from PDF contracts (royalty rates, territory definitions, exclusivity, term dates)
- Licence request intake, rights availability check, deal-memo approval, and agreement execution pipeline
- Obligation calendaring and renewal alerting

### Royalties & Statements

- Configurable royalty calculation engine: flat fee, percentage of receipts, escalating rates, co-author splits, advances, recoupment, minimum guarantees
- Automated royalty statement generation (PDF + CSV) and distribution on configurable cycles
- Multi-currency support with historical exchange-rate conversion
- Configurable royalty rule DSL rather than hard-coded rule types

### Sales Data, Revenue & Audit

- Sales-data ingestion from major DSP and retail channels
- Revenue tracking matched against contractual minimums and guarantees
- Anomaly detection on incoming sales statements (under-reporting, missed channels)
- Discrepancy flagging and full audit trail for every royalty calculation and rights change

### Access, Portals & Interoperability

- Role-based access for rights owners, licensees, payees, and auditors with data scoping per role
- Self-service licensee / payee portal with e-signature
- ONIX, DDEX, EIDR, CWR, EDItEUR import/export for industry interoperability
- Optional cryptographic audit trail for rights-of-record disputes

---

## AI-Native Advantage

AI is used to extract clauses and obligations from heterogeneous contract PDFs, auto-classify rights grants into structured dimensions, and answer natural-language avails queries such as "Can we stream Title X on TVOD in DACH after 2027-01-01?". Anomaly detection flags under-reporting and missed channels in incoming royalty statements, and rights-graph reasoning surfaces implicit conflicts including chain-of-title gaps. None of the surveyed incumbents offer NLP-based clause extraction, and AI-assisted contract review remains a documented underserved area in the market.

---

## Tech Stack & Deployment

The platform is designed as a cloud-native, multi-tenant SaaS with a self-host option. It exposes REST APIs for avails, rights, and contract management, supports SSO, and integrates with ERP and finance systems. Industry standards — DDEX, CWR, ONIX, EIDR, EDItEUR — are first-class for interoperability. An optional blockchain or cryptographic audit-trail layer is available for rights and payment provenance. The data model is intentionally vertical-agnostic so a single deployment can serve music, film, publishing, games, and sports catalogues, scaling to hundreds of thousands of titles with complex rights trees such as music sub-publishing deals.

---

## Market Context

The candidate-projects table classifies Media Rights Management with complexity 7, low domain availability, and medium demand. The incumbent landscape is dominated by enterprise-priced, vertical-specific SaaS — Rightsline, Klopotek STREAM, Vistex Counterpoint, Mymediabox, FilmTrack, knk, and others — with pricing typically opaque and out of reach for mid-market buyers. Primary buyers are mid-tier publishers, music labels and publishers, indie film distributors, sports and brand licensors, and content groups operating across more than one vertical.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
