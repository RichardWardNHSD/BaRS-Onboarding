# Onboarding to the Endpoint Catalogue API

## Overview

The **Endpoint Catalogue** (also referred to as the Endpoint Catalog or EPC) is the service discovery component of the BaRS ecosystem. It stores and serves information about **where** healthcare services can be reached — mapping service identifiers to routable endpoint addresses.

Any system that needs to:

- **Discover** where to send a booking, referral, or other BaRS interaction (consumer/read)
- **Register** its own endpoints so others can find it (provider/write)
- **Manage** healthcare services and their endpoint configurations (admin)

...will need to onboard to the Endpoint Catalogue API.

This guide covers onboarding for both **internal NHSE teams** and **external suppliers/providers**.

---

## What the Endpoint Catalogue Provides

The Endpoint Catalogue API is a FHIR R4 interface that manages:

| Resource | Purpose |
|---|---|
| **Endpoint** | The routable address (URL) for a service, with connection type and payload information |
| **HealthcareService** | The logical service (identified by ODS code and service ID) that owns one or more Endpoints |
| **List** | Priority ordering of Endpoints for a given HealthcareService |

### Environments

| Environment | Base URL |
|---|---|
| Sandbox | `https://sandbox.api.service.nhs.uk/endpoint-catalog/FHIR/R4` |
| Integration (INT) | `https://int.api.service.nhs.uk/endpoint-catalog/FHIR/R4` |
| Production (PROD) | `https://api.service.nhs.uk/endpoint-catalog/FHIR/R4` |

---

## How It Fits into BaRS

The Endpoint Catalogue is a **separate API** from the BaRS Proxy, but they work together:

```
┌─────────────┐         ┌─────────────────────────────────────────┐
│   Sender    │────────▶│  Apigee (NHS API Platform)              │
│   System    │         │                                         │
└─────────────┘         │  ┌───────────────┐   ┌──────────────┐  │
                        │  │  BaRS Proxy   │──▶│  EPC Backend │  │
                        │  │  (messaging)  │   │  (AWS)       │  │
                        │  └───────┬───────┘   └──────────────┘  │
                        └──────────┼──────────────────────────────┘
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │  Receiver System    │
                        └─────────────────────┘
```

- **BaRS Proxy** (inside Apigee) handles runtime messaging — routing bookings, referrals, and other BaRS messages to Receivers.
- **Endpoint Catalogue** (AWS backend) stores the endpoint directory — the Proxy calls it internally to resolve where to send messages.
- **Admin users and supplier systems** call the EPC directly to manage their endpoint registrations.

### Routing

| Path | Handled by | Purpose |
|---|---|---|
| `/$process-message`, `/Appointment*`, `/ServiceRequest*`, `/Slot`, etc. | BaRS Proxy | Runtime messaging |
| `/Endpoint*`, `/HealthcareService*`, `/List*` | EPC Backend (direct) | Catalogue management |

---

## Authentication & Authorisation

The Endpoint Catalogue supports two authentication modes:

| Mode | Token Type | Use Case |
|---|---|---|
| **Application-restricted** | Signed JWT → bearer token | System-to-system reads, automated supplier writes, BaRS Proxy internal lookups |
| **User-restricted (CIS2)** | NHS CIS2 login → bearer token | Human administrators managing endpoints (DoS leads, suppliers, programme admins) |

### Which operations require which mode

| Operation | App-restricted | CIS2 (user) |
|---|---|---|
| `GET /Endpoint` (lookup) | ✅ | Optional |
| `GET /HealthcareService` | ✅ | Optional |
| `GET /List` | ✅ | Optional |
| `POST /Endpoint` (create) | ⚠️ Automated only | ✅ for manual |
| `PUT /Endpoint/{id}` (update) | ⚠️ Automated only | ✅ for manual |
| `POST /HealthcareService` | ❌ | ✅ |
| `PUT /HealthcareService/{id}` | ❌ | ✅ |
| `DELETE` (any resource) | ❌ | ✅ (System Admin only) |
| `POST /List`, `PUT /List/{id}` | ❌ | ✅ |

### Authorisation checks

Apigee validates the token (signature, expiry, audience). The EPC backend then enforces:

- **ODS match** – The caller's ODS code (from token claims) must match the `NHSD-End-User-Organisation-ODS` header.
- **RBAC** – The user's role must permit the requested operation.
- **Ownership** – The caller's ODS/Product ID must match the resource's owner for write operations.

---

## Onboarding Steps

### For External Suppliers / Providers

External organisations onboarding to the Endpoint Catalogue should follow these steps:

#### 1. Register on the NHS Developer Portal

- Go to [https://digital.nhs.uk/developer](https://digital.nhs.uk/developer)
- Create an account and register your application.
- You'll receive an **App ID** and configure your **JWKS** (JSON Web Key Set) for signed JWT authentication.

#### 2. Request API Access

- From the Developer Portal, request access to the **Endpoint Catalog API** product.
- For INT: Access is generally available once your application is registered.
- For PROD: Access requires a completed Technical Conformance Certificate (TCC) from Solution Assurance, or explicit agreement with the BaRS Team.

#### 3. Connect to the API

**For read operations (service discovery):**

- Use application-restricted authentication (signed JWT).
- Call `GET /Endpoint` or `GET /HealthcareService` with appropriate query parameters.
- Include required headers: `X-Request-Id`, `X-Correlation-Id`, `NHSD-End-User-Organisation-ODS`.

**For write operations (registering your endpoints):**

- If automated (supplier system managing its own endpoints): use application-restricted auth with appropriate product scopes.
- If manual (human admin): use CIS2 user-restricted auth.
- Call `POST /Endpoint`, `POST /HealthcareService`, etc.

#### 4. Register Your Service

To be discoverable by the BaRS Proxy and other consumers:

1. Create a **HealthcareService** resource representing your logical service (with your ODS code and service identifier).
2. Create one or more **Endpoint** resources with the address (URL) of your receiver.
3. Optionally, create a **List** resource if you need to define priority ordering for multiple endpoints.

#### 5. Validate in INT

- Confirm your Endpoint is discoverable via `GET /Endpoint` queries.
- If using BaRS messaging, verify that the BaRS Proxy can resolve your endpoint and route messages to you.
- Use TKW or a partner supplier in INT to test end-to-end.

#### 6. Move to Production

- Once assured (TCC issued or BaRS Team agreement), repeat the registration steps against the Production environment.
- Update any firewall exceptions to receive messages from the BaRS Proxy in production.

---

### For Internal NHSE Teams

Internal teams have a lighter-touch process:

#### 1. Contact the BaRS Team

Email `england.bookingandreferralstandard@nhs.net` with:

- Your programme name
- What you need from the Endpoint Catalogue (read-only discovery, or managing your own endpoints)
- Your ODS code(s)
- Whether you need application-restricted access, CIS2 access, or both

#### 2. Get Credentials

**For application-restricted access:**
- Register on the NHS Developer Portal (same process as external).
- Request access to the Endpoint Catalog API product.

**For CIS2 user-restricted access:**
- Ensure your users have NHS CIS2 smartcards or equivalent identity.
- The BaRS Team will confirm your organisation's access level.

#### 3. Read-Only Access (Service Discovery)

If you only need to **look up** where services are registered:

- Use `GET /HealthcareService` to find services by ODS code or service identifier.
- Use `GET /Endpoint` to find the routable address for a given service.
- Use the `_include=HealthcareService:endpoint` parameter to get both in a single call.

No endpoint registration is required if you're only consuming.

#### 4. Write Access (Managing Endpoints)

If you need to **register your own service** in the catalogue:

- Agree the approach with the BaRS Team (manual via admin UI, or automated via API).
- For automated registration, your system will need a registered application with write scopes.
- For manual registration, your admin users will authenticate via CIS2.

#### 5. Environments

Same as external: onboard to INT first, validate, then move to PROD. Each environment is independent.

---

## Key API Operations

### Discovering a Service

```
GET /HealthcareService?identifier={system}|{value}&_include=HealthcareService:endpoint
```

This returns a Bundle containing the HealthcareService and its associated Endpoint resources.

### Querying Endpoints Directly

```
GET /Endpoint?identifier={system}|{value}
```

Or filter by connection type / payload type for more specific lookups.

### Creating an Endpoint

```
POST /Endpoint
Content-Type: application/fhir+json

{
  "resourceType": "Endpoint",
  "status": "active",
  "connectionType": { ... },
  "payloadType": [ ... ],
  "address": "https://your-receiver.nhs.uk/FHIR/R4",
  "managingOrganization": { ... },
  ...
}
```

### Required Headers (All Requests)

| Header | Required | Description |
|---|---|---|
| `X-Request-Id` | Yes | A unique UUID for the request |
| `X-Correlation-Id` | Yes | A UUID to correlate related requests |
| `NHSD-End-User-Organisation-ODS` | Yes | The ODS code of the requesting organisation |
| `Accept` | Yes | `application/fhir+json` |

---

## Differences from BaRS Proxy Onboarding

| Aspect | BaRS Proxy | Endpoint Catalogue |
|---|---|---|
| **Purpose** | Send/receive bookings, referrals, messages | Discover and manage service endpoints |
| **Authentication (Sender)** | OAuth (signed JWT) | OAuth (signed JWT) or CIS2 |
| **Authentication (Receiver)** | TLS-MA certificate | N/A (EPC is not a receiver of messages) |
| **Onboarding artefact** | SCAL → TCC | Developer Portal registration + BaRS Team agreement |
| **Assurance** | Formal SCAL process | Light-touch (confirm operations work in INT) |
| **Base URL** | `/booking-and-referral/FHIR/R4` | `/endpoint-catalog/FHIR/R4` |
| **Who calls it** | Sender systems (via Proxy) | Admin tools, supplier systems, BaRS Proxy (internally) |

---

## Common Scenarios

### "I'm a supplier building a BaRS Application"

You need **both**:
1. BaRS Proxy onboarding (to send/receive messages) — see [current onboarding process](./current-onboarding-process.md)
2. Endpoint Catalogue registration (so the Proxy can find your receiver) — this guide

### "I'm an internal team using the Standard Pattern"

You may need:
1. Endpoint Catalogue registration (if using the BaRS Proxy for routing) — this guide
2. Standard Pattern implementation — see [internal onboarding guide](./internal-onboarding-standard-pattern.md)

Or if you're **not using the Proxy** (Option B in the internal guide), you don't need to register in the Endpoint Catalogue at all.

### "I'm building an admin tool for managing endpoints"

You need:
1. Endpoint Catalogue onboarding with **CIS2 user-restricted** access — this guide
2. Agreement with the BaRS Team on roles and permissions

### "I just need to look up where a service is"

You need:
1. Application-restricted access to the Endpoint Catalogue — this guide (read-only)
2. No BaRS Proxy onboarding required

---

## Contacts

| Purpose | Contact |
|---|---|
| BaRS Team / EPC onboarding | `england.bookingandreferralstandard@nhs.net` |
| NHS Developer Portal support | [https://digital.nhs.uk/developer](https://digital.nhs.uk/developer) |
| API Platform issues | `api.management@nhs.net` |

---

## Sources

- [Endpoint Catalog API Specification (v1.4.0-alpha)](https://digital.nhs.uk/developer/api-catalogue/endpoint-catalog)
- [BaRS Implementation Guide](https://simplifier.net/guide/nhsbookingandreferralstandard/Home?version=1.11.1)
- [BaRS Testing & Environments](https://simplifier.net/guide/nhsbookingandreferralstandard/Home/Build/Testing-and-Environments?version=1.11.1)
- [NHS Developer Portal](https://digital.nhs.uk/developer)
