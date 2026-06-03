# BaRS Onboarding Process – How It Currently Works

## Overview

The Booking and Referral Standard (BaRS) onboarding process is a structured, step-by-step journey that takes a supplier from initial analysis through to live deployment. It is guided by the [BaRS Implementation Guide](https://simplifier.net/guide/nhsbookingandreferralstandard/Home?version=1.11.0) and involves technical development, formal assurance, and operational readiness activities.

The process can be broadly divided into seven stages:

1. **Analysis** – Understanding what is required
2. **BaRS Core** – Familiarising with the standard's core elements
3. **Applications** – Selecting the appropriate use case
4. **FHIR Assets** – Working with the FHIR implementation details
5. **Testing & Environments** – Connecting to and testing in BaRS environments
6. **Assurance (SCAL)** – Completing the formal assurance process
7. **Deployment** – Going live in production

---

## Stage 1: Analysis

The Analysis section provides foundational technical information to support suppliers in implementing BaRS effectively. It covers:

- **Principles and Prerequisites** – Core assumptions and requirements for a BaRS implementation.
- **Infrastructure** – The technical infrastructure underpinning BaRS (internet-first, no requirement for HSCN connectivity).
- **Roles and Responsibilities** – Who does what in the BaRS ecosystem (Senders and Receivers).
- **Releases and Versioning** – How BaRS versions are managed and released.
- **Content Negotiation** – How BaRS handles content negotiation between systems.

---

## Stage 2: BaRS Core

Suppliers must familiarise themselves with BaRS Core, which covers:

- End-to-end workflow
- Security and authorisation (application-restricted RESTful APIs using signed JWT authentication)
- Error handling and failure scenarios

---

## Stage 3: BaRS Applications

Suppliers select the use case and Application that meets their needs. Current applications include:

- **Application 1 & 2** – 111 to ED (Booking and Referral)
- **Application 3** – Referral only workflows
- **Application 4** – Validation workflows
- **Application 5** – Primary Care to Community Pharmacy (Minor Illness)
- **Application 6** – Out-of-Area CAD-CAD Referral
- **Application 7** – Additional Booking and Referral workflows

In most applications, a solution acts as both a Sender and a Receiver.

---

## Stage 4: FHIR Assets

Supporting information about the use of FHIR R4 to implement the standard, including access to:

- FHIR examples
- Bundles
- CodeSystems
- MessageDefinitions

---

## Stage 5: Testing & Environments

### Environments

BaRS provides three environments:

| Environment | Purpose | URL |
|---|---|---|
| **Sandbox** | Demonstrates key functionality without security overhead | `sandbox.api.service.nhs.uk` |
| **Integration (INT)** | Live-like environment for development and testing | `int.api.service.nhs.uk` |
| **Production (PROD)** | The live environment (requires completed assurance) | `api.service.nhs.uk` |

### Connecting as a Sender

To connect to the BaRS proxy as a Sender:

1. Follow the NHS Developer authentication and authorisation process (application-restricted, signed JWT).
2. Trust the relevant Certificate Authorities.
3. Provide the BaRS Team (via email to `england.bookingandreferralstandard@nhs.net`) with:
   - App Name, App ID (PROD), App ID (INT), App Description
   - Organisation name and ODS code
   - Product Name (as stated in the SCAL or TCC)
   - JWKS resource URL or KID/public key

### Connecting as a Receiver

BaRS uses TLS-MA (Transport Layer Security – Mutual Authentication) to communicate with Receiving endpoints. Receivers must:

1. Apply for an `nhs.uk` domain.
2. Request a certificate under the NHS Root CA (different chains for INT and PROD).
3. Create a Certificate Signing Request (CSR) – the FQDN must match the domain and be in the `.nhs.uk` domain.
4. For **INT**: Contact ITOC via the "Combined endpoint and service registration request" form.
5. For **PROD**: Send the CSR to `dir@nhs.net` (only after the Technical Conformance Certificate is issued).
6. Email `england.bookingandreferralstandard@nhs.net` with the Receiver URL to be added to the Endpoint Catalogue.

### Toolkit Workbench (TKW)

TKW is a testing tool embedded in the INT environment that:

- Supports testing of key workflows (booking, referral, validation).
- Inspects low-level technical requirements.
- Produces downloadable Validation Reports.
- Supports consistent error states for development and assurance.
- Uses "sentinel" values (specific NHS Numbers and NHSD-Target-Identifiers) to trigger particular behaviours.

**Important**: TKW is stateless by default (it does not check relationships between requests), except for a limited set of stateful scenarios for 111-to-ED workflows.

Suppliers must register a portal account at [https://maitportal.testlab.nhs.uk/](https://maitportal.testlab.nhs.uk/) to use TKW.

---

## Stage 6: Assurance (SCAL)

### Purpose

Assurance ensures that solutions:

- Conform to the requirements and specifications of the Standard.
- Handle higher-risk and complex workflow scenarios safely.
- Are ready for Production deployment.

### The SCAL Process

The **Supplier Conformance Assessment List (SCAL)** is the primary assurance mechanism. It is a lightweight, self-assessment document supported by evidence (log extracts, screenshots, TKW validation reports, etc.).

#### Starting the Process

1. Email the BaRS Team at `bookingandreferralstandard@nhs.net` with:
   - Organisation details
   - Product details
   - Which BaRS Application is being undertaken
2. Receive a link to create an account with the [Digital Onboarding Service](https://onboarding.prod.api.platform.nhs.uk/).

#### Completing the SCAL

The SCAL is completed online and is broken into four key sections. Section 3 (Technical Conformance Requirements) is the most significant and mirrors BaRS product structure:

- **Core** – Every supplier must complete this.
- **Role-specific sections** – e.g., Booking Sender, Referral Sender (111 to ED), etc.

Each section contains assessment items. Answering questions prompts for evidence (e.g., `.zip` files, log extracts, images). Progress can be saved at any point.

#### Section Statuses

| Status | Meaning |
|---|---|
| In Progress | Section is being worked on |
| Ready to Submit | All items are complete; ready for review |
| Awaiting Review | Submitted; with Solution Assurance team |
| In Review | Being reviewed by Solution Assurance |
| Information Required | Additional information needed from supplier |
| Review Completed | Review finished |
| Approved | Section has been formally approved |

#### Timeline

The assurance process typically takes **2–4 weeks** from first complete submission, allowing time for resubmissions after review.

#### End-to-End Demonstration

An end-to-end real-time demonstration of the solution is required as part of assurance, witnessed by:

- The BaRS Team
- Clinical representatives
- Solution Assurance representatives

### Sign-Off

All SCAL sections must have the status of "Approved" to complete assurance. Solution Assurance then issues a **Technical Conformance Certificate (TCC)** via email. The TCC is used to request access to the Production environment as part of the Connection Agreement.

---

## Stage 7: Deployment

### Key Deployment Activities

1. **Business Change** – Mapping current and future processes, developing SOPs, staff training.
2. **Solution Deployment & Configuration** – Completing "Connecting to environments" steps, installing and configuring the solution, providing service identifiers to the Endpoint Catalogue.
3. **Testing** – Multiple stages from INT development through to Production.
4. **Business Go Live** – Activating the system for live patient use.

### Testing Stages

| Stage | Environment | API Platform | Solution Environment | DoS |
|---|---|---|---|---|
| API Spec 'Try this API' | Sandbox | `sandbox.api.service.nhs.uk` | – | – |
| INT Development | INT | `int.api.service.nhs.uk` | Development (supplier) | UserTest |
| INT Assurance | INT | `int.api.service.nhs.uk` | Development (supplier) | UserTest |
| UAT Testing | INT | `int.api.service.nhs.uk` | UAT (provider) | UserTest |
| Technical Go Live | Prod | `api.service.nhs.uk` | Production (provider) | Production |

### Onboarding per Environment

Onboarding must occur for each environment independently – the steps are similar but must be replicated for INT and PROD.

### Technical Go Live

After successful UAT:

1. Schedule Technical Go Live.
2. Test all expected functional (happy path) workflows in the Production environment.
3. Make a Go/No Go decision.
4. Set a go-live date.

**Important**: Testing must occur in the live environment, but the service cannot handle patient interactions until all functional tests are successfully completed.

### Post Go-Live Support

Once in Production and running as BAU, if an issue arises:

- Service providers follow existing support processes with their supplier.
- If the issue lies with the BaRS API or a third party, raise an incident with the **National Service Desk** (`ssd.nationalservicedesk@nhs.net`).

---

## Information Governance

- An [NHS England Direction](https://digital.nhs.uk/about-nhs-digital/corporate-information-and-documents/directions-and-data-provision-notices/nhs-england-directions/booking-and-referral-standard-direction-2022) enables NHS Digital to develop BaRS.
- A [GDPR Register entry](https://digital.nhs.uk/data-and-information/keeping-data-safe-and-benefitting-the-public/gdpr/gdpr-register/booking-and-referral-standard) exists for BaRS.
- A Data Protection Impact Assessment (DPIA) is in place.
- Data traversing NHS Digital infrastructure is a combination of personal data and special category data.
- NHS Digital only collects BaRS API transactional data.
- The information model has been developed and approved by the Professional Records Standards Body (PRSB).

---

## Key Contacts

| Purpose | Contact |
|---|---|
| BaRS Team | `england.bookingandreferralstandard@nhs.net` |
| Propose a change to the standard | [Enquiry form](https://digital.nhs.uk/services/booking-and-referral-standard/enquiry-form) |
| Production incidents | `ssd.nationalservicedesk@nhs.net` |
| PROD Receiver certificates | `dir@nhs.net` |

---

## Sources

- [BaRS Implementation Guide v1.11.1](https://simplifier.net/guide/nhsbookingandreferralstandard/Home?version=1.11.1)
- [BaRS Assure Page](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Assure/Assure?version=1.11.1)
- [BaRS Testing & Environments](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Build/Testing-and-Environments?version=1.11.1)
- [BaRS Deployment Guide](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Deploy/Deploy?version=1.11.1)
- [Digital Onboarding Service](https://onboarding.prod.api.platform.nhs.uk/)
