# Terms & Conditions — Versioned Acceptance with Audit Trail

**Electra Hub · Driver Portal · May 2026 · v1.0**

Covers: iOS app flows · Backend API & data model · Auth gate · Admin ops

---

## Table of Contents

1. [Overview](#1-overview)
2. [Data Model](#2-data-model)
3. [Backend API](#3-backend-api)
4. [Auth Gate & Enforcement](#4-auth-gate--enforcement)
5. [iOS App Flows](#5-ios-app-flows)
6. [Admin / Ops Tooling](#6-admin--ops-tooling)
7. [Audit Information Detail](#7-audit-information-detail)
8. [Sequence Diagrams](#8-sequence-diagrams)
9. [Open Questions](#9-open-questions)

---

## 1. Overview

This document describes the design for versioned Terms & Conditions acceptance across the Electra Hub driver portal ecosystem. The feature ensures every driver has explicitly accepted the current Terms before using protected APIs, and that a full, tamper-evident audit trail — including the accepting device — is stored for every acceptance event.

### 1.1 Goals

- Prompt every new driver to accept the current Terms & Conditions during signup.
- When Terms are updated, automatically detect which drivers have not accepted the new version and force a re-acceptance gate before they can continue.
- Record a tamper-evident audit record for every acceptance, capturing who accepted, when, from which device, from which IP address, and with which app version.
- Allow the operations team to publish a new Terms version and configure whether it requires universal re-acceptance.
- Keep acceptance enforcement inside the API gateway so no individual microservice needs to implement it.

### 1.2 Non-Goals

- Storing the full HTML/PDF content of each Terms version inside the database (a URL to immutable object storage is sufficient).
- Fine-grained regional Terms variants (future work).
- Cookie consent or GDPR data-processing consent (separate feature).

### 1.3 Terminology

| Term | Definition |
|---|---|
| **Terms Version** | A published, immutable snapshot of the Terms & Conditions document, identified by a sequential version number. |
| **Active Version** | The single version currently in force. Only one version is active at any point in time. |
| **Acceptance Record** | An audit row created when a user explicitly taps 'Accept' on a specific Terms Version, including full device and network context. |
| **Re-acceptance Gate** | The API-level enforcement that blocks requests and prompts the app to show the updated Terms when a user's latest acceptance is for an outdated version. |

---

## 2. Data Model

All Terms data lives in the existing relational database used by the Auth Service. Two new tables are introduced.

### 2.1 terms_versions

Stores each published version of the Terms & Conditions document. Rows are append-only and immutable once published.

| Field | Type | Nullable | Description |
|---|---|---|---|
| `id` | UUID | No | Primary key, auto-generated. |
| `version_number` | INTEGER | No | Monotonically increasing integer (1, 2, 3…). Used in API responses and gate comparisons. |
| `version_label` | VARCHAR(32) | No | Human-readable label shown in the UI, e.g. `2.1` or `May 2026`. |
| `content_url` | TEXT | No | HTTPS URL to the immutable Terms document in object storage (S3 / GCS). Must be a permanent, versioned URL. |
| `content_sha256` | VARCHAR(64) | No | SHA-256 hex digest of the document at `content_url`. Verified by the app before presenting to the user. |
| `requires_re_acceptance` | BOOLEAN | No | If TRUE, all users who have not yet accepted this version are blocked by the auth gate. |
| `effective_date` | TIMESTAMPTZ | No | UTC datetime from which this version is enforced. Can be set in the future for scheduled rollout. |
| `published_by` | UUID | No | FK → users.id. The admin account that published this version. |
| `created_at` | TIMESTAMPTZ | No | Row creation timestamp (server clock). |
| `is_active` | BOOLEAN | No | TRUE for the single currently-enforced version. Updated atomically when a new version becomes effective. |

> **Immutability rule:** Once a `terms_versions` row is inserted and `effective_date` has passed, no field may be updated (except `is_active` which flips to FALSE when a newer version activates). This preserves the integrity of all historical acceptance records that reference it.

### 2.2 terms_acceptances

An append-only audit log. One row is created every time a user taps 'I Accept'. Rows are never updated or deleted.

| Field | Type | Nullable | Description |
|---|---|---|---|
| `id` | UUID | No | Primary key, auto-generated. |
| `user_id` | UUID | No | FK → users.id. The driver who accepted. |
| `terms_version_id` | UUID | No | FK → terms_versions.id. The exact version that was accepted. |
| `accepted_at` | TIMESTAMPTZ | No | UTC timestamp recorded by the server at the moment the acceptance request is processed. |
| `device_id` | VARCHAR(128) | No | iOS: `identifierForVendor` UUID. Web: stable browser fingerprint ID. |
| `device_model` | VARCHAR(64) | No | iOS: `UIDevice.current.model` + `sysctlbyname("hw.machine")`, e.g. `iPhone16,2`. Web: `web`. |
| `os_version` | VARCHAR(32) | No | iOS: `UIDevice.current.systemVersion`, e.g. `18.4.1`. Web: parsed from user-agent. |
| `app_version` | VARCHAR(32) | No | `CFBundleShortVersionString` + `(CFBundleVersion)`, e.g. `2.3.1 (412)`. |
| `platform` | VARCHAR(16) | No | Enum: `iOS` \| `Android` \| `Web`. |
| `ip_address` | VARCHAR(45) | No | Recorded server-side from `X-Forwarded-For` / `REMOTE_ADDR`. Supports IPv4 and IPv6. |
| `user_agent` | TEXT | No | Full HTTP `User-Agent` header string as sent in the acceptance request. |

### 2.3 Indexes

| Index | Purpose |
|---|---|
| `terms_acceptances(user_id, terms_version_id)` | Fast lookup: has user X accepted version Y? Used by the auth gate on every request. |
| `terms_acceptances(terms_version_id)` | Admin reporting: count of acceptances per version, acceptance rate. |
| `terms_versions(is_active)` | Single-row lookup for the active version; used by the auth gate cache warm-up. |
| `terms_versions(effective_date)` | Scheduled activation job: find versions whose `effective_date` has arrived. |

### 2.4 Caching Strategy

The active Terms version changes extremely rarely (a few times per year). The auth gate caches the active `version_number` in Redis with a 5-minute TTL. On publication of a new version, the cache key is explicitly invalidated.

- Cache key: `terms:active_version_number`
- TTL: 300 seconds (refreshed on read).
- On new version activation: `HDEL terms:active_version_number` fired by the Admin Service.
- Per-user acceptance cache: `terms:user:{userId}:accepted_version`, TTL 1 hour. Invalidated when the user calls the Accept endpoint.

---

## 3. Backend API

The Terms feature is implemented as a new `terms-service` microservice behind the existing API gateway. It exposes four groups of endpoints.

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/terms/api/v1/terms/current` | Public | Return the active Terms version metadata (version number, label, content URL, SHA-256). |
| `GET` | `/terms/api/v1/terms/status` | Bearer | Return whether the authenticated user has accepted the current active version. |
| `POST` | `/terms/api/v1/terms/accept` | Bearer | Accept the current Terms version. Body: device audit payload. Creates an acceptance record. |
| `GET` | `/terms/api/v1/terms/history` | Bearer | Return the authenticated user's full acceptance history (all versions they have ever accepted). |
| `GET` | `/admin/api/v1/terms` | Admin | List all Terms versions with acceptance counts. |
| `POST` | `/admin/api/v1/terms` | Admin | Publish a new Terms version (draft, not yet active). Body: `version_label`, `content_url`, `content_sha256`, `requires_re_acceptance`, `effective_date`. |
| `PUT` | `/admin/api/v1/terms/{id}/activate` | Admin | Immediately activate a Terms version (overrides `effective_date`). |
| `GET` | `/admin/api/v1/terms/{id}/acceptances` | Admin | Page through acceptance records for a specific version, with device audit fields. |

### 3.1 GET /terms/api/v1/terms/current

Public — no authentication required. Used by the iOS app on first launch and at signup to display the Terms before the user has a token.

```json
{
  "versionNumber": 3,
  "versionLabel": "May 2026",
  "contentUrl": "https://cdn.electrahub.com/legal/terms/v3.html",
  "contentSha256": "a1b2c3d4...64 hex chars...",
  "effectiveDate": "2026-05-01T00:00:00Z"
}
```

### 3.2 POST /terms/api/v1/terms/accept

Requires a valid Bearer token. The request body carries the device audit payload assembled by the iOS app. The server adds `ip_address` and `user_agent` from the HTTP request.

**Request body:**

| Field | Type | Description |
|---|---|---|
| `deviceId` | string | iOS `identifierForVendor` UUID. |
| `deviceModel` | string | Raw hardware identifier, e.g. `iPhone16,2`. |
| `osVersion` | string | iOS version string, e.g. `18.4.1`. |
| `appVersion` | string | `CFBundleShortVersionString (CFBundleVersion)`, e.g. `2.3.1 (412)`. |
| `platform` | string | Enum: `iOS` \| `Android` \| `Web`. |

```json
{
  "deviceId": "B4D1F95A-1234-5678-ABCD-EF0123456789",
  "deviceModel": "iPhone16,2",
  "osVersion": "18.4.1",
  "appVersion": "2.3.1 (412)",
  "platform": "iOS"
}
```

**Idempotency:** if the user has already accepted the current version, the endpoint returns 200 with the existing acceptance record rather than creating a duplicate row.

**Response (201 Created):**

| Field | Description |
|---|---|
| `acceptanceId` | UUID of the newly created acceptance record. |
| `acceptedAt` | Server-recorded UTC timestamp. |
| `termsVersionNumber` | Confirms which version was accepted. |

### 3.3 GET /terms/api/v1/terms/status

Returns a simple object indicating whether the caller's latest acceptance covers the current active version.

| Field | Type | Meaning |
|---|---|---|
| `termsAccepted` | boolean | TRUE if user has accepted the current active version. |
| `currentVersionNumber` | integer | The active version number. |
| `acceptedVersionNumber` | integer? | The version the user last accepted (null if never accepted). |

---

## 4. Auth Gate & Enforcement

Terms enforcement lives entirely inside the API gateway (`api-gateway` service). No individual microservice needs to implement any check.

### 4.1 Gateway Filter Logic

A new `TermsGatewayFilter` runs after authentication but before request forwarding. It is applied to all routes that require authentication except the Terms endpoints themselves.

1. Extract `userId` from the validated JWT claims.
2. Read the active terms `version_number` from the Redis cache (or DB fallback).
3. Check the per-user acceptance cache: `terms:user:{userId}:accepted_version`.
4. If cached value equals the active `version_number` → allow the request through immediately.
5. Otherwise query `terms_acceptances` for `(user_id, active version_id)`. Update the cache from the result.
6. If the user has not accepted the current version → return **HTTP 451** with the gate error body.

> **HTTP 451 — Unavailable For Legal Reasons:** RFC 7725 defines status 451 for legal compliance blocks. The iOS app intercepts this specific status code at the API client level to trigger the Terms acceptance modal, so no feature-level code needs to handle it.

### 4.2 Gate Error Response (HTTP 451)

```json
{
  "error": "TERMS_ACCEPTANCE_REQUIRED",
  "currentVersionNumber": 3,
  "currentVersionLabel": "May 2026",
  "contentUrl": "https://cdn.electrahub.com/legal/terms/v3.html",
  "contentSha256": "a1b2c3..."
}
```

### 4.3 Excluded Routes

The following routes bypass the Terms gate to avoid circular dependencies:

- `/terms/api/v1/terms/**` — Terms endpoints themselves.
- `/auth/api/auth/login`, `/auth/api/auth/register` — login and signup must work before Terms can be accepted.
- `/auth/api/auth/refresh`, `/auth/api/auth/logout-*` — token lifecycle.
- `/session/api/v1/sessions/active/stream` — SSE long-lived stream (Terms checked at connection start only).

### 4.4 New Version Rollout — Race Condition Handling

When a new Terms version activates, in-flight requests may carry a stale cached acceptance. The filter handles this gracefully:

- **Stale cache hit** (user accepted v2, v3 just activated, cache says v2): the next request after TTL expiry will miss the cache, re-query the DB, find the user has not accepted v3, and return 451.
- There is no risk of permanently blocking a user — as soon as they accept the new version, the per-user cache is explicitly invalidated and subsequent requests succeed immediately.

---

## 5. iOS App Flows

### 5.1 New Files

| File | Responsibility |
|---|---|
| `Services/TermsService.swift` | Fetches current Terms metadata and posts acceptance with audit payload. |
| `Models/TermsModels.swift` | `TermsVersion`, `TermsStatus`, `TermsAcceptRequest`, `TermsAcceptResponse` structs. |
| `Utilities/DeviceAuditInfo.swift` | Assembles the device audit payload (device ID, model, OS version, app version). |
| `Views/Terms/TermsAcceptanceView.swift` | Full-screen modal: renders Terms `WKWebView` + Accept / Decline buttons. |
| `Views/Terms/TermsGateModifier.swift` | SwiftUI `ViewModifier` that listens for `termsRequired` signal and presents the modal. |

### 5.2 Flow A — Signup (First-Time Acceptance)

Terms are shown during the registration flow before the account is created, so the user has a token-free view of the Terms content before committing.

1. User opens the app → `AuthFlowView` → taps 'Create Account'.
2. Registration form is filled in. On 'Continue', the app calls `GET /terms/api/v1/terms/current` (no auth required).
3. App presents `TermsAcceptanceView` as a sheet — the Terms HTML is loaded in a `WKWebView` from `contentUrl`. The app verifies the SHA-256 of the downloaded content matches `contentSha256` before rendering.
4. User scrolls to the bottom; 'I Accept' button becomes enabled.
5. User taps 'I Accept':
   - App calls `POST /auth/api/auth/register` to create the account and receive a JWT.
   - App immediately calls `POST /terms/api/v1/terms/accept` with the device audit payload and the JWT from registration.
   - Both calls succeed → user is taken to the main app.
6. If the user taps 'Decline', registration is cancelled and they are returned to the form.

> **Why accept after registration?** The user must have an account (and `userId`) before an acceptance record can be created. The order is: create account → accept Terms. The Terms gate will not block the accept call itself (it is in the excluded routes list).

### 5.3 Flow B — Re-Acceptance Gate (Returning User, Updated Terms)

This flow triggers transparently whenever any API call returns HTTP 451.

1. User opens the app, token refreshes successfully, user navigates to any screen.
2. A background API call (e.g. `getActiveSessions`) returns HTTP 451 with `TERMS_ACCEPTANCE_REQUIRED`.
3. `APIClient.decode()` sees status 451 → throws `TermsAcceptanceRequiredError`, embedding the `TermsVersion` from the response body.
4. `TermsGateModifier` (attached at the root `NavigationStack` level) observes the error through the shared `TermsGate` publisher.
5. `TermsAcceptanceView` slides up as a full-screen cover, blocking all app navigation.
6. User reads and accepts the new Terms (same `WKWebView` + SHA-256 verification as Flow A).
7. App calls `POST /terms/api/v1/terms/accept` → 201 Created.
8. `TermsAcceptanceView` dismisses. The original triggering request is retried automatically.

> **Decline on re-acceptance:** If the user declines updated Terms, they are shown an informational screen explaining they cannot use the service until they accept. The app is not logged out — the user may choose to accept later by returning to the app.

### 5.4 TermsService — Key Interface

| Method | Description |
|---|---|
| `getCurrentTerms() async throws -> TermsVersion` | Calls `GET /terms/current`. Caches result in memory for the app session. |
| `getTermsStatus() async throws -> TermsStatus` | Calls `GET /terms/status`. Returns whether current user is up to date. |
| `acceptTerms(version: TermsVersion) async throws -> TermsAcceptResponse` | Builds `DeviceAuditInfo`, calls `POST /terms/accept`. Invalidates local terms status cache. |

### 5.5 DeviceAuditInfo — Audit Payload Assembly

`DeviceAuditInfo.swift` collects the four device-side audit fields and exposes them as a Swift struct that is `Encodable` into the accept request body.

| Field | Source | Example | Notes |
|---|---|---|---|
| `deviceId` | iOS | `B4D1F95A-1234-...` | `UIDevice.current.identifierForVendor?.uuidString`. Stable per device per app vendor until app uninstall. |
| `deviceModel` | iOS | `iPhone16,2` | Read via `sysctlbyname("hw.machine")`. The raw hardware identifier, not the marketing name. |
| `osVersion` | iOS | `18.4.1` | `UIDevice.current.systemVersion`. |
| `appVersion` | iOS | `2.3.1 (412)` | `Bundle.main.infoDictionary`: `CFBundleShortVersionString` + `CFBundleVersion`. Uniquely identifies the exact build. |
| `platform` | iOS | `iOS` | Hardcoded enum value in the iOS client. |
| `ip_address` | Server | `203.0.113.42` | Captured server-side from `X-Forwarded-For` header (first hop). Never sent by the client. |
| `user_agent` | Server | `DriverPortal/2.3.1 (iPhone; iOS 18.4.1)` | Full HTTP `User-Agent` header captured server-side. |

---

## 6. Admin / Ops Tooling

### 6.1 Publishing a New Terms Version

Publishing is a two-step operation so that the legal/ops team can review a draft before it goes live. Only users with the `ADMIN` role can call the admin endpoints.

1. Ops team uploads the new Terms HTML/PDF to the CDN (object storage), notes the permanent URL and SHA-256 hash.
2. Ops calls `POST /admin/api/v1/terms` with the version metadata. The new version is created with `is_active = FALSE`.
3. Ops optionally sets `effective_date` in the future (scheduled activation) or calls `PUT /admin/api/v1/terms/{id}/activate` for immediate activation.
4. On activation:
   - The current active version's `is_active` flag is set to `FALSE`.
   - The new version's `is_active` is set to `TRUE`.
   - Redis cache keys `terms:active_version_number` and all `terms:user:*:accepted_version` are invalidated.
   - An audit event is written to the platform audit log: `TERMS_VERSION_ACTIVATED` by admin `userId`.

> **Scheduled activation:** Setting `effective_date` in the future allows ops to schedule a Terms update (e.g. '2026-06-01 00:00 UTC'). A daily background job (`TermsActivationScheduler`) in the `terms-service` checks for versions whose `effective_date` has passed and activates them automatically.

### 6.2 Viewing Acceptance Records

`GET /admin/api/v1/terms/{versionId}/acceptances` returns paginated acceptance records with all audit fields.

| Query param | Description |
|---|---|
| `page` / `size` | Pagination (default: `page=0`, `size=100`). |
| `userId` | Filter to a specific user's acceptance record. |
| `deviceId` | Filter to a specific device (useful for fraud investigation). |
| `from` / `to` | UTC datetime range filter on `accepted_at`. |

### 6.3 Backward-Compatible vs. Requiring Re-Acceptance

The `requires_re_acceptance` flag controls whether the auth gate enforces the new version on existing users.

| Scenario | Recommended setting |
|---|---|
| Typo fix, formatting only | `requires_re_acceptance = FALSE`. The new version is recorded, but existing users are not blocked. |
| New data-sharing clause | `requires_re_acceptance = TRUE`. All users must re-accept before using the API. |
| Legal jurisdiction change | `requires_re_acceptance = TRUE`. Coordinate with legal on a re-acceptance deadline. |

> **Rollback:** If a newly activated version causes problems, an admin can re-activate the previous version via `PUT /admin/api/v1/terms/{prevId}/activate`. Existing acceptance records are unaffected. Users who already accepted the new version will not need to re-accept the rolled-back version since their old acceptance record still exists.

---

## 7. Audit Information Detail

Every acceptance record is a legally admissible evidence artefact.

### 7.1 Field Sources

| Field | Source | Example | Notes |
|---|---|---|---|
| `accepted_at` | Server clock | `2026-05-07T14:32:01.412Z` | Recorded by the `terms-service` at the moment the transaction commits. Client-supplied timestamps are not accepted. |
| `device_id` | iOS app | `B4D1F95A-...` | `identifierForVendor`. Note: resets on app reinstall. Stored as-is; uniqueness within a session is sufficient for audit. |
| `device_model` | iOS app | `iPhone16,2` | Raw `hw.machine` sysctl. The `terms-service` maps this to a human-readable marketing name (`iPhone 16 Pro`) in admin reports. |
| `os_version` | iOS app | `18.4.1` | `UIDevice.systemVersion`. Validated server-side: must be a semver string, max 32 chars. |
| `app_version` | iOS app | `2.3.1 (412)` | `CFBundleShortVersionString` + `CFBundleVersion`. Validated: must match pattern `X.Y.Z (N)`, max 32 chars. |
| `platform` | iOS app | `iOS` | Enum; validated server-side. Allowed: `iOS` \| `Android` \| `Web`. |
| `ip_address` | Server | `203.0.113.42` | Extracted from `X-Forwarded-For` (first untrusted IP). Falls back to `REMOTE_ADDR`. Stored as `VARCHAR(45)` to support IPv6. |
| `user_agent` | Server | `DriverPortal/2.3.1 ...` | Full HTTP `User-Agent` header. Stored as `TEXT`. Never truncated. |

### 7.2 Data Retention

- Acceptance records must be retained for a minimum of **7 years** from the date of acceptance (legal obligation).
- Records are never deleted by application code. Hard-deletes require a DBA operation with a written justification ticket.
- The `terms_acceptances` table is excluded from any automated GDPR erasure jobs. Legal holds take precedence.

### 7.3 Integrity Assurance

- All rows are written in a database transaction: the acceptance record is only committed if the HTTP response is 201. No orphan records.
- The `terms_versions.content_sha256` field allows auditors to independently verify the exact document the user saw at the time of acceptance.
- Database-level triggers (`INSERT ONLY`) prevent `UPDATE` or `DELETE` on `terms_acceptances`. Any attempt raises an exception and writes to the DB audit log.
- Admin actions (publish, activate) are recorded in the platform audit log with the admin's `userId`, IP, and timestamp.

---

## 8. Sequence Diagrams

### 8.1 Signup Flow

```
iOS App
  1. GET /terms/current  →  Terms Service  →  {versionNumber, contentUrl, sha256}
  2. Download & verify SHA-256 of Terms document from CDN
  3. Present TermsAcceptanceView (WKWebView)
  4. User taps 'I Accept'
  5. POST /auth/register  →  Auth Service  →  {accessToken}
  6. POST /terms/accept + DeviceAuditInfo  →  Terms Service  →  {acceptanceId}
  7. Navigate to Dashboard
```

### 8.2 Re-Acceptance Gate Flow

```
iOS App (returning user, Terms v3 now active)
  1. GET /session/sessions/active  →  API Gateway
  2. Gateway: validate JWT  →  check Redis cache  →  cache miss
  3. Gateway: query DB  →  user accepted v2, active is v3  →  block
  4. HTTP 451 {TERMS_ACCEPTANCE_REQUIRED, contentUrl, sha256}  →  iOS
  5. APIClient throws TermsAcceptanceRequiredError  →  TermsGate publisher
  6. TermsGateModifier presents TermsAcceptanceView (full-screen cover)
  7. User accepts  →  POST /terms/accept + DeviceAuditInfo  →  201 OK
  8. Gateway cache for user invalidated
  9. Modal dismissed  →  original request retried  →  200 OK
```

---

## 9. Open Questions

| # | Question | Notes |
|---|---|---|
| 1 | Should the terms content be stored in the DB or only linked via URL? | URL + SHA-256 preferred. Avoids DB bloat; CDN handles versioned immutability. |
| 2 | How long should the Terms gate cache TTL be? | 5 min recommended for the active version; 1 hr per-user. Adjust based on rollout urgency requirements. |
| 3 | Do we need regional variants (e.g. separate EU Terms)? | Out of scope for v1. If needed, add a `region` column to `terms_versions` and resolve via user profile. |
| 4 | What happens if the CDN URL for Terms content is unreachable at accept time? | App should display a cached version if available; otherwise show an error and prevent acceptance until content can be verified. |
| 5 | Should we send a push notification when Terms change? | Useful UX improvement. Out of scope for v1 — the gate handles enforcement without a notification. |
