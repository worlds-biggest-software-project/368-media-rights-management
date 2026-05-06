# Standards & API Reference

> Project: Media Rights Management · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO 15706 — International Standard Audiovisual Number (ISAN)** — https://www.iso.org/standard/79302.html — Persistent unique identifier for audiovisual works; foundational for cross-platform rights tracking of films and TV programmes.
- **ISO 21047 — International Standard Text Code (ISTC)** — https://www.iso.org/standard/57420.html — Persistent identifier for textual works; enables rights tracking independent of edition or format.
- **ISO 3901 — International Standard Recording Code (ISRC)** — https://www.iso.org/standard/64817.html — Identifier for sound and music-video recordings; central to music royalty matching.
- **ISO 15707 — International Standard Musical Work Code (ISWC)** — https://www.iso.org/standard/79394.html — Identifier for musical compositions (separate from recordings); essential for publishing-side royalty distribution.
- **ISO 26324 — Digital Object Identifier (DOI)** — https://www.iso.org/standard/43506.html — Generic persistent identifier; used for academic works, datasets, and increasingly for rights metadata.
- **ISO 27729 — International Standard Name Identifier (ISNI)** — https://www.iso.org/standard/44292.html — Persistent identifier for contributors (authors, performers, rights holders).
- **ISO 4217 — Currency Codes** — https://www.iso.org/iso-4217-currency-codes.html — Required for multi-currency royalty accounting.
- **ISO 3166 — Country Codes** — https://www.iso.org/iso-3166-country-codes.html — Territory definitions in rights grants.
- **ISO 639 — Language Codes** — https://www.iso.org/iso-639-language-code — Language-specific rights grants (dubbing, subtitling, translation rights).
- **ISO/IEC 27001 — Information Security Management** — https://www.iso.org/standard/27001 — Security baseline expected by enterprise rights-holders entrusting contracts and financials.

### W3C & IETF Standards

- **W3C ODRL — Open Digital Rights Language 2.2** — https://www.w3.org/TR/odrl-model/ — Formal language for expressing permissions, prohibitions, and obligations on digital assets; the closest thing to a machine-readable rights vocabulary recommendation.
- **W3C ODRL Vocabulary & Expression** — https://www.w3.org/TR/odrl-vocab/ — Companion vocabulary for ODRL policies.
- **W3C PROV-O Provenance Ontology** — https://www.w3.org/TR/prov-o/ — Useful for chain-of-title and rights-provenance modelling.
- **W3C Verifiable Credentials Data Model 2.0** — https://www.w3.org/TR/vc-data-model-2.0/ — For attestations of rights ownership and licence grants.
- **W3C Decentralized Identifiers (DIDs) 1.0** — https://www.w3.org/TR/did-core/ — Identifier layer for decentralised rights-holder identity.
- **RFC 7231 — HTTP/1.1 Semantics** — https://www.rfc-editor.org/rfc/rfc7231 — Baseline for REST API design.
- **RFC 9110 — HTTP Semantics** — https://www.rfc-editor.org/rfc/rfc9110 — Updated HTTP semantics superseding RFC 7231.
- **RFC 7519 — JSON Web Token (JWT)** — https://www.rfc-editor.org/rfc/rfc7519 — Token format for API authentication.
- **RFC 6749 — OAuth 2.0 Authorization Framework** — https://www.rfc-editor.org/rfc/rfc6749 — Delegated access for licensee/payee portals and third-party integrations.
- **RFC 7515 / 7516 / 7517 / 7518 — JOSE (JWS, JWE, JWK, JWA)** — https://www.rfc-editor.org/rfc/rfc7515 — Underpins signed/encrypted statements and contract attestations.

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — https://spec.openapis.org/oas/v3.1.0 — REST API contract definition.
- **JSON Schema 2020-12** — https://json-schema.org/draft/2020-12/release-notes — Validation of rights, contract, and royalty payloads.
- **GraphQL Spec (October 2021)** — https://spec.graphql.org/October2021/ — Alternative API surface, well-suited to graph-shaped rights data.
- **DDEX Standards (Digital Data Exchange)** — https://ddex.net/ — Family of XML standards for the music industry covering message exchange between labels, distributors, DSPs, and CMOs. Key messages:
  - **ERN — Electronic Release Notification** — https://kb.ddex.net/display/HBK/ERN+(Electronic+Release+Notification)+Knowledge+Base — Catalogue/release delivery to DSPs.
  - **DSR — Digital Sales Reporting** — https://kb.ddex.net/display/HBK/DSR+(Digital+Sales+Reporting)+Knowledge+Base — Sales/usage reports from DSPs back to rights holders.
  - **MWN — Musical Work Notification** — https://kb.ddex.net/ — Notifications between publishers, societies, and licensees.
  - **RDR — Recording Data and Rights** — https://kb.ddex.net/ — Recording metadata and rights claims.
- **CWR — Common Works Registration** — https://www.cisac.org/services/information-services/cis-net — CISAC standard for registering musical works with collective management organisations.
- **ONIX for Books 3.0** — https://www.editeur.org/93/Release-3.0-Downloads/ — EDItEUR XML standard for book product information including rights and territory data.
- **ONIX-PL — ONIX for Publication Licenses** — https://www.editeur.org/21/ONIX-PL/ — Expression of licence terms for publications.
- **EIDR — Entertainment Identifier Registry API** — https://www.eidr.org/technology/ — Registry and REST API for audiovisual content identifiers.
- **IPTC RightsML** — https://iptc.org/standards/rightsml/ — News-industry profile of ODRL for expressing rights on news content.
- **PLUS — Picture Licensing Universal System** — https://useplus.com/ — Standard licence vocabularies for image rights.
- **IIIF — International Image Interoperability Framework** — https://iiif.io/api/ — Relevant where image rights and presentation are linked.
- **MARC 21 / Linked Data BIBFRAME** — https://www.loc.gov/bibframe/ — Library/cultural-heritage rights metadata interop.
- **Schema.org CreativeWork & License** — https://schema.org/CreativeWork — Lightweight rights metadata for web/SEO contexts.

### Security & Authentication Standards

- **OAuth 2.1 (draft)** — https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/ — Consolidated OAuth best-practice profile.
- **OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html — Identity layer over OAuth for SSO with enterprise rights-holders.
- **FAPI 2.0 — Financial-grade API** — https://openid.net/specs/fapi-2_0-security.html — Higher-assurance API security profile relevant to royalty payment flows.
- **OWASP API Security Top 10 (2023)** — https://owasp.org/API-Security/editions/2023/en/0x00-header/ — API hardening guidance.
- **OWASP ASVS 4.0** — https://owasp.org/www-project-application-security-verification-standard/ — Application security verification baseline.
- **NIST SP 800-63-3 — Digital Identity Guidelines** — https://pages.nist.gov/800-63-3/ — Authentication assurance level reference.
- **NIST SP 800-53 Rev. 5** — https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final — Controls catalogue commonly referenced in enterprise procurement.
- **SOC 2 (AICPA TSC)** — https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2 — De facto enterprise audit standard for SaaS handling sensitive financial data.
- **GDPR (Regulation (EU) 2016/679)** — https://eur-lex.europa.eu/eli/reg/2016/679/oj — Personal data of payees, contributors, and licensee contacts.
- **eIDAS Regulation (EU) 910/2014** — https://eur-lex.europa.eu/eli/reg/2014/910/oj — Legal framework for electronic signatures on contracts in the EU.
- **US ESIGN Act / UETA** — https://www.fdic.gov/resources/supervision-and-examinations/consumer-compliance-examination-manual/documents/10/x-3-1.pdf — US e-signature legal basis.

### MCP Server Specifications

- **Model Context Protocol Specification** — https://modelcontextprotocol.io/specification — Protocol for exposing tools and resources to LLM clients; relevant for an MCP server exposing rights-catalogue queries, avails checks, and contract clause lookups to AI agents.
- **MCP Reference Servers** — https://github.com/modelcontextprotocol/servers — Reference implementations to model an MCP server for media rights operations (e.g., a `rights-avails` tool, a `royalty-calc` tool, a `contracts` resource).

## Similar Products — Developer Documentation & APIs

### Rightsline
- **Description:** Enterprise rights, contract, and royalty management platform for film, TV, sports, and publishing.
- **API Documentation:** https://www.rightsline.com/api/ (developer access typically gated to customers)
- **SDKs/Libraries:** None public; REST API consumable from any language
- **Developer Guide:** Customer portal only
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0 / API key (per customer configuration)

### Mymediabox
- **Description:** Rights, royalty, and product-approval suite for consumer-products licensing.
- **API Documentation:** Customer-only (https://www.mymediabox.com/)
- **SDKs/Libraries:** None public
- **Developer Guide:** Not public
- **Standards:** REST/JSON; CSV/Excel templates for licensee data
- **Authentication:** API key

### FilmTrack
- **Description:** Rights, sales, and royalty platform for film and TV distributors.
- **API Documentation:** Customer-only (https://www.filmtrack.com/)
- **SDKs/Libraries:** None public
- **Developer Guide:** Not public
- **Standards:** REST
- **Authentication:** API key

### Vistex Counterpoint Suite
- **Description:** High-volume IP, rights, and royalty processing for major music and publishing groups.
- **API Documentation:** SAP-integrated; documentation via SAP partner portal (https://www.vistex.com/)
- **SDKs/Libraries:** SAP-native (ABAP, BAPIs, OData services)
- **Developer Guide:** SAP partner network
- **Standards:** OData, SAP IDoc/BAPI, REST
- **Authentication:** SAP-managed (SAML, OAuth via SAP)

### Curve Royalty Systems
- **Description:** SaaS royalty accounting for music labels and publishers, with broad DSP statement ingestion.
- **API Documentation:** https://curveroyalties.com/ (customer-only API docs)
- **SDKs/Libraries:** None public
- **Developer Guide:** Customer onboarding
- **Standards:** DDEX DSR ingestion; REST/JSON APIs
- **Authentication:** API key / OAuth

### Reprtoir
- **Description:** Music catalogue, distribution, and royalty platform for indie labels and publishers.
- **API Documentation:** https://www.reprtoir.com/ (customer-gated)
- **SDKs/Libraries:** None public
- **Developer Guide:** In-app
- **Standards:** DDEX ERN for distribution; REST/JSON
- **Authentication:** OAuth / API key

### MetaComet RoyaltyShare
- **Description:** Royalty calculation and statement distribution service for publishers and creators.
- **API Documentation:** Customer-only (https://metacomet.com/)
- **SDKs/Libraries:** None public
- **Developer Guide:** Bureau / customer support
- **Standards:** REST; CSV imports
- **Authentication:** API key

### EIDR (Entertainment Identifier Registry)
- **Description:** Open registry of identifiers for audiovisual works, releases, and edits.
- **API Documentation:** https://www.eidr.org/documents/ (REST API guides)
- **SDKs/Libraries:** Java SDK published by EIDR
- **Developer Guide:** https://www.eidr.org/technology/
- **Standards:** REST/XML; ISO 26324 (DOI)
- **Authentication:** Membership credentials / mTLS

### MLC (Mechanical Licensing Collective) Public API
- **Description:** US-mandated mechanical-rights collective; publishes claim/match data and accepts registrations.
- **API Documentation:** https://www.themlc.com/api-documentation
- **SDKs/Libraries:** None official
- **Developer Guide:** https://www.themlc.com/
- **Standards:** REST/JSON; DDEX MWN
- **Authentication:** API key

### ASCAP / BMI / SOCAN / PRS — Performing Rights Society APIs
- **Description:** Member CMOs offering catalogue lookup and (in some cases) registration APIs (e.g., PRS for Music ICE platform).
- **API Documentation:** Society-specific developer portals
- **SDKs/Libraries:** None standard
- **Developer Guide:** Society-specific
- **Standards:** CWR (Common Works Registration); REST in newer endpoints
- **Authentication:** Member credentials / OAuth

### Spotify for Artists / Apple Music for Artists / YouTube Content ID
- **Description:** Distribution-platform reporting feeds — primary sources of usage data feeding royalty engines.
- **API Documentation:** Provider-specific (e.g., https://developer.spotify.com/, https://developers.google.com/youtube/partner)
- **SDKs/Libraries:** Provider SDKs (JS, Python, Java)
- **Developer Guide:** Provider portals
- **Standards:** OAuth 2.0; JSON; some DDEX DSR delivery
- **Authentication:** OAuth 2.0

### DocuSign / Adobe Sign
- **Description:** E-signature platforms commonly integrated for contract execution.
- **API Documentation:** https://developers.docusign.com/, https://developer.adobe.com/document-services/docs/overview/adobe-sign/
- **SDKs/Libraries:** Official SDKs in multiple languages
- **Developer Guide:** Provider portals
- **Standards:** REST/JSON; OAuth 2.0; eIDAS, ESIGN Act compliance
- **Authentication:** OAuth 2.0; JWT

## Notes

- The single biggest interoperability lever in this domain is **DDEX** (music) and **ONIX/EDItEUR** (publishing). An open-source rights-management platform that offers first-class import/export of these formats out of the box would have an immediate technical advantage over many incumbents whose support is partial or upgrade-gated.
- **W3C ODRL** is the only mature, vendor-neutral, machine-readable rights expression language. Adopting ODRL as the internal canonical representation of rights grants would future-proof interoperability with content registries and emerging AI-training-data licence frameworks (e.g., the work emerging from C2PA and the IETF AI-Preferences working group).
- **AI-training-data licensing** is a fast-moving area without a settled standard as of mid-2026. Worth monitoring: C2PA content credentials, IETF AI-Preferences (`ai.txt`), and proposed extensions to robots.txt for training opt-outs. A rights-management platform that treats "AI training" as a first-class right type alongside "streaming" and "broadcast" would address an emerging customer need.
- **MCP server exposure** of avails queries, royalty calculations, and contract clause search is an under-explored opportunity to make rights data directly consumable by LLM-based assistants for legal, sales, and finance teams.
- Public, openly documented APIs are rare among incumbents; documentation is typically gated behind customer agreements. This represents both a barrier to integration ecosystems and an opportunity for an open-API-first newcomer.
