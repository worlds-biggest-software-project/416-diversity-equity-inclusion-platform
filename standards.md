# Standards & API Reference

> Project: Diversity, Equity & Inclusion Platform · Generated: 2026-05-07

---

## Industry Standards & Specifications

### Regulatory Reporting Frameworks

**EU Corporate Sustainability Reporting Directive (CSRD) — ESRS S1: Own Workforce**
- URL: https://finance.ec.europa.eu/financial-markets/company-reporting-and-auditing/company-reporting/corporate-sustainability-reporting_en
- ESRS S1 is the mandatory disclosure standard for own workforce reporting under CSRD, covering working conditions, social dialogue, diversity, equal treatment, health and safety, training, and human rights. Disclosure requirement S1-9 mandates gender distribution at top management and age distribution data; S1-12 requires percentage of employees with disabilities. The November 2025 Omnibus amendments streamlined the datapoints; mandatory application is targeted for financial years beginning on or after 1 January 2027, with voluntary early use possible in 2026. Any DEI platform targeting European enterprise customers must produce ESRS S1-aligned reports.

**EU Pay Transparency Directive (Directive 2023/970/EU)**
- URL: https://ogletree.com/insights-resources/blog-posts/the-june-2026-eu-pay-transparency-directive-implementation-deadline-looms/
- Requires all EU member states to implement pay transparency obligations by 7 June 2026. Employers with 250+ employees report annually; 100–249 employees every three years. If a gender pay gap exceeds 5% and cannot be justified by objective criteria, employers must conduct a joint pay assessment. A DEI platform must generate regulatory-compliant gender pay gap reports and support joint pay assessment workflows.

**US EEOC EEO-1 Component 1 Report**
- URL: https://www.eeoc.gov/data/eeo-data-collections
- Mandatory annual data collection for US private-sector employers with 100+ employees and federal contractors with 50+ employees meeting specific criteria. Requires workforce demographic data classified by 10 job categories, 7 race/ethnicity categories, and sex. The EEOC expanded race/ethnicity categories in 2024. DEI platforms serving US employers must generate EEO-1-formatted exports and support the voluntary self-identification workflow preferred by the EEOC.

### ISO Standards

**ISO 30415:2021 — Human Resource Management: Diversity and Inclusion**
- URL: https://www.iso.org/standard/71164.html
- The primary international standard providing guidance on diversity and inclusion for organisations. Covers governance, HR lifecycle processes (workforce planning, recruitment, remuneration, induction, learning and development), supply chain relationships, and external stakeholder engagement. A DEI platform's data model and process framework should be consistent with the HR lifecycle stages and D&I governance concepts defined in ISO 30415.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The global standard for information security management. Directly relevant because DEI platforms process special category personal data (demographic, health, disability, sexual orientation data under GDPR Article 9). ISO 27001 certification demonstrates the security controls required for enterprise procurement and data processing agreements.

**ISO/IEC 42001:2023 — AI Management Systems**
- URL: https://www.iso.org/standard/81230.html
- Establishes requirements for responsible development and use of AI systems. Directly applicable to AI features in a DEI platform (recommendation engines, predictive attrition models, inclusive language detection), particularly as enterprise procurement increasingly requires AI governance evidence.

### Privacy & Data Protection Standards

**GDPR Article 9 — Processing of Special Categories of Personal Data**
- URL: https://gdpr-info.eu/art-9-gdpr/
- Racial or ethnic origin, health data, sexual orientation, disability status, and religious beliefs are "special category" data under GDPR. Processing is generally prohibited except under specific legal bases (explicit consent, employment law obligations, substantial public interest). DEI platforms collecting demographic data from EU employees must identify a valid Article 9 basis, conduct Data Protection Impact Assessments (DPIAs), implement pseudonymisation, and enforce strict access controls. This is a fundamental architectural constraint, not an optional compliance item.

**UK GDPR and ICO Guidance on Special Category Data**
- URL: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/lawful-basis/special-category-data/
- Post-Brexit UK equivalent of GDPR Article 9. Employers processing diversity data in the UK must have a lawful basis under both Article 6 and Article 9 (UK GDPR), with additional Schedule 1 condition under the Data Protection Act 2018. DEI platforms must support separate consent and data processing configurations for UK vs. EU deployments.

**NIST Privacy Framework (v1.1)**
- URL: https://secureprivacy.ai/blog/nist-privacy-framework
- US framework for identifying and mitigating privacy risks in systems and data practices. Relevant for US-market deployments, particularly for federal contractors subject to EEO-1 requirements. Provides structured guidance on data minimisation, consent management, and privacy-by-design architecture.

### AI Fairness & Bias Standards

**NIST AI Risk Management Framework (AI RMF 1.0) — NIST AI 100-1**
- URL: https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-1.pdf
- NIST's framework for managing AI risks throughout the AI lifecycle, including bias detection, fairness measurement, and explainability. The RMF's trustworthy AI characteristics (valid, reliable, safe, secure, explainable, fair with harmful bias managed, privacy-enhanced) should govern design of any AI recommendation engine, predictive model, or automated decision support feature in a DEI platform.

**EU AI Act — High-Risk AI Systems in Employment Contexts**
- URL: https://artificialintelligenceact.eu/
- AI systems used in employment, worker management, and access to self-employment contexts are classified as high-risk under Annex III of the EU AI Act. DEI platforms using AI for recruitment screening, performance evaluation, or promotion recommendations must comply with high-risk AI obligations: risk management systems, data governance, technical documentation, transparency, human oversight, and accuracy/robustness measures. Mandatory for EU-market deployments by August 2026.

### ESG & Sustainability Reporting Frameworks

**GRI 405: Diversity and Equal Opportunity 2016 (under revision)**
- URL: https://www.globalreporting.org/publications/documents/english/gri-405-diversity-and-equal-opportunity-2016/
- The widely used GRI standard for workforce diversity reporting, covering gender breakdown across governance bodies and employees, and gender pay ratio by employee category. GRI is actively revising GRI 405 with an updated Diversity and Inclusion standard (consultation closed September 2025; effective likely mid-2026) that will add governance and accountability disclosures and address direct and indirect discrimination. DEI platforms must track this revision and support updated GRI exports.

**Bloomberg Gender-Equality Index (GEI) Framework**
- URL: https://assets.bbhub.io/company/sites/46/2021/01/2021_Methodology_PDF_FNL-2.pdf
- Standardised voluntary reporting framework used by public companies to demonstrate gender equity commitment. Scores across five pillars: female leadership and talent pipeline, equal pay and gender pay parity, inclusive culture, anti-sexual harassment policies, and pro-women brand. Scored 0–100%. DEI platforms serving listed companies benefit from automated GEI data extraction and report generation.

### Identity & Access Management Standards

**SCIM 2.0 (System for Cross-domain Identity Management) — RFC 7642, 7643, 7644**
- URL: https://scim.cloud/
- Standard protocol for automated user provisioning and deprovisioning between identity providers and applications. DEI platforms should implement SCIM 2.0 to receive employee roster data (hire, transfer, termination events) automatically from HRIS systems and identity providers (Okta, Azure AD, Google Workspace). Core endpoints: `/Users`, `/Groups`.

**OAuth 2.0 (RFC 6749) and OpenID Connect**
- URL: https://oauth.net/2/ and https://openid.net/connect/
- OAuth 2.0 is the standard authorisation framework for API access; OpenID Connect (OIDC) adds identity layer for SSO. DEI platforms must implement OAuth 2.0 for API authorisation and OIDC for enterprise SSO with identity providers (Okta, Azure AD, Ping Identity). SCIM provisioning is typically secured with OAuth 2.0 bearer tokens.

**SAML 2.0**
- URL: https://www.oasis-open.org/standards/#samlv2.0
- XML-based federation standard still widely required for enterprise SSO in large organisations (particularly those not yet on OIDC). DEI platforms targeting enterprise buyers should support both SAML 2.0 and OIDC to cover the full range of enterprise identity configurations.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de facto standard for documenting REST APIs. A DEI platform's public API and HRIS integration layer should be documented via OAS 3.1 to enable developer self-service, SDK generation, and API gateway integration. Machine-readable OAS 3.1 documents enable automated testing, mocking, and client generation.

**JSON Schema (draft-07 and later)**
- URL: https://json-schema.org/
- Standard for describing the structure and validation of JSON data. Relevant for defining and validating DEI data payloads: demographic records, survey responses, pay equity analysis inputs/outputs, and ESG reporting data structures.

---

## Similar Products — Developer Documentation & APIs

### Visier People Analytics

- **Description:** Enterprise people analytics platform with 50+ out-of-the-box metrics covering diversity, retention, performance, and succession. Proprietary benchmarking database from 25,000+ organisations.
- **API Documentation:** https://docs.visier.com/developer/apis/apis.htm
- **Getting Started:** https://docs.visier.com/developer/apis/apis-get-started-home.htm
- **SDKs/Libraries:** Python SDK available; Postman collections provided
- **Developer Guide:** https://docs.visier.com/developer/Platform/overview.htm
- **Data Ingestion:** REST API over HTTPS; GraphQL API for selective data retrieval
- **Standards:** REST/JSON, GraphQL, OpenAPI documented
- **Authentication:** OAuth 2.0

### Culture Amp

- **Description:** Employee engagement, DEI survey, and performance platform with scientifically validated DEI survey template measuring 7 inclusion dimensions. Global benchmarking database.
- **API Documentation:** https://docs.api.cultureamp.com/docs/resources-getting-started
- **API Overview:** https://www.cultureamp.com/company/api
- **Reporting API:** https://support.cultureamp.com/en/articles/9232738-reporting-api
- **Standards:** REST/JSON; OpenAPI 3 specification published
- **Authentication:** OAuth 2.0 Client Credentials Flow
- **Notes:** API included in standard subscription at no additional cost. Reporting API requires raw data to be enabled before survey launch.

### Workday HCM

- **Description:** Full-suite HCM platform with embedded DEI module (VIBE™). HRIS-native demographic data; no separate data ingestion for Workday customers. Used by most large enterprises globally.
- **REST API Directory:** https://community.workday.com/sites/default/files/file-hosting/restapi/
- **Web Services Directory (SOAP):** https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html
- **Integration Guide:** https://www.getknit.dev/blog/workday-api-integration-in-depth
- **Standards:** REST/JSON (growing), SOAP/XML (legacy, still required for many operations)
- **Authentication:** OAuth 2.0 (REST); WS-Security (SOAP legacy)
- **Rate Limits:** 10 API calls per second
- **Notes:** Integration requires an Integration System User (ISU). Complex security model with Workday-specific configuration. Critical HR operations often still require SOAP API.

### SAP SuccessFactors

- **Description:** Enterprise HCM suite with DEI module covering diversity dashboards, pay equity analytics, and CSRD/ESRS S1-aligned workforce reporting via People Analytics Stories.
- **API Documentation:** https://api.sap.com/package/SFSFOData/odata
- **Merge Integration Guide (simplified):** https://www.merge.dev/integrations/sap-successfactors
- **Standards:** OData v2/v4; REST/JSON for newer APIs
- **Authentication:** OAuth 2.0 (SAML bearer assertion); Basic Auth (legacy)
- **Notes:** OData-based API; extensive schema. SAP Business Technology Platform (BTP) used for extensibility and custom integrations.

### Syndio (Pay Equity)

- **Description:** AI-native pay equity platform with real-time gap monitoring, in-workflow pay decision guidance (Slack/Teams integration), and global compliance reporting for EU Pay Transparency Directive, CSRD, and US state laws.
- **Website:** https://synd.io
- **API/Integration docs:** Not publicly documented (enterprise sales-gated)
- **Standards:** REST/JSON (inferred from HRIS integration pattern)
- **Authentication:** OAuth 2.0 (inferred)
- **Notes:** Syndio positions as a platform with expert advisory services, not a developer-self-service product. Integration is handled via professional services for enterprise deployments. Direct API documentation is not publicly accessible.

### Merge (Unified HRIS API)

- **Description:** Unified API layer enabling a single integration point for 70+ HRIS, ATS, and payroll systems (Workday, SAP SuccessFactors, BambooHR, ADP, Gusto, UKG Pro, etc.). Normalises disparate HR schemas into a single data model. Highly relevant for a new DEI platform needing broad HRIS connectivity without building individual integrations.
- **API Documentation:** https://docs.merge.dev/hris/
- **SDKs/Libraries:** JavaScript/TypeScript, Python, Ruby, Java, Go, C# SDKs
- **Standards:** REST/JSON; OpenAPI 3.0
- **Authentication:** API key (your server-to-Merge) + OAuth 2.0 (end-user authorisation of HRIS access)
- **Notes:** Merge charges per linked account (per customer HRIS connection). Using Merge reduces HRIS integration engineering burden significantly but introduces a third-party dependency and ongoing per-account cost.

### Finch (Unified HRIS & Payroll API)

- **Description:** Unified API connecting to 220+ HRIS and payroll systems through one standardised data model. Focused specifically on HR/payroll (vs. Merge's broader multi-category scope). Provides normalised HR schemas suited to compliance-heavy use cases.
- **API Documentation:** https://developer.tryfinch.com/
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0 (Finch Connect for end-user authorisation)
- **Notes:** Strong fit for DEI platforms needing reliable payroll and compensation data alongside demographic data for pay equity analysis. Covers ADP, Gusto, Paychex, and other payroll-primary systems that Workday/SuccessFactors-focused connectors miss.

### BambooHR

- **Description:** Popular HRIS for mid-market companies (typically 50–1,000 employees). REST API provides access to employee records, time-off, departments, custom fields, and custom reports. Key integration target for DEI platforms serving mid-market.
- **API Documentation:** https://documentation.bamboohr.com/reference
- **Integration Guide:** https://bindbee.dev/blog/bamboohr-api
- **Standards:** REST/JSON
- **Authentication:** API key (Basic Auth with API key as username)
- **Notes:** Well-documented and developer-friendly. Custom fields supported, enabling ingestion of diversity demographic data stored as custom BambooHR fields.

### ADP Workforce Now / ADP API

- **Description:** One of the largest global payroll and HR platforms. Comprehensive compensation and workforce data. Essential for DEI platforms serving mid-to-large US enterprises where ADP is the payroll system of record.
- **API Documentation:** https://developers.adp.com/
- **Merge Integration (simplified):** https://www.merge.dev/integrations/adp-workforce-now
- **Standards:** REST/JSON; some SOAP legacy endpoints
- **Authentication:** OAuth 2.0 (requires ADP partner access agreement)
- **Notes:** ADP integration requires a formal partnership agreement with ADP, which adds procurement lead time. Using Merge or Finch as a middleware layer is a common alternative to direct ADP API access.

---

## Notes

**Unified API layer recommendation:** A new DEI platform should strongly consider using Merge or Finch as the HRIS integration layer rather than building direct integrations with Workday, SAP SuccessFactors, ADP, and 20+ other systems. The engineering cost of maintaining direct HRIS integrations is substantial, and platforms like Merge provide 70+ connectors through a single normalised API at the cost of per-account pricing.

**EU AI Act compliance timeline:** AI features (recommendation engines, predictive attrition models, automated pay equity alerts) in a DEI platform likely qualify as high-risk AI systems under EU AI Act Annex III (employment and HR use cases). Compliance obligations are mandatory from August 2026 for EU deployments. Architecture and documentation should be designed with EU AI Act high-risk requirements in mind from the start.

**GDPR Article 9 as architectural constraint:** The requirement to process demographic (race, ethnicity, disability, sexual orientation) data under a specific Article 9 legal basis — with DPIAs, pseudonymisation, and restricted access controls — should be designed into the data model and access control architecture from day one, not retrofitted. This is particularly important for international deployments where different EU member states have different Article 9 implementations.

**GRI 405 revision in progress:** The GRI is actively revising GRI 405 (Diversity and Equal Opportunity) with a new Labour Standards standard. Organisations using GRI 405 for ESG reporting should monitor GRI's publication timeline (effective expected mid-2026) and plan for schema updates in their ESG report generation module.

**Pay Transparency Directive implementation deadline:** 7 June 2026 is the EU-wide implementation deadline. DEI platforms with European customers should ensure gender pay gap reporting functionality is in place and compliant with member-state implementations by this date.
