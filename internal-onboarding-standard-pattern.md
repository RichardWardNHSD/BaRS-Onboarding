# BaRS Onboarding for Internal Teams – Standard Pattern (Appointments)

## Purpose

This guide is for **internal NHSE programme teams** who want to adopt the BaRS Standard Pattern for Appointment Management. These teams typically:

- Do not have a formal BaRS Application (use case) assigned to them.
- Want to use the standard appointment workflows (book, view, update, cancel, reschedule, rebook).
- May or may not require the full BaRS Proxy infrastructure depending on their architecture.

The Standard Pattern – Appointments is currently in **Alpha** and only available to internal NHSE programmes. External suppliers should refer to the [full onboarding process](./current-onboarding-process.md).

---

## What Is the Standard Pattern – Appointments?

The BaRS Standard Pattern provides a generic **Appointment Management Foundation** for teams that need booking functionality but don't fit into one of the published BaRS Applications (e.g., 111-to-ED, Validation, Minor Illness referral).

It supports the following capabilities using the FHIR R4 [UKCore-Appointment](https://simplifier.net/HL7FHIRUKCoreR4/UKCore-Appointment) resource:

| Capability | Endpoint | Method | Description |
|---|---|---|---|
| **List** | `/DocumentReference` | GET | View existing appointments for a patient via the central Registry |
| **View** | `/Appointment/{id}` | GET | Retrieve a specific Appointment from the owning service |
| **View (Search)** | `/Appointment` | GET | Look up by NHS Number or demographics if the `id` is unknown |
| **Get Slots** | `/Slot` | GET | Obtain available booking slots from a receiving system |
| **Book** | `/Appointment` | POST | Create a new booking (uses discrete endpoint, not `$process-message`) |
| **Cancel** | `/Appointment/{id}` | PUT | Set status to `cancelled` or `entered-in-error` |
| **Update** | `/Appointment/{id}` | PUT | Modify `.status` or `.reasonCode` |
| **Reschedule** | `/Appointment/{id}` | PATCH | Change the slot against an existing appointment |
| **Rebook** | Cancel + Book | Composite | Cancel the existing booking then create a new one |

**Key rule**: All operations that modify a resource must be preceded by a GET (read before write).

### Registry (DocumentReference)

The Standard Pattern introduces a central **Registry** using the DocumentReference resource. After a booking is made, the Receiver must POST a pointer to the Registry. This allows Senders to discover existing bookings for a patient without needing to know which service holds them.

> **⚠️ Note on Registry / NRL Integration**
>
> Whilst the BaRS standard states the Registry (DocumentReference) integration as a **MUST** requirement, the planned integration with the **National Record Locator (NRL)** has been **placed on hold**. This means that the central repository for booking pointers is not currently available for use.
>
> Internal teams should be aware of this when planning their implementation. In practice:
> - The DocumentReference interactions (POST, PUT, DELETE pointers) described in the standard cannot currently be fulfilled against the NRL.
> - Teams may choose to implement the pattern against a local or programme-level registry if cross-service appointment visibility is needed now.
> - Alternatively, teams can defer the Registry integration and implement the core booking workflows (Book, View, Update, Cancel, Reschedule, Rebook) without it, noting that the List capability (which relies on the Registry) will not be available until the NRL integration is resumed.
>
> Contact the BaRS Team for the latest status on this.

---

## Adoption Options

Internal teams have flexibility in how they adopt the standard. The two main options relate to whether the team uses the **BaRS Proxy** (the central API-M infrastructure) or implements the standard pattern **directly** between systems.

### Option A: Full BaRS Proxy (Recommended)

This option uses the central BaRS API Proxy hosted on the NHS API Platform. It is the recommended approach as it provides:

- **Centralised authentication** – The Proxy handles OAuth token validation for Senders.
- **Mutual TLS** – The Proxy handles TLS-MA to Receivers, providing a consistent security model.
- **Service discovery** – The Endpoint Catalogue routes requests to the correct Receiver based on service identifiers.
- **Capability negotiation** – Senders and Receivers confirm compatibility via `/metadata` and `/MessageDefinition` before engaging.
- **Consistency** – Follows the same patterns as all other BaRS implementations, making interoperability simpler.

**What you'll need to do:**

1. Contact the BaRS Team (`england.bookingandreferralstandard@nhs.net`) to register your interest.
2. Connect as a Sender (OAuth/signed JWT via NHS Developer portal).
3. Connect as a Receiver (TLS-MA certificate under NHS Root CA).
4. Register your service in the Endpoint Catalogue.
5. Implement the Standard Pattern workflows.
6. Implement the DocumentReference (Registry) pattern.

**Environments**: You'll onboard to INT first, then PROD. Each environment requires separate onboarding.

**Assurance**: For internal teams using the Standard Pattern, the formal SCAL process is **significantly lighter** than for external suppliers. Agree the scope with the BaRS Team – typically this involves:

- Demonstrating the core workflows function correctly in INT.
- Submitting TKW validation reports (if applicable).
- A brief walkthrough with the BaRS Team rather than a full end-to-end demonstration.

### Option B: Standard Only (No Proxy)

Some internal teams may want to adopt the **BaRS standard and data model** without routing traffic through the central Proxy. This is appropriate when:

- Both Sender and Receiver are within the same programme or trust boundary.
- There is an existing point-to-point integration channel (e.g., internal API gateway, service mesh).
- The team wants interoperability alignment with BaRS without the overhead of Proxy onboarding.

**What this means in practice:**

- You adopt the **FHIR resource definitions** (UKCore-Appointment, Slot, DocumentReference, etc.).
- You implement the **same RESTful interaction patterns** (POST to book, GET before PUT to update, PATCH to reschedule).
- You follow the **same status lifecycle** (`booked` → `cancelled`, `entered-in-error`).
- You may implement your own service discovery mechanism (or use a shared configuration).
- You handle authentication and authorisation between your own systems.

**What you don't need:**

- NHS Developer portal registration (unless you want it for other APIs).
- TLS-MA certificates under the NHS Root CA.
- Endpoint Catalogue registration.
- Formal connection to the BaRS Proxy.

**What you should still do:**

- Conform to the UKCore-Appointment profile.
- Follow the read-before-write pattern for all modifications.
- Implement the DocumentReference (Registry) pattern if you need cross-service appointment visibility.
- Contact the BaRS Team to ensure alignment and avoid divergence from the standard.

**Assurance**: No formal SCAL is required. The BaRS Team may ask for a lightweight review to confirm conformance to the data model and interaction patterns.

---

## Comparison of Options

| Aspect | Option A: Full Proxy | Option B: Standard Only |
|---|---|---|
| **Security model** | Centralised (OAuth + TLS-MA via Proxy) | Self-managed |
| **Service discovery** | Endpoint Catalogue | Self-managed |
| **Interoperability** | Immediate with any BaRS-connected system | Limited to systems within your boundary |
| **Onboarding effort** | Moderate (INT + PROD registration) | Low (data model conformance only) |
| **Assurance** | Light-touch (demo + TKW reports) | Informal (BaRS Team review) |
| **Future-proofing** | Full – ready for cross-organisation use | Partial – would need Proxy onboarding later |
| **Registry access** | Yes (central DocumentReference store) | Self-hosted or not required |

---

## Getting Started

Regardless of which option you choose:

1. **Read [BaRS Core](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1)** – Understand the end-to-end workflow, security model, and error handling.
2. **Review the [Standard Pattern – Appointments](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1/Appointment-StandardPattern?version=1.11.1)** – Understand the specific interactions, resource examples, and sequence diagrams.
3. **Review the [DocumentReference Standard Pattern](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1/DocumentReference-StandardPattern?version=1.11.1)** – Understand how the Registry works for listing bookings.
4. **Contact the BaRS Team** at `england.bookingandreferralstandard@nhs.net` – Let them know your programme, your chosen option, and your timeline.

---

## Key Differences from Full External Onboarding

| Full External Process | Internal Standard Pattern |
|---|---|
| Must select a published BaRS Application | No Application required – uses generic Appointment Management Foundation |
| Full SCAL completion (2–4 weeks) | Light-touch assurance or informal review |
| End-to-end demonstration with BaRS Team, Clinical, and Solution Assurance | Brief walkthrough with BaRS Team (Option A) or data model review (Option B) |
| Technical Conformance Certificate (TCC) required for PROD | TCC may not be required depending on option |
| Must use `$process-message` for bookings within Applications | Uses discrete `/Appointment` endpoint (POST, PUT, PATCH) |
| Mandatory connection to BaRS Proxy | Optional (Option B allows direct integration) |

---

## Important Notes

- The Standard Pattern is **Alpha** – it is subject to change.
- Only **internal NHSE programmes** are currently permitted to build against it.
- If you are interested in external adoption, contact the BaRS Team.
- The BaRS Team can advise on whether your use case might be better served by a published Application.
- Even in Option B, alignment with the standard ensures you can transition to the full Proxy model later without rework.

---

## Contacts

| Purpose | Contact |
|---|---|
| BaRS Team (general enquiries, onboarding) | `england.bookingandreferralstandard@nhs.net` |
| Propose changes to the standard | [Enquiry form](https://digital.nhs.uk/services/booking-and-referral-standard/enquiry-form) |

---

## Sources

- [Standard Pattern – Appointments (Core v1.4.1)](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1/Appointment-StandardPattern?version=1.11.1)
- [DocumentReference Standard Pattern](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1/DocumentReference-StandardPattern?version=1.11.1)
- [BaRS Core](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Core/1-4-1?version=1.11.1)
- [BaRS API Specification v1.4.1](https://digital.nhs.uk/developer/api-catalogue/booking-and-referral-fhir/v1.4.1)
- [Testing & Environments](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Build/Testing-and-Environments?version=1.11.1)
