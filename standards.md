# Standards & API Reference

> Project: Biometric Attendance System · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 19794 — Biometric Data Interchange Formats**
- URL: https://www.iso.org/standard/50862.html (Part 1 Framework); https://www.iso.org/standard/50864.html (Part 2 Fingerprint); https://en.wikipedia.org/wiki/ISO/IEC_19794-5 (Part 5 Face)
- Multi-part series defining standard data formats for exchanging biometric data between sensing, storage, and matching systems. Part 2 covers fingerprint minutiae; Part 4 covers finger image data; Part 5 covers face image data; Part 6 covers iris image data. Directly relevant to ensuring biometric template interoperability across device vendors.

**ISO/IEC 19785 — Common Biometric Exchange Formats Framework (CBEFF)**
- URL: https://www.iso.org/standard/77892.html (Part 1); https://www.iso.org/standard/72192.html (Part 3); https://www.iso.org/standard/89072.html (Part 4, 2025)
- Defines a container structure (Biometric Information Record, BIR) that wraps any biometric data with a standard header (SBH), biometric data block (BDB), and optional security block (SB). CBEFF enables any CBEFF-compliant system to interpret biometric data from another compliant system without vendor-specific parsers.

**ISO/IEC 30107 — Biometric Presentation Attack Detection (PAD)**
- URL: https://www.iso.org/standard/53227.html (Part 1); https://www.iso.org/standard/79520.html (Part 3, 2023)
- Defines a framework and testing methodology for evaluating liveness detection (anti-spoofing) mechanisms. Part 3 specifies test levels (Level 1, 2, 3) with error rate thresholds. iBeta is the primary accredited lab for PAD testing. Any commercial biometric attendance terminal should carry ISO 30107-3 Level 2 certification to defend against printed photo, video replay, and silicone finger attacks.

**ISO/IEC 24745 — Biometric Information Protection**
- URL: https://www.iso.org/standard/75302.html (2022 edition)
- Specifies requirements for protecting biometric templates during storage and transmission. The three core requirements are: *irreversibility* (templates cannot be reverse-engineered to raw biometric), *unlinkability* (templates from different systems cannot be cross-correlated to the same person), and *revocability/renewability* (a compromised template can be cancelled and a new one issued without new biometric capture). This standard underpins the "cancellable biometrics" architecture that is best practice for any cloud-connected attendance system.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001 (redirect to current edition)
- The foundational information security management standard. Biometric data is personal and sensitive; ISO 27001 certification demonstrates that organisational controls for access, breach response, vendor management, and data security are in place. Covers approximately 75–80% of GDPR compliance requirements.

**ISO/IEC 27701:2025 — Privacy Information Management (PIMS)**
- URL: https://www.iso.org/standard/71670.html
- An extension to ISO 27001 that adds privacy-specific controls. The 2025 revision includes a GDPR mapping annex, allowing ISO 27701 certification to serve as evidence of GDPR compliance. Relevant for any biometric attendance system operator processing EU personal data.

---

### W3C & IETF Standards

**W3C Web Authentication (WebAuthn) Level 2**
- URL: https://www.w3.org/TR/webauthn-2/
- The browser-side API that enables passwordless and biometric authentication. Although primarily designed for website login rather than physical attendance terminals, WebAuthn is the standard underpinning FIDO2/passkey integration for web-based time-clock portals and employee self-service. Any modern web-based attendance interface that offers biometric login should implement WebAuthn.

**FIDO2 / CTAP2 — Client to Authenticator Protocol**
- URL: https://fidoalliance.org/specifications/
- The FIDO Alliance specification for communication between a platform (browser/OS) and an external authenticator. Together with WebAuthn, FIDO2 defines the full passkey authentication stack. Relevant for the employee self-service portal and mobile clock-in app where device biometrics (Face ID, fingerprint) are used for authentication.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The standard for delegated authorization; defines four grant types (authorization code, implicit, resource owner password, client credentials). All REST API integrations between the attendance system and downstream HRMS/payroll systems should use OAuth 2.0 for secure token-based access, removing the need to store raw credentials.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines the JWT format used by OAuth 2.0 / OpenID Connect for bearer tokens. Attendance system API endpoints should validate JWT signatures and expiry on every request.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0, adding a standard `id_token` (JWT) carrying user identity claims. Enables SSO integration with enterprise identity providers (Azure AD, Okta, Google Workspace) so employees and HR admins authenticate via their existing corporate account rather than a separate attendance-system password.

**RFC 7617 — HTTP Basic Authentication / RFC 6750 — Bearer Token Usage**
- URLs: https://datatracker.ietf.org/doc/html/rfc7617 ; https://datatracker.ietf.org/doc/html/rfc6750
- RFC 6750 defines how to include OAuth 2.0 bearer tokens in HTTP requests. All REST API calls to and from the attendance system should use bearer tokens rather than API-key headers where third-party access is involved.

**RFC 5246 / RFC 8446 — TLS 1.2 / TLS 1.3**
- URL: https://datatracker.ietf.org/doc/html/rfc8446
- All biometric data in transit between terminal devices, the backend, and integrated systems must be encrypted via TLS 1.2 minimum (TLS 1.3 preferred). This applies to attendance event push from hardware terminals, REST API calls, and the employee self-service portal.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.1**
- URL: https://spec.openapis.org/oas/v3.1.1.html
- The industry standard for describing REST APIs. The attendance system's public API (for HRMS integration, reporting, and hardware connectors) should be described in OpenAPI 3.1 format and published via Swagger UI or Redoc. JSON Schema Draft 2020-12 is now a superset of the OpenAPI schema object.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification
- Used to formally define and validate the structure of JSON request/response bodies in OpenAPI definitions. Relevant for defining clock event payloads, employee enrolment records, shift schedule objects, and payroll export formats.

**iCalendar (RFC 5545) / iCal Format**
- URL: https://datatracker.ietf.org/doc/html/rfc5545
- The standard data format for calendar and scheduling data. Shift schedules and leave calendars exported from the attendance system should support iCal format to enable import into Google Calendar, Outlook, and Apple Calendar for employee visibility.

**CSV / ISO 8601 Datetime**
- CSV remains the lowest-common-denominator export format for payroll; timestamps should follow ISO 8601 (YYYY-MM-DDTHH:MM:SSZ) throughout all exports and API payloads to avoid timezone ambiguity.

---

### Security & Authentication Standards

**NIST SP 800-76-2 — Biometric Specifications for Personal Identity Verification**
- URL: https://csrc.nist.gov/pubs/sp/800/76/2/final
- Companion to FIPS 201; specifies acquisition and formatting requirements for fingerprint, iris, and facial data used in US federal PIV credentials. While targeted at government identity programmes, the technical specifications (image resolution, compression, template format) represent best practice for any high-assurance biometric system.

**NIST SP 800-63-4 — Digital Identity Guidelines (2025)**
- URL: https://pages.nist.gov/800-63-4/
- The 2025 update to NIST's identity assurance framework. Defines Identity Assurance Levels (IAL) and Authentication Assurance Levels (AAL). A biometric attendance system operating at AAL2 or AAL3 must incorporate phishing-resistant multi-factor authentication (passkeys/FIDO2) and liveness-verified biometrics.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- The canonical guide to API security risks. Biometric system REST APIs are high-value targets; the system must address: Broken Object Level Authorisation (BOLA), Broken Authentication, Excessive Data Exposure, and Security Misconfiguration. Biometric template endpoints must never return raw template data in API responses.

**Illinois Biometric Information Privacy Act (BIPA) — 740 ILCS 14**
- URL: https://www.ilga.gov/legislation/ilcs/ilcs3.asp?ActID=3004
- The strictest US biometric privacy statute. Requires written notice before collection, a written data retention/destruction policy, and written consent. Prohibits sale or profit from biometric data. Imposes per-violation statutory damages ($1,000–$5,000). Any system deployed to Illinois employers must include BIPA-compliant consent workflows and data destruction scheduling. The 2024 amendment clarifies electronic consent is valid and limits per-occurrence liability for continuous collection.

**EU GDPR Article 9 — Special Categories of Personal Data**
- URL: https://gdpr-info.eu/art-9-gdpr/
- Biometric data processed for the purpose of uniquely identifying a natural person is a special category under Article 9, requiring explicit consent (Article 9(2)(a)) or another listed legal basis. The system must implement: explicit consent capture, purpose limitation, data minimisation, automated deletion on request (Article 17), and breach notification within 72 hours (Article 33).

**California CPRA / CCPA — Biometric Data as Sensitive Personal Information**
- URL: https://oag.ca.gov/privacy/ccpa
- California Privacy Rights Act classifies biometric data as sensitive personal information, triggering opt-out rights and restricted processing. Compliant systems must surface a "Limit the Use of My Sensitive Personal Information" option to California employees.

---

### MCP Server Specifications

The Model Context Protocol is potentially relevant if the system exposes AI-powered natural-language querying of attendance data, anomaly detection explanations, or HR chat assistants that need to access attendance records as context.

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/specification
- An open standard for connecting AI models to external data sources and tools. An MCP server wrapping the attendance system's read-only reporting API would allow LLM-based HR assistants (e.g., "show me departments with rising absenteeism") to query live attendance data securely without exposing raw credentials to the model. Relevant for the AI-native feature roadmap.

---

## Similar Products — Developer Documentation & APIs

### ZKTeco ZKBioTime API

- **Description:** The most widely deployed biometric attendance hardware ecosystem globally. ZKBioTime is the cloud/on-premises attendance management platform; its REST API is used by thousands of third-party integrations.
- **API Documentation:** https://www.zkteco.me/download-file/2051 (BioTime 8.5 API User Manual)
- **SDKs/Libraries:** Node.js: https://coding-libs.github.io/zkteco-js/ ; Python (Flask wrapper): https://github.com/zeidanbm/zkteco-restful-api ; .NET: https://www.nuget.org/packages/ZkTeco.Attendance.API
- **Developer Guide:** https://ampletrails.com/zkteco-api-integration-guide/ ; https://zktecoapi.com/
- **Standards:** REST/JSON; JWT bearer tokens; TLS 1.2+
- **Authentication:** JWT token obtained via `/api/jwt-api-token-auth/`

---

### Suprema BioStar X API

- **Description:** Enterprise biometric access control and time-and-attendance platform with the highest biometric accuracy in the market. BioStar X (successor to BioStar 2) provides a cloud-based REST API to manage devices, users, and events.
- **API Documentation:** https://www.supremainc.com/en/support/development-tools_biostar-x-api.asp
- **SDKs/Libraries:** BioStar 2 Device SDK (C, C#, Java): https://github.com/supremainc/BioStar2_device_SDK ; Frappe/ERPNext integration: https://github.com/navariltd/navari-frappehr-biostar
- **Developer Guide:** https://www.kimaldi.com/en/product/biostar_2_api/
- **Standards:** REST/JSON; Swagger/OpenAPI documented endpoints
- **Authentication:** API key + session token; OAuth 2.0 in BioStar X

---

### Jibble REST API

- **Description:** Freemium SaaS time-and-attendance with facial recognition, GPS geofencing, and kiosk mode. Widely used for mobile-first deployments without hardware terminals.
- **API Documentation:** https://developer.jibble.io (REST API reference)
- **SDKs/Libraries:** Not published; REST/JSON only
- **Developer Guide:** https://www.jibble.io/time-and-attendance-software
- **Standards:** REST/JSON; OpenAPI documented
- **Authentication:** OAuth 2.0 bearer tokens

---

### Truein API

- **Description:** AI face recognition attendance SaaS purpose-built for contract and multi-site workforces. Native integrations with major Indian HRMS platforms.
- **API Documentation:** https://truein.com (contact for developer access)
- **SDKs/Libraries:** Not published
- **Developer Guide:** https://truein.com/blogs/biometric-attendance-software
- **Standards:** REST/JSON; webhook event delivery
- **Authentication:** API key + OAuth 2.0

---

### ADP Workforce Now API

- **Description:** The leading US payroll and HCM platform. The Workforce Now API is the target integration endpoint for pushing verified attendance and timesheet data to trigger automated payroll processing.
- **API Documentation:** https://developers.adp.com/ ; https://developers.adp.com/articles/guides/adp-workforce-now-api-catalog
- **SDKs/Libraries:** JavaScript, Python, Java SDKs available on the ADP developer portal
- **Developer Guide:** https://www.getknit.dev/blog/adp-api-integration-in-depth
- **Standards:** REST/JSON; OpenAPI documented
- **Authentication:** OAuth 2.0 (requires purchase of API Central add-on)

---

### Workday HCM REST API

- **Description:** Enterprise HCM platform used by large organisations globally. The Time Tracking module accepts attendance data; the REST API covers HR, payroll, recruiting, and time tracking.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/restapi/ ; SOAP directory: https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html
- **SDKs/Libraries:** Not officially published; community-maintained Python and JavaScript wrappers
- **Developer Guide:** https://www.getknit.dev/blog/workday-api-integration-in-depth ; https://cloudfoundation.com/blog/workday-api-integration-tutorial/
- **Standards:** REST/JSON and SOAP/XML (legacy); OpenAPI partially documented
- **Authentication:** OAuth 2.0; rate limit: 10 API calls/second

---

### Xero Payroll API

- **Description:** Cloud accounting and payroll platform widely used by SMBs. The Payroll API supports importing timesheets, managing employees, and triggering pay runs — making it a natural output target for attendance data.
- **API Documentation:** https://developer.xero.com/documentation/api/payrollau/overview (AU); https://developer.xero.com/documentation/api/payrollnz/timesheets (NZ); https://developer.xero.com/documentation/api/payrolluk/overview (UK)
- **SDKs/Libraries:** Official SDKs for .NET, PHP, Java, Node.js, Python, Ruby: https://developer.xero.com/documentation/sdks-and-tools/sdk-overview/
- **Developer Guide:** https://developer.xero.com/documentation/guides/how-to-guides/how-to-integrate-my-payroll-system-with-xero/
- **Standards:** REST/JSON; OpenAPI 3.0 specification published
- **Authentication:** OAuth 2.0 (PKCE flow)

---

### Keka HR API

- **Description:** Leading cloud HRMS in India supporting 3,000+ biometric device models via four integration protocols. Relevant as both a competing product and a potential integration target for Indian market deployments.
- **API Documentation:** https://help.keka.com/admin/a-simple-guide-to-supported-integration-methods
- **SDKs/Libraries:** Not published externally; device integration via SQL, SDK, push, or REST
- **Developer Guide:** https://help.keka.com/admin/getting-started-with-biometric-device-integration
- **Standards:** REST/JSON; SQL direct integration; SDK (device vendor protocols)
- **Authentication:** API key + OAuth 2.0

---

### Frappe / ERPNext Biometric Sync Tool

- **Description:** Open-source middleware that pulls attendance logs from ZKTeco and other devices and pushes them into ERPNext via the Frappe REST API. A reference implementation for building open-source device adapters.
- **API Documentation:** https://github.com/frappe/biometric-attendance-sync-tool
- **SDKs/Libraries:** Python; uses `zklibs` for device communication
- **Developer Guide:** README in the GitHub repo above
- **Standards:** REST/JSON (Frappe API); ZKTeco proprietary device protocol
- **Authentication:** Frappe API key + API secret header

---

## Notes

**Evolving / Emerging Standards**

- **ISO/IEC 39794** (successor to 19794): The next generation biometric data interchange format series, with a more flexible, extensible encoding (ASN.1 and JSON representations). Adoption by hardware vendors is still partial as of 2026; monitoring this series is recommended for long-term interoperability.
- **NIST FRVT (Face Recognition Vendor Testing)**: Not a standard but the de-facto benchmark for facial recognition accuracy. Vendors that score well in NIST FRVT evaluations (https://pages.nist.gov/frvt/) have measurably better performance. Evaluating hardware terminals by their FRVT ranking is recommended.
- **CBEFF/ANSI INCITS 398**: The US national standard implementing CBEFF; some legacy US government integrations require INCITS rather than ISO formatting. Low relevance for commercial deployments outside the US federal sector.
- **Biometric Open Protocol Standard (BOPS) — IEEE 2410**: Defines cloud-based biometric authentication including template protection, liveness, and privacy. Still relatively niche but aligns well with the privacy-first architecture described in research.md.
- **MCP server ecosystem**: The Model Context Protocol is maturing rapidly. Building a read-only MCP server exposing attendance summaries, anomaly alerts, and shift data would enable integration with any MCP-compatible AI assistant without building bespoke chat interfaces.
