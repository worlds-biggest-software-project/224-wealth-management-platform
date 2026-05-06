# Standards & API Reference

> Project: Wealth Management Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 20022 — Universal Financial Industry Message Scheme**
- URL: https://www.iso.org/standard/20022-1
- The dominant global standard for electronic data interchange between financial institutions. Covers payment transactions, securities settlement, and portfolio transfer messaging. SWIFT mandated full migration by November 2025 for cross-border payments; relevant to custody bank connectivity, trade instruction, and position reconciliation in wealth platforms. The Federal Reserve adopted ISO 20022 for Fedwire Funds Service in July 2025.

**ISO/IEC 27001:2022 — Information Security Management Systems (ISMS)**
- URL: https://www.iso.org/standard/27001
- Internationally recognised certification framework for information security management. Required or strongly expected by institutional clients and custodian partners evaluating a wealth platform vendor's data handling. Covers risk assessment, access controls, incident response, and supplier relationships. Complements SOC 2 (preferred for US clients) and aligns with GDPR requirements.

**ISO/IEC 27017:2015 — Cloud Security Controls**
- URL: https://www.iso.org/standard/43757.html
- Extension of ISO 27001 specifically addressing cloud service environments. Relevant to wealth platforms deployed as SaaS that process sensitive client financial data across cloud infrastructure. Defines security controls for both cloud service providers and cloud service customers.

**ISO/IEC 27701:2019 — Privacy Information Management**
- URL: https://www.iso.org/standard/71670.html
- Extension to ISO 27001 for privacy management, aligning with GDPR and other data protection frameworks. Directly applicable when handling personally identifiable financial data (account balances, transaction history, beneficiary information) for wealth management clients.

**ISO/FDIS 32212 — Sustainable Finance: Net Zero Transition Planning for Financial Institutions**
- URL: https://www.iso.org/standard/32212
- Emerging standard for sustainable finance reporting. Increasingly relevant to wealth platforms as ESG portfolio screening, impact reporting, and sustainability disclosures become standard client expectations and regulatory requirements in EU and UK markets.

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Foundation for delegated authorisation across all wealth platform APIs and third-party integrations. Required for secure, consent-based access to custodian data, financial planning tools, and account aggregation services. All major wealth management APIs (Schwab, Plaid, Addepar, eMoney) use OAuth 2.0.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard for compact, URL-safe token representation used in OAuth 2.0 flows. Widely used for authentication and secure transmission of claims between identity providers, platform APIs, and third-party integrations in wealth management SaaS environments.

**RFC 7636 — Proof Key for Code Exchange (PKCE)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Security extension to OAuth 2.0's authorisation code flow, mitigating authorisation code interception attacks. Mandatory under FAPI 2.0 (see below) and required by platforms such as Schwab Trader API for public client applications.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Standard for expressing typed links in HTTP responses. Underpins pagination patterns (via `Link` headers) used by REST APIs including Addepar's JSON:API-based portfolio endpoints.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 enabling single sign-on and identity federation. Required for advisor and client portal login, multi-custodian SSO flows, and integration with enterprise identity providers (Azure AD, Okta). Used in Orion's SAML/SSO integrations and eMoney's developer authentication.

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- Industry-standard format for describing RESTful API interfaces. OAS 3.1 achieves full JSON Schema Draft 2020-12 compatibility. Addepar, Plaid, Schwab, and eMoney all publish OAS-compliant API documentation, enabling client SDK generation and automated testing. A wealth platform should publish its own OAS 3.1 API contract for third-party integrators.

**JSON:API Specification**
- URL: https://jsonapi.org/
- Specification for building consistent REST APIs that return structured JSON responses with relationships, pagination, filtering, and sparse fieldsets. Addepar's API is built on JSON:API, making it a de-facto standard within the wealth management technology stack.

**XBRL (eXtensible Business Reporting Language)**
- URL: https://www.xbrl.org/the-standard/what/
- International standard for digital financial reporting. Required for SEC EDGAR filings (Forms 10-K, 10-Q, 8-K) and FINRA regulatory submissions. Relevant to wealth platforms that support institutional clients with reporting or compliance obligations.

**FIX Protocol (Financial Information eXchange)**
- URL: https://www.fixtrading.org/
- De-facto messaging standard for electronic trade communication between broker-dealers, exchanges, and order management systems. Required for integrations involving trade execution, order routing, and post-trade reporting. The FIX Trading Community has published explicit MiFID II mapping guidelines for FIX tag populations.

**Financial Data Exchange (FDX) API Standard**
- URL: https://financialdataexchange.org/
- US nonprofit standard now covering investment accounts, holdings, transactions, tax data, and insurance. Over 62 million consumer accounts connected as of 2025. Recognised by the CFPB as the leading open banking specification in the US. Wealth platforms building account aggregation should implement FDX-compliant data sharing.

**OpenWealth API Standard**
- URL: https://openwealth.ch/
- The global open API standard specifically developed for wealth management connectivity between custody banks and WealthTech providers. Platform-agnostic REST APIs covering three domains: Custody Services (position and transaction data), Customer Management (KYC and onboarding), and Order Placement (stock exchange, OTC, and treasury orders). On the verge of receiving official recognition under the EU's Financial Data Access (FiDA) regulation.

### Security & Authentication Standards

**FAPI 2.0 — Financial-grade API Security Profile**
- URL: https://openid.net/specs/fapi-security-profile-2_0-final.html
- OpenID Foundation security profile built on OAuth 2.0 and OpenID Connect for high-value, high-risk API interactions in financial services. Mandates Pushed Authorization Requests (PAR), PKCE, and Demonstration of Proof-of-Possession (DPoP). Increasingly required by regulators globally — Colombia mandated FAPI 2.0 in 2024, and EU open finance frameworks are expected to follow.

**PCI DSS v4.0**
- URL: https://www.pcisecuritystandards.org/standards/
- Payment Card Industry Data Security Standard. Applies to any wealth platform that processes, stores, or transmits cardholder data (e.g., ACH funding, advisory fee payments). Scope can be reduced to near-zero through tokenisation and hosted payment fields. Validated annually.

**NIST Cybersecurity Framework 2.0**
- URL: https://www.nist.gov/cyberframework
- Voluntary US framework for managing cybersecurity risk through five functions: Identify, Protect, Detect, Respond, Recover. Increasingly expected by enterprise and institutional clients when evaluating fintech vendors. Aligns with SEC's cybersecurity risk management rules for investment advisers.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa-cima.com/resources/landing/soc-2
- US-centric audit standard evaluating security, availability, processing integrity, confidentiality, and privacy controls. US institutional clients and RIA custodians (Fidelity, Schwab, Pershing) typically require SOC 2 Type II reports from technology vendors before onboarding.

**EU DORA — Digital Operational Resilience Act (Regulation 2022/2554)**
- URL: https://www.eba.europa.eu/regulation-and-policy/operational-resilience/digital-operational-resilience-act-dora
- EU regulation effective January 2025 requiring financial entities and their critical ICT third-party providers to demonstrate operational resilience. Directly affects EU-facing wealth platforms as vendors to banks and investment firms. Mandates ICT risk management frameworks, incident reporting, and third-party supplier assessments.

**MiFID II / MiFIR (EU Directive 2014/65/EU)**
- URL: https://www.esma.europa.eu/rules-databases/regulatory-activities/mifid-ii
- EU framework governing investment services. Mandates best-execution reporting, product transparency (KID/KIID documents), client categorisation (retail/professional/eligible counterparty), and suitability assessments before investment advice or portfolio management. The FCA has onshored equivalent rules post-Brexit. Directly shapes the data models and workflow logic of a compliant EU wealth platform.

**SEC Regulation Best Interest (Reg BI) — Rule 17 CFR 240.15l-1**
- URL: https://www.sec.gov/info/smallbus/secg/regulation-best-interest
- US standard requiring broker-dealers and investment advisers to act in retail clients' best interests. Drives suitability assessment workflows, disclosure requirements (Form CRS), and conflict-of-interest management in US-facing wealth platforms.

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Open standard introduced by Anthropic (November 2024) for connecting LLM-based AI systems to external tools and data sources via standardised JSON-RPC 2.0 messages. Already adopted in production by Evergreen Wealth, Bloomberg, and Saxo Bank for AI-assisted financial workflows. Flanks has launched an MCP server for wealth management that connects banking and investment data to AI assistants (Claude Desktop, ChatGPT, Cursor). A wealth management platform exposing an MCP server would enable AI agents to query portfolio analytics, run planning scenarios, and retrieve client data in a standardised way.

---

## Similar Products — Developer Documentation & APIs

### Addepar

- **Description:** Enterprise wealth management platform for aggregating, analysing, and reporting complex multi-asset portfolios. Used by RIAs, family offices, and private banks.
- **API Documentation:** https://developers.addepar.com/
- **API Reference:** https://developers.addepar.com/reference/introduction
- **SDKs/Libraries:** Code examples provided in cURL, Node.js, Ruby, JavaScript, and Python via the developer portal
- **Standards:** REST, JSON:API specification, HTTPS-only
- **Authentication:** API Key-based (per the portal onboarding flow)
- **Key Endpoints:** Portfolio API, Entities API, Transactions API, Positions API, Users API, Jobs API (for async large-data queries), Files API, Attributes API, Groups API, Roles API, Entity Types API

### Orion Advisor Tech

- **Description:** All-in-one RIA platform covering portfolio accounting, trading, billing, compliance, and client reporting. Exposes 100% of its data via REST APIs.
- **API Documentation / Developer Portal:** https://developers.orionadvisor.com/
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** Basic Auth, OAuth 2.0, SAML 2.0 SSO (Azure AD, Okta), and impersonation flows
- **Key Capabilities:** Portfolio accounting, Eclipse trading/rebalancing, model management, tax-loss harvesting, billing, CRM data sync, flat-file SFTP delivery for bulk extracts, webhook notifications for real-time events

### Charles Schwab Trader API

- **Description:** Official brokerage API for self-directed accounts and RIA integrations, replacing the legacy TD Ameritrade API (shut down May 2024).
- **API Documentation:** https://developer.schwab.com/
- **Authentication Guide:** https://developer.schwab.com/user-guides/get-started/authenticate-with-oauth
- **Community SDKs:** Python (schwab-py), R (schwabr), TypeScript (schwab-api on GitHub)
- **Standards:** REST/JSON, OAuth 2.0 with three-legged authorisation code flow, PKCE
- **Authentication:** OAuth 2.0; access tokens valid 30 minutes, refresh tokens valid 7 days
- **Key Endpoints:** Accounts and Trading Production (positions, orders, cash), Market Data Production (quotes, price history, daily movers)

### Plaid Investments API

- **Description:** Financial data aggregation covering investment holdings, securities, and 24 months of transaction history across 2,400+ US and Canadian institutions.
- **API Documentation:** https://plaid.com/docs/investments/
- **API Reference:** https://plaid.com/docs/api/products/investments/
- **SDKs/Libraries:** Official server-side client libraries for Node, Python, Ruby, Go, Java
- **Standards:** REST/JSON, FDX-aligned data model, OAuth 2.0 (via Plaid Link flow)
- **Authentication:** API Keys + client_id; end-user consented via Plaid Link
- **Key Endpoints:** `/investments/holdings/get`, `/investments/transactions/get`, `/investments/refresh`

### Morningstar Direct Web Services

- **Description:** Investment data, research, and calculation engine APIs for powering digital platforms with fund analytics, portfolio X-Ray, risk scoring, and ESG screening.
- **API Documentation:** https://developer.morningstar.com/direct-web-services/documentation/overview/about
- **Developer Portal:** https://developer.morningstar.com/
- **SDKs/Libraries:** Python (`morningstar-data` package on PyPI), Excel API add-in
- **Standards:** REST/JSON, OAuth 2.0 (via Morningstar API Centre at apicenter.morningstar.com)
- **Authentication:** OAuth 2.0 client credentials
- **Key Capabilities:** Portfolio Analysis X-Ray, fund screening, risk scoring, ESG/sustainability screener, editorial summarisation, AI Intelligence Engine (generative AI building blocks)

### eMoney Advisor Developer API

- **Description:** Financial planning and account aggregation APIs enabling third parties to embed eMoney's planning engine and data aggregation into their own client experiences.
- **Developer Portal:** https://developer.emoneyadvisor.com/
- **Getting Started:** https://developer.emoneyadvisor.com/getting-started
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0 with API access controlled via partner agreement
- **Key API Packages:**
  - *Core API* — data aggregation (connecting client financial accounts), user and household management
  - *Expanded Planning API* — read/write access to full planning data, real-time data transfers, planning projections and scenario calculations
- **Notable Integrations:** Tolerisk, YCharts, Vanilla, Wealth.com, Luminary, Zocks, Jump AI, Black Diamond, Asset-Map, FP Alpha

### OpenWealth API (SIX Group / OpenWealth Association)

- **Description:** Open standard REST APIs for connectivity between custody banks and WealthTech providers. Platform-agnostic. Sandbox available for testing.
- **Official Site:** https://openwealth.ch/
- **GitHub:** https://github.com/OpenWealth
- **Sandbox:** https://openwealth-portal.apps.ndgit.com/
- **SIX Group Implementation:** https://blink.six-group.com/en/services/custody-services
- **Standards:** REST, OpenAPI-compliant specifications, platform-agnostic
- **Authentication:** OAuth 2.0 (per sandbox documentation)
- **API Domains:** Custody Services (positions, transactions, securities, cash, loans), Customer Management (master data, KYC, documentation), Order Placement (exchange, OTC, and treasury orders with real-time status)

### Fidelity Investments — Integration Xchange / Wealthscape

- **Description:** Custodial data and workflow APIs for RIAs custodying with Fidelity. Integration Xchange provides access to 400+ APIs and connections with 100+ third-party technology providers.
- **Integration Xchange:** https://clearingcustody.fidelity.com/app/item/RD_9883092/integration-xchange.html
- **eMoney APIs (Fidelity subsidiary):** https://developer.emoneyadvisor.com/
- **Standards:** REST/JSON (Wealthscape APIs), SFTP data feeds (for historical data connectivity)
- **Authentication:** API Key and OAuth 2.0 depending on integration type; custom partner agreements required
- **Key Capabilities:** Account data (positions, transactions, balances), trading workflows, data feed subscriptions, client portal white-labelling, financial planning embedding via eMoney

### Nitrogen (formerly Riskalyze)

- **Description:** Risk tolerance assessment and proposal generation platform. Provides a public REST API for embedding risk scoring into advisor workflows and third-party applications.
- **Developer API:** https://developers.riskalyze.com/
- **Postman Workspace:** https://www.postman.com/riskalyze/workspace/riskalyze-s-public-workspace/documentation/
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** OAuth 2.0 with authorisation code flow
- **Key Capabilities:** Risk number assessment, proposal generation, portfolio stress testing, account data retrieval

---

## Notes

**Evolving regulatory landscape:** The EU's Financial Data Access (FiDA) Regulation — expected to enter into force in 2026–2027 — will extend open banking principles to investment accounts, insurance, and pension data, requiring financial institutions to expose standardised APIs for third-party access. OpenWealth is positioning its standard as the candidate FiDA API schema for wealth management data. UK and US equivalents (Smart Data, CFPB Section 1033 rule) are following similar trajectories.

**AI integration layer:** MCP is rapidly becoming the integration standard for AI-native financial workflows. Flanks, Bloomberg, and Saxo Bank are among early movers. A wealth management platform should consider exposing an MCP server as a first-class integration surface alongside its REST API — this allows AI agents to query portfolio data, run scenarios, and surface client insights without custom adapter code.

**Custodian API maturity varies widely:** Schwab's Trader API is production-ready with a strong developer ecosystem. Fidelity and Pershing operate via partner programmes requiring commercial agreements rather than self-service onboarding. This asymmetry has created demand for intermediary aggregators (Plaid, SnapTrade, Wealthica) that normalise multi-custodian data access.

**OpenWealth vs FDX:** OpenWealth is the leading standard in Europe and Switzerland; FDX dominates in North America. A globally-positioned platform needs to support both, plus direct custodian API integrations for markets where neither standard is widely adopted.
