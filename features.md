# Media Rights Management — Feature & Functionality Survey

> Candidate #368 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Rightsline | Enterprise SaaS | Commercial / proprietary | https://www.rightsline.com/ |
| Mymediabox (Mediabox-RM) | Enterprise SaaS | Commercial / proprietary | https://www.mymediabox.com/ |
| FilmTrack (City National 2.0) | Enterprise SaaS | Commercial / proprietary | https://www.filmtrack.com/ |
| knk Publishing Rights & Royalties | Enterprise SaaS (D365) | Commercial / proprietary | https://www.knkpublishingsoftware.com/rights-and-royalties/ |
| Klopotek STREAM | Enterprise SaaS | Commercial / proprietary | https://www.klopotek.com/ |
| Crealo | SaaS (publishing) | Commercial / proprietary | https://www.crealo.com/ |
| Molten Cloud | Modern SaaS | Commercial / proprietary | https://www.moltencloud.com/ |
| FilmChain | Blockchain platform | Commercial / proprietary | https://filmchain.io/ |
| Vistex Counterpoint Suite | Enterprise (royalties/IP) | Commercial / proprietary | https://www.vistex.com/ |
| MetaComet RoyaltyShare | SaaS royalties | Commercial / proprietary | https://metacomet.com/ |
| Curve Royalty Systems | SaaS royalties (music) | Commercial / proprietary | https://curveroyalties.com/ |
| Reprtoir | Music rights/catalogue SaaS | Commercial / SaaS | https://www.reprtoir.com/ |

## Feature Analysis by Solution

### Rightsline

**Core features**
- Centralised rights catalogue (titles, talent, episodes, components)
- Avails / rights availability search across territories, windows, languages, platforms
- Contract lifecycle: deal memos, long-form contracts, amendments
- Royalty processing engine with participations and recoupment
- Sales/order entry and invoicing
- Workflow and approvals; digital signature integration
- Reporting and BI dashboards

**Differentiating features**
- Highly configurable rights data model adaptable to film, TV, sports, publishing
- Open REST API and integration framework

**UX patterns**
- Web-based grid + form views; admin-driven configuration of fields and pick-lists
- Heavy use of saved filters/queries for avails

**Integration points**
- REST API; integrations with Salesforce, NetSuite, SAP; SSO

**Known gaps**
- Steep learning curve; long implementation cycles
- Pricing opaque and out of reach for mid-market

**Licence / IP notes**
- Proprietary, commercial. No open API spec published.

### Mymediabox (Mediabox-RM)

**Core features**
- Rights/contract management for licensing (brand, sports, entertainment merchandising)
- Royalty calculation and statement generation
- Product approval workflow (PA) — adjacent module
- Multi-currency, multi-tax handling
- Licensee self-service portal for sales reporting

**Differentiating features**
- Tightly coupled royalty + product-approval suite — the de facto standard in consumer-products licensing

**UX patterns**
- Licensee portal for direct royalty sales submission
- Form-driven contract entry with templated clauses

**Integration points**
- Imports from licensee sales data (CSV/Excel templates)
- API for ERP integration

**Known gaps**
- Limited fit for music publishing or film distribution
- UI considered dated by users

**Licence / IP notes**
- Proprietary, commercial.

### FilmTrack

**Core features**
- Film/TV avails and rights tracking
- Deal management, holdbacks, exclusivity windows
- Royalties and participations
- Sales pipeline and contract management
- Servicing/delivery tracking

**Differentiating features**
- Backed by City National Bank's entertainment division; tight ties to entertainment finance
- Strong holdback and conflict-detection logic

**UX patterns**
- Calendar/timeline visualisations of windowing

**Integration points**
- ERP and royalty bank integration

**Known gaps**
- Film/TV-only focus; not suitable for music or publishing

**Licence / IP notes**
- Proprietary.

### knk Publishing — Rights & Royalties

**Core features**
- Rights inventory per title with territory/language/format dimensions
- Royalty calculation with author splits, escalators, advances/recoupment
- Statement generation in multiple currencies and languages
- Subrights selling workflow (translation, audio, serial, film options)
- Built on Microsoft Dynamics 365 Business Central

**Differentiating features**
- Native ERP integration (D365) — finance, AP/AR, GL flow native
- Translation rights and book-fair workflows tailored to publishing

**UX patterns**
- D365 standard UI; Office 365 integration

**Integration points**
- D365 ecosystem; Power BI; Outlook
- ONIX import/export for bibliographic data

**Known gaps**
- Strong publishing fit; weak elsewhere
- Requires D365 commitment

**Licence / IP notes**
- Proprietary; built on Microsoft platform stack.

### Klopotek STREAM

**Core features**
- Rights & royalties for global publishers
- Contract templates and clause libraries
- Royalty calculation, statements, recoupment
- Subrights, permissions, and licensing workflows
- Title and product master data management

**Differentiating features**
- Long-established publishing-industry de facto choice (large trade and academic houses)
- Permissions module for rights-clearance requests

**UX patterns**
- Web (STREAM) replacing legacy thick-client (RPM)

**Integration points**
- ONIX, EDItEUR standards; SAP/finance connectors

**Known gaps**
- Implementation cost and complexity
- Migration from legacy RPM ongoing for many customers

**Licence / IP notes**
- Proprietary.

### Crealo

**Core features**
- Catalogue, contract and royalty management for publishers
- Product master data
- Sales reporting ingestion

**Differentiating features**
- Lightweight, modern SaaS aimed at small/mid publishers

**UX patterns**
- Modern web UI; faster onboarding than enterprise incumbents

**Integration points**
- ONIX import/export
- API for ERP

**Known gaps**
- Smaller feature breadth; publishing-only

**Licence / IP notes**
- Proprietary.

### Molten Cloud

**Core features**
- Combined rights, royalties, and content operations
- Multi-vertical (publishing, media)
- Cloud-native, multi-tenant

**Differentiating features**
- Configurable rights data model in a modern stack
- Marketed at mid-market replacing legacy systems

**UX patterns**
- Modern web SaaS; configurable dashboards

**Integration points**
- REST APIs; SSO

**Known gaps**
- Newer entrant; smaller customer base and ecosystem
- Limited public API documentation

**Licence / IP notes**
- Proprietary.

### FilmChain

**Core features**
- Revenue collection and split distribution for film/TV/music
- Smart-contract-based waterfall payouts
- Recoupment and participation tracking
- Real-time dashboards for producers, financiers, talent

**Differentiating features**
- Blockchain-backed audit trail of payments
- Automated split payouts to participants

**UX patterns**
- Stakeholder dashboards; transparent waterfall visualisation

**Integration points**
- Bank/payment rails; collection-account-management replacement

**Known gaps**
- Focused on collection/disbursement, not contract/avails management
- Blockchain layer adds complexity that many studios resist

**Licence / IP notes**
- Proprietary; uses public blockchain elements (specifics not fully disclosed).

### Vistex Counterpoint Suite

**Core features**
- IP/rights management
- Royalty processing at very high volume
- Contract management
- SAP-integrated finance flows

**Differentiating features**
- Designed for huge catalogues (major music/publishing groups)
- Deep SAP integration

**UX patterns**
- SAP-style enterprise UX

**Integration points**
- Native SAP; broad ERP connectivity

**Known gaps**
- Enterprise-only; long implementations
- Not suited to mid-market

**Licence / IP notes**
- Proprietary.

### MetaComet RoyaltyShare

**Core features**
- Royalty calculation, statement generation, payee portal
- Imports sales data from publishing channels and DSPs
- Support for advances, recoupment, splits

**Differentiating features**
- Hosted royalty bureau model in addition to software
- Strong payee experience (portal, electronic statements)

**UX patterns**
- Payee self-service for statements and tax forms

**Integration points**
- Importers for major DSPs and retail channels
- 1099/tax reporting integrations (US)

**Known gaps**
- Royalty-centric; lighter on rights/avails

**Licence / IP notes**
- Proprietary.

### Curve Royalty Systems

**Core features**
- Royalty accounting for music labels and publishers
- DSP statement ingestion and matching
- Mechanical, performance, neighbouring-rights handling
- Recoupment and advance tracking

**Differentiating features**
- Specialised in music DSP data normalisation across hundreds of source formats

**UX patterns**
- Modern SaaS with statement preview and drill-down to source lines

**Integration points**
- Importers for Spotify, Apple, Amazon, YouTube, Beatport, Bandcamp, etc.

**Known gaps**
- Music vertical only

**Licence / IP notes**
- Proprietary.

### Reprtoir

**Core features**
- Music catalogue and metadata management
- Rights ownership tracking (master + composition)
- Royalty distribution
- Asset distribution to DSPs
- Collaboration with co-writers, labels, sub-publishers

**Differentiating features**
- Combined catalogue/distribution/royalties for indie music

**UX patterns**
- Modern web app with collaboration model

**Integration points**
- DDEX-based DSP delivery
- ISRC/ISWC lookups

**Known gaps**
- Indie-music focus; not for film/publishing

**Licence / IP notes**
- Proprietary; uses DDEX standards.

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Rights catalogue with title/territory/window/format/language/platform dimensions
- Avails search and conflict/holdback detection
- Contract storage and metadata capture (parties, term, exclusivity, financial terms)
- Royalty calculation engine (flat, %, escalators, splits, recoupment, advances, MGs)
- Royalty statement generation and distribution
- Multi-currency with historical FX
- Sales-data ingestion from common channels (DSPs, retailers, distributors)
- Role-based access for owners, licensees, payees, auditors
- Audit trail for every calculation and rights change
- Reporting / BI dashboards

### Differentiating Features
- NLP-based clause extraction from PDF contracts (not present in incumbents)
- Real-time avails API consumable by sales/CRM systems
- Cryptographic / immutable audit trail of rights and payments
- Smart-contract-driven automatic disbursement (FilmChain pattern)
- Vertical-agnostic data model (most incumbents specialise)
- Self-service licensee/payee portals with electronic signatures and tax forms
- DDEX/ONIX/EIDR/CWR native interoperability
- Configurable royalty DSL rather than hard-coded rule types

### Underserved Areas / Opportunities
- Mid-market price point (between spreadsheets and Rightsline/Klopotek)
- Cross-vertical platform (most tools are music-only, publishing-only, or film-only)
- Open APIs and developer-first integrations
- Independent creators and small catalogues priced per-title rather than enterprise seat
- AI-assisted contract review and obligation extraction
- Transparent rights provenance for AI-training-data licensing — a fast-emerging need
- Real-time conflict detection during negotiation, not post-hoc

### AI-Augmentation Candidates
- Clause and obligation extraction from heterogeneous contract PDFs
- Auto-classification of rights grants into structured dimensions
- Anomaly detection on incoming sales/royalty statements (under-reporting, missed channels)
- Natural-language avails queries ("Can we stream Title X on TVOD in DACH after 2027-01-01?")
- Statement narrative generation in payee-friendly language
- Translation of contracts and statements across languages
- Predictive renewal recommendations based on historical performance
- Rights-graph reasoning to surface implicit conflicts (e.g., chain-of-title gaps)

## Legal & IP Summary

All twelve solutions surveyed are proprietary, commercial products with no open-source equivalents identified at parity. No patents were specifically discovered that would block a clean-room implementation of standard rights/royalty features, but practitioners should be aware that royalty-calculation methods and avails-engine implementations have been the subject of past patent activity in the licensing-software industry — a freedom-to-operate review is recommended before commercialisation. Industry data formats relevant to interoperability — DDEX, CWR, ONIX, EIDR, EDItEUR — are open standards governed by neutral bodies and are safe to implement against. Blockchain-based audit-trail patterns (à la FilmChain) are not believed to be patent-encumbered in their generic form, though specific smart-contract designs may be. Contract text ingested by a system remains the IP of the contracting parties; any AI feature that fine-tunes models on customer contracts must offer per-tenant isolation and explicit consent. No copyrighted material from the surveyed vendors' documentation was reproduced in this survey; all observations are paraphrased from publicly accessible product pages and analyst write-ups.

## Recommended Feature Scope

**Must-have (MVP)**
- Rights catalogue with title/territory/window/format/language dimensions and conflict detection
- Contract storage with structured metadata capture (parties, term, exclusivity, financials)
- Royalty calculation engine supporting flat, %, escalators, splits, advances, recoupment
- Royalty statement generation (PDF + CSV) with multi-currency and historical FX
- Sales-data importers for at least three major DSP/retail formats
- Role-based access (owner / licensee / payee / auditor) with full audit trail
- REST API for avails queries and rights CRUD

**Should-have (v1.1)**
- AI clause extraction from PDF contracts
- Self-service licensee/payee portal with e-signature
- Anomaly detection on incoming sales statements
- ONIX / DDEX / EIDR import/export
- Configurable royalty rule DSL
- Renewal/obligation calendar with alerts

**Nice-to-have (backlog)**
- Cryptographic audit trail (optional, opt-in)
- Smart-contract disbursement integration
- Natural-language avails search
- AI-training-data licensing module (provenance + permission tracking)
- Multi-language contract translation
- Predictive renewal recommendations
