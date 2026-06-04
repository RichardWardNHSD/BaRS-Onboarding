# Onboarding to the Endpoint Catalogue API

## Overview

The **Endpoint Catalogue** (also referred to as the EPC) is the service discovery component of the BaRS ecosystem. It stores and serves information about **where** healthcare services can be reached — mapping service identifiers to routable endpoint addresses.

Any system that needs to:

- **Manage** endpoints directly (self-service registration, updates, decommissioning)
- **Discover** services for purposes outside of BaRS messaging (e.g., admin tooling, directory queries)

...will need to onboard to the Endpoint Catalogue API directly.

> **Important**: If you are an external supplier building a standard BaRS integration, you do **not** need direct access to the Endpoint Catalogue API. The BaRS Proxy handles endpoint resolution (`GET /Endpoint`) internally on your behalf. Your endpoint is registered in the catalogue by the **BaRS run/maintain team** as part of supplier onboarding. See the [current onboarding process](./current-onboarding-process.md) for that journey.

This guide covers direct EPC onboarding for both **internal NHSE teams** and **external suppliers/providers who need to manage endpoints themselves**.

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

- **BaRS Proxy** (inside Apigee) handles runtime messaging — routing bookings, referrals, and other BaRS messages to Receivers. The Proxy calls the EPC internally to resolve target endpoints; **suppliers do not need direct EPC access for this**.
- **Endpoint Catalogue** (AWS backend) stores the endpoint directory. During standard BaRS onboarding, the **run/maintain team** registers supplier endpoints here on the supplier's behalf.
- **Admin users and internal systems** call the EPC directly to manage endpoint registrations (this is the focus of this guide).

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

#### Standard BaRS Integration (Most Suppliers)

If you are building a BaRS integration (booking, referral, validation, etc.), you do **not** need to onboard to the Endpoint Catalogue API directly. Here's how it works:

1. You complete the standard [BaRS onboarding process](./current-onboarding-process.md) — connecting as a Sender (OAuth) and/or Receiver (TLS-MA).
2. As part of that onboarding, you provide the BaRS Team with your Receiver URL.
3. The **BaRS run/maintain team** registers your endpoint in the Endpoint Catalogue on your behalf.
4. The BaRS Proxy then resolves your endpoint internally when routing messages to you.

You never call `GET /Endpoint` yourself — the Proxy does it for you.

> **Future state**: Full self-service endpoint management for suppliers is planned but not yet available. Until then, endpoint registration and updates are handled by the run/maintain team during supplier onboarding.

#### Direct EPC Access (Self-Service Endpoint Management)

Some suppliers may need direct access to the Endpoint Catalogue API — for example, if they are:

- Building tooling to manage their own endpoints across multiple services
- Operating as a platform provider managing endpoints on behalf of multiple organisations
- Participating in a self-service pilot

In this case:

##### 1. Register on the NHS Developer Portal

- Go to [https://digital.nhs.uk/developer](https://digital.nhs.uk/developer)
- Create an account and register your application.
- You'll receive an **App ID** and configure your **JWKS** (JSON Web Key Set) for signed JWT authentication.

##### 2. Request API Access

- From the Developer Portal, request access to the **Endpoint Catalog API** product.
- For INT: Access is generally available once your application is registered.
- For PROD: Access requires explicit agreement with the BaRS Team.

##### 3. Connect to the API

**For read operations (querying the catalogue):**

- Use application-restricted authentication (signed JWT).
- Call `GET /Endpoint` or `GET /HealthcareService` with appropriate query parameters.
- Include required headers: `X-Request-Id`, `X-Correlation-Id`, `NHSD-End-User-Organisation-ODS`.

**For write operations (managing endpoints):**

- If automated (supplier system managing its own endpoints): use application-restricted auth with appropriate product scopes.
- If manual (human admin): use CIS2 user-restricted auth.
- Call `POST /Endpoint`, `POST /HealthcareService`, etc.

##### 4. Validate in INT

- Confirm your Endpoint is discoverable via `GET /Endpoint` queries.
- If using BaRS messaging, verify that the BaRS Proxy can resolve your endpoint and route messages to you.

##### 5. Move to Production

- Once agreed with the BaRS Team, repeat the steps against the Production environment.

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
- Use `GET /Endpoint?_has:HealthcareService:endpoint:_id={id}` to find the routable address for a given service.

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

To find a HealthcareService:

```
GET /HealthcareService?identifier={system}|{value}
```

Then retrieve its Endpoints:

```
GET /Endpoint?_has:HealthcareService:endpoint:_id={healthcare-service-id}
```

This two-step pattern ensures Endpoints are returned with full visibility filtering (status, period, HealthcareService.active) and support for `connectionType` and `payloadType` filtering.

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
| **Who calls it** | Sender systems (via Proxy) | Admin tools, internal systems, BaRS run/maintain team, future self-service suppliers |

---

## Common Scenarios

### "I'm a supplier building a BaRS Application"

You **don't** need direct EPC access. The BaRS Proxy resolves endpoints internally, and the run/maintain team registers your endpoint for you during onboarding. See the [current onboarding process](./current-onboarding-process.md).

### "I'm a supplier and want to manage my own endpoints"

You need direct Endpoint Catalogue onboarding — see the "Direct EPC Access" section above. This is not yet generally available; contact the BaRS Team to discuss.

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
