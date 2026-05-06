# Standards & API Reference

> Project: Personal Trainer Management · Generated: 2026-05-04

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- Personal trainer platforms store sensitive health, payment, and biometric data. ISO 27001 certification provides a governance framework for protecting that data and is increasingly required by enterprise gym chains procuring SaaS tools.

**ISO/IEC 29101 — Privacy Architecture Framework**
- URL: https://www.iso.org/standard/45124.html
- Defines a privacy reference architecture for systems handling personal data. Directly relevant when building a platform that ingests health history, body composition, and biometric streams from wearables.

**ISO 9241-210 — Human-Centred Design for Interactive Systems**
- URL: https://www.iso.org/standard/77520.html
- Governs the ergonomics of human-system interaction. Relevant to the design of trainer dashboards and client mobile apps, particularly for accessible onboarding and progressive disclosure of complex programme data.

---

### W3C & IETF Standards

**RFC 9110 — HTTP Semantics**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The foundational standard governing HTTP request/response semantics used by all REST APIs in the personal trainer software ecosystem, including Mindbody and wearable vendor APIs.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- OAuth 2.0 is the mandatory authentication standard for all major wearable APIs (Fitbit, Garmin, Withings, Strava). Any platform ingesting wearable data must implement OAuth 2.0 authorisation flows.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://www.rfc-editor.org/rfc/rfc6750
- Governs how access tokens are transmitted in HTTP requests. All wearable and fitness platform APIs surveyed use Bearer token authentication.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- JWT is the dominant token format for API authentication across fitness SaaS platforms. Relevant for inter-service authentication in a multi-trainer business architecture.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the `Link` header used for pagination in REST APIs. Required when implementing pagination for large client rosters or workout history queries.

**W3C WCAG 2.2 — Web Content Accessibility Guidelines**
- URL: https://www.w3.org/TR/WCAG22/
- Accessibility requirements for client-facing web portals and trainer dashboards. Relevant for fitness platforms serving clients with physical limitations.

---

### Data Model & API Specifications

**HL7 FHIR R4 — Fast Healthcare Interoperability Resources**
- URL: https://www.hl7.org/fhir/R4/
- FHIR defines RESTful resource types (Patient, Observation, Goal, ServiceRequest) relevant when a personal trainer platform operates in a medically-adjacent context (physiotherapy referrals, cardiac rehab, clinical exercise prescription). Increasingly used as a data exchange format for health apps.

**HL7 FHIR Physical Activity Implementation Guide v1.0.1**
- URL: https://build.fhir.org/ig/HL7/physical-activity/
- US-specific implementation guide built on FHIR 4.0.1 (`hl7.fhir.us.physical-activity#1.0.1`). Defines interoperability expectations for systems measuring, reporting, and improving patient physical activity levels. Identifies three cooperating system types: Care Managers (EHRs), Service Providers (personal trainers, community fitness centers), and Patients/Consumers. The primary exchange measure is the Exercise Vital Sign (LOINC 89574-8). REST is the preferred interoperability mechanism.

**Open mHealth Schema (IEEE P1752 Standard for Mobile Health Data)**
- URL: https://www.openmhealth.org/
- GitHub: https://github.com/openmhealth
- An open JSON schema library for standardising mobile health data points including heart rate, physical activity, respiratory rate, sleep, and stress. The Shimmer library maps third-party wearable API responses (Fitbit, Runkeeper) into Open mHealth-compliant JSON. Serves as a normalisation layer between heterogeneous wearable APIs and a unified data model.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The standard format for documenting REST APIs. Mindbody publishes its API in OpenAPI-compatible format. Any personal trainer management platform should document its public API using OpenAPI 3.1 to enable ecosystem integrations.

**Garmin FIT Protocol**
- URL: https://developer.garmin.com/fit/protocol/
- The binary file format used by Garmin devices for activity export (.FIT files). Contains timestamped records for GPS, heart rate, power, cadence, and more. The Activity API on Garmin Connect provides .FIT, .GPX, and .TCX activity file access.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) — See W3C & IETF above**

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0. Enables single sign-on for trainers managing multiple businesses or operating within a white-label app. Standard for user authentication in modern SaaS platforms.

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/standards/
- Mandatory for any personal trainer software processing card payments. Defines security requirements for environments storing, processing, or transmitting payment account data. Applies to session fee collection, recurring subscriptions, and package purchases.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr.eu/
- Applies when serving EU-based trainers or clients. Health data (body composition, injury history, medical intake forms) is a special category of personal data under Article 9, requiring explicit consent and stricter processing controls. Right to erasure and data portability are directly relevant for client offboarding.

**HIPAA — Health Insurance Portability and Accountability Act (US)**
- URL: https://www.hhs.gov/hipaa/
- Applies when a personal trainer platform is used in medically-adjacent settings (clinical exercise prescription, cardiac rehab, physiotherapy referrals). Health history and injury intake forms may constitute Protected Health Information (PHI) when trainers operate under physician referral.

**OWASP Top 10 (2021)**
- URL: https://owasp.org/www-project-top-ten/
- The foundational security checklist for web application development. Injection, broken access control, and insecure design are the highest-risk categories for a multi-tenant SaaS platform where trainer data must be isolated from other trainers' clients.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is relevant when building AI assistant features inside a personal trainer management platform. An MCP server exposing client data, programme history, and wearable readings would allow an LLM-powered coaching co-pilot to query context (e.g. "show me John's last 4 weeks of compliance and HRV trend") without requiring manual data retrieval.

**Model Context Protocol Specification**
- URL: https://modelcontextprotocol.io/specification
- Defines the JSON-RPC 2.0-based protocol for tool definitions, resource exposure, and sampling requests. A personal trainer platform MCP server could expose resources for: client profiles, programme history, session logs, wearable metrics, compliance rates, and invoicing data.

---

## Similar Products — Developer Documentation & APIs

### Mindbody (MINDBODY Online)

- **Description:** Enterprise fitness studio management platform serving scheduling, client management, payments, and multi-location operations. The most mature developer API in the personal fitness space.
- **API Documentation:** https://developers.mindbodyonline.com/
- **Webhooks API:** https://developers.mindbodyonline.com/WebhooksDocumentation
- **Affiliate API:** https://developers.mindbodyonline.com/AffiliateDocumentation
- **Developer Guide:** https://www.mindbodyonline.com/business/developer-tools
- **Standards:** REST/JSON, standard HTTP verbs (GET, POST, DELETE), API sandbox available for testing
- **Authentication:** API key + site credentials; OAuth for consumer-facing Affiliate API flows

---

### ABC Trainerize

- **Description:** Market-leading personal trainer delivery platform with AI Workout Builder, wearable integrations, and an open API for data extraction.
- **API / Integration Documentation:** https://help.trainerize.com/hc/en-us/categories/115000017566-Add-ons-and-Integrations
- **Webhooks:** Available for real-time event notifications
- **Developer Guide:** Accessible via Trainerize Help Center
- **Standards:** REST/JSON; Zapier connectors available
- **Authentication:** API key; OAuth for third-party wearable integrations

---

### Apple HealthKit

- **Description:** Apple's centralised health data framework for iOS and watchOS. Stores heart rate, steps, sleep, body composition, workout records, and more. All data is local to the device; an iOS app must request permission and relay data to a server.
- **Developer Documentation:** https://developer.apple.com/documentation/healthkit
- **SDK:** iOS HealthKit framework (Swift/Objective-C); available from iOS 8.0+, Mac Catalyst 13.0+, watchOS 2.0+
- **Getting Started:** https://developer.apple.com/documentation/healthkit/setting_up_healthkit
- **Standards:** Local-only data store; no backend REST API — requires native iOS integration
- **Authentication:** iOS permission request model (read/write entitlements per data type)

---

### Google Health Connect (formerly Google Fit)

- **Description:** Android's centralised health and fitness data repository replacing the deprecated Google Fit REST API (retired June 2025). Provides a local SDK for reading/writing health data across Android apps.
- **Developer Documentation:** https://developer.android.com/health-and-fitness/guides/health-connect
- **SDK:** Android Health Connect SDK (Kotlin/Java)
- **Migration Guide (from Google Fit):** https://developer.android.com/health-and-fitness/guides/health-connect/migrate/from-google-fit
- **Standards:** Local on-device data store; REST API removed — requires native Android integration
- **Authentication:** Android permission request model per data type

---

### Fitbit Web API

- **Description:** Fitbit's REST API providing access to activity, heart rate, sleep, body weight, nutrition, and device data from Fitbit and Google Pixel Watch devices.
- **API Documentation:** https://dev.fitbit.com/build/reference/web-api/
- **Developer Guide:** https://dev.fitbit.com/
- **SDKs:** No official SDK; community libraries available in Python, JavaScript, Go
- **Standards:** REST/JSON; OAuth 2.0 with PKCE for personal apps; webhook subscriptions for push notifications
- **Authentication:** OAuth 2.0 (Authorization Code with PKCE)

---

### Garmin Connect Developer Program

- **Description:** Garmin's API suite for accessing user activity, health, and wellness data from Garmin wearables. Covers activities, daily summaries, sleep, body composition, heart rate, and training metrics.
- **API Documentation:** https://developer.garmin.com/gc-developer-program/
- **Health API:** https://developer.garmin.com/gc-developer-program/health-api/
- **Activity API:** https://developer.garmin.com/gc-developer-program/activity-api/ (FIT, GPX, TCX file access)
- **Training API:** https://developer.garmin.com/gc-developer-program/training-api/
- **Standards:** REST/JSON for Health API; binary FIT protocol for activity files
- **Authentication:** OAuth 2.0; partner application required (approval process)

---

### Terra API

- **Description:** Unified wearable and health data aggregation API connecting 500+ providers (Garmin, Fitbit, Apple Health, Oura, WHOOP, Polar, Samsung Health, Withings, Strava, MyFitnessPal, CGM devices) through a single integration. Normalises heterogeneous data formats into a consistent schema. Event-based: data pushed via webhook after user authentication.
- **API Documentation:** https://docs.tryterra.co/
- **Getting Started:** https://docs.tryterra.co/health-and-fitness-api/getting-started
- **Core Concepts:** https://docs.tryterra.co/reference/health-and-fitness-api/core-concepts
- **Standards:** REST/JSON; webhooks for real-time push; GDPR, HIPAA, and SOC 2 compliant
- **Authentication:** API key + per-user OAuth flows (Terra handles provider-specific OAuth)

---

### Stripe

- **Description:** Payment processing API used by Trainerize, My PT Hub, TrueCoach, FitBudd, CoachAccountable, and most other platforms for session billing, package sales, and recurring subscriptions.
- **API Documentation:** https://stripe.com/docs/api
- **SDKs:** JavaScript, Python, Ruby, Go, Java, PHP, .NET (official)
- **Developer Guide:** https://stripe.com/docs
- **Standards:** REST/JSON; webhooks for payment events; PCI DSS compliant
- **Authentication:** API key (publishable + secret); webhook signature verification

---

### Healthie

- **Description:** HIPAA-compliant telehealth and health coaching platform with an ONC-certified FHIR API. Relevant for personal trainers operating in clinical or medically-supervised contexts requiring health record interoperability.
- **API Documentation:** https://help.gethealthie.com/article/1013-hl7-fhir-standards
- **Developer Guide:** https://developer.gethealthie.com/
- **Standards:** HL7 FHIR R4; REST/JSON; GraphQL API also available
- **Authentication:** OAuth 2.0; SMART on FHIR for EHR-integrated contexts

---

## Notes

**Google Fit API retirement**: The Google Fit REST API was retired on 30 June 2025. New integrations must use the Android Health Connect SDK for on-device data access. Cross-platform health data for Android users is now managed through Health Connect rather than a cloud REST endpoint.

**Terra API as preferred aggregation layer**: For teams that need multi-wearable support without building individual OAuth flows for Garmin, Fitbit, Oura, and WHOOP, Terra API is the lowest-friction path. It normalises data formats and handles GDPR/HIPAA compliance, making it particularly well-suited to an AI-native platform where wearable data feeds adaptive programming algorithms.

**HL7 FHIR Physical Activity IG adoption**: The Physical Activity Implementation Guide is US-centric and most relevant if the platform plans to receive referrals from healthcare providers or participate in value-based care programmes. For general personal training, the Exercise Vital Sign (LOINC 89574-8) offers a lightweight standardised measure of activity level that could be included in client records without full FHIR implementation.

**OpenAPI 3.1 for developer ecosystem**: Publishing a public OpenAPI 3.1 specification from launch will enable Zapier, Make (Integromat), and n8n connectors to be built by the community, accelerating ecosystem growth without requiring a dedicated integrations team.
