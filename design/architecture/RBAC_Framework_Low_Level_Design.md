# RBAC Framework Low Level Design (LLD)

Date: 2026-03-20  
Scope: Implementation-level design for RBAC across `api-gateway`, `user-service`, and `admin-poratl-ui`.

## 1. Repository Scope

- Gateway enforcement:
  - `/Users/amolsurjuse/development/projects/api-gateway/src/main/java/com/electrahub/gateway/security/ApiPolicyAuthorizationManager.java`
  - `/Users/amolsurjuse/development/projects/api-gateway/src/main/java/com/electrahub/gateway/security/CachedRbacPolicySnapshotProvider.java`
  - `/Users/amolsurjuse/development/projects/api-gateway/src/main/java/com/electrahub/gateway/security/RbacCacheController.java`
  - `/Users/amolsurjuse/development/projects/api-gateway/src/main/java/com/electrahub/gateway/config/RbacProperties.java`

- Policy source and admin APIs:
  - `/Users/amolsurjuse/development/projects/user-service/src/main/java/com/electrahub/user/service/RbacPolicyService.java`
  - `/Users/amolsurjuse/development/projects/user-service/src/main/java/com/electrahub/user/api/AdminRbacController.java`
  - `/Users/amolsurjuse/development/projects/user-service/src/main/java/com/electrahub/user/api/InternalRbacController.java`
  - `/Users/amolsurjuse/development/projects/user-service/src/main/java/com/electrahub/user/domain/RbacPolicy.java`
  - `/Users/amolsurjuse/development/projects/user-service/src/main/java/com/electrahub/user/domain/RbacPolicyRule.java`
  - `/Users/amolsurjuse/development/projects/user-service/src/main/resources/db/changelog/changes/0005-create-rbac-policy.yaml`

- Admin UI:
  - `/Users/amolsurjuse/development/projects/admin-poratl-ui/src/pages/RbacPolicyPage.tsx`
  - `/Users/amolsurjuse/development/projects/admin-poratl-ui/src/api/rbacPolicy.ts`

## 2. Component-Level Design

## 2.1 Gateway Components
- `JwtAuthFilter`
  - Parses `Authorization: Bearer <token>`.
  - Validates issuer/signature/expiration.
  - Applies denylist and token-version checks.
  - Builds Spring authorities (`ROLE_*`).

- `CachedRbacPolicySnapshotProvider`
  - Maintains atomic policy state.
  - On-demand refresh for authorization path.
  - Scheduled refresh every 30s.
  - Pulls policy from user-service internal API using `X-Internal-Api-Key`.
  - Falls back to last good snapshot or config-based policy.

- `ApiPolicyAuthorizationManager`
  - Compiles snapshot to runtime rules keyed by policy version.
  - Evaluates method+path using `AntPathMatcher`.
  - Enforces `DENY` precedence, allow-anonymous logic, required roles.
  - Expands effective roles via role hierarchy.

- `RbacCacheController`
  - `POST /internal/rbac/cache/invalidate`.
  - Validates internal API key.
  - Invalidates and eagerly re-fetches policy snapshot.

## 2.2 user-service Components
- `AdminRbacController`
  - `GET /api/v1/admin/rbac/policy`
  - `PUT /api/v1/admin/rbac/policy`
  - Protected with `@PreAuthorize("hasRole('SYSTEM_ADMIN')")`.

- `InternalRbacController`
  - `GET /api/internal/rbac/policy`
  - Uses `InternalApiKeyGuard` header validation.

- `RbacPolicyService`
  - Reads and maps persisted policy/rules to admin and gateway DTOs.
  - Normalizes/validates hierarchy, decisions, methods, roles.
  - Rebuilds ordered rules and bumps policy version.
  - Calls `GatewayRbacCacheInvalidationClient.invalidate()` post-update.

- `GatewayRbacCacheInvalidationClient`
  - Sends `POST` to gateway invalidate path.
  - Non-blocking behavior on failure (warns, does not roll back update).

## 2.3 Admin UI Components
- `RbacPolicyPage`
  - Loads current policy, drafts editable rules.
  - Client-side checks for mandatory fields.
  - Sends `PUT` payload preserving rule order.
  - Shows success toast for update + invalidation.

## 3. Data Model

## 3.1 Tables
- `rbac_policies`
  - `id uuid pk`
  - `policy_key varchar(64) unique not null`
  - `role_hierarchy text not null`
  - `default_decision varchar(8) not null`
  - `version bigint not null`
  - `updated_at timestamptz not null`

- `rbac_policy_rules`
  - `id uuid pk`
  - `policy_id uuid not null` -> FK `rbac_policies.id` ON DELETE CASCADE
  - `sort_order int not null`
  - `name varchar(120) not null`
  - `methods varchar(200) not null` (CSV in persistence)
  - `path_pattern varchar(255) not null`
  - `effect varchar(8) not null`
  - `allow_anonymous boolean not null`
  - `required_roles varchar(400)` (CSV in persistence)
  - Unique constraint: `(policy_id, sort_order)`

## 3.2 Entity Mapping
- `RbacPolicy` has `@OneToMany(orphanRemoval=true)` with `@OrderBy("sortOrder ASC")`.
- `RbacPolicyRule` has `@ManyToOne(policy)`.
- Version is incremented by service method (`bumpVersion`).

## 4. API Contracts

## 4.1 Admin API (user-service)
- `GET /api/v1/admin/rbac/policy`
  - Response: `RbacPolicyResponse`
  - Includes:
    - `policyKey`
    - `roleHierarchy`
    - `defaultDecision`
    - `version`
    - `updatedAt`
    - `availableRoles`
    - `rules[]`

- `PUT /api/v1/admin/rbac/policy`
  - Request: `RbacPolicyUpdateRequest`
    - `roleHierarchy` (required)
    - `defaultDecision` (`ALLOW|DENY`)
    - `rules[]` non-empty
  - Response: updated `RbacPolicyResponse`

## 4.2 Internal Policy Source API (user-service)
- `GET /api/internal/rbac/policy`
  - Header: `X-Internal-Api-Key`
  - Response: `GatewayRbacPolicyResponse`

## 4.3 Internal Cache API (gateway)
- `POST /internal/rbac/cache/invalidate`
  - Header: `X-Internal-Api-Key`
  - Response: `204 No Content`

## 5. Authorization Algorithm (Gateway)

Pseudocode:

```text
snapshot = policySnapshotProvider.currentPolicy()
compiled = compileIfVersionChanged(snapshot)

matched = false
granted = false

for rule in compiled.rules:
  if !rule.matches(method, path):
    continue

  matched = true

  if rule.effect == DENY:
    return DENY

  if rule.allowAnonymous:
    granted = true
    continue

  auth = resolveAuthentication()
  if auth is anonymous:
    continue

  if rule.requiredRoles is empty:
    granted = true
    continue

  effectiveRoles = hierarchyExpand(auth.roles)
  if intersects(effectiveRoles, rule.requiredRoles):
    granted = true

if matched:
  return granted ? ALLOW : DENY
else:
  return compiled.defaultDecision
```

Evaluation properties:
- Explicit DENY short-circuits.
- Multiple matching ALLOW rules are ORed.
- No match uses policy default.

## 6. Policy Refresh and Caching

`CachedRbacPolicySnapshotProvider` refresh triggers:
- `@PostConstruct` initial fetch.
- `currentPolicy()` on-demand check.
- `@Scheduled(fixedDelay=30s)`.
- explicit `invalidate()`.

Refresh decision:
- If invalidated -> refresh now.
- Else refresh when `fetchedAt + refreshInterval` elapsed.

Failure handling:
- Keep current snapshot on remote fetch error.
- Use fallback snapshot from local config if no remote data.

## 7. Policy Update Transaction Semantics (user-service)

Update flow (`RbacPolicyService.updatePolicy`):
1. Load or create policy by key.
2. Normalize role hierarchy and decision.
3. Build and validate rules:
   - Non-empty name/path/methods.
   - Normalize method and role casing.
   - Validate required roles exist via `RoleRepository`.
4. Clear existing rules and `saveAndFlush` (avoids unique key collision on `policy_id, sort_order`).
5. Apply new hierarchy/decision/rules.
6. Increment policy version.
7. `saveAndFlush`.
8. Trigger gateway cache invalidation.

Concurrency note:
- Current design relies on transactional overwrite + version increment.
- No optimistic lock column; last writer wins.

## 8. Security and Trust Boundaries

- External boundary: API Gateway.
- Internal trust: `X-Internal-Api-Key` between gateway and user-service.
- Admin mutation control:
  - user-service admin RBAC endpoints require `SYSTEM_ADMIN`.
- Internal endpoints are network-exposed but key-protected.

## 9. Configuration Matrix

## 9.1 user-service
- `app.rbac.policy-key` default `gateway-default`
- `app.rbac.internal-api-key`
- `app.rbac.gateway-base-url`
- `app.rbac.gateway-invalidate-path`

## 9.2 api-gateway
- `app.rbac.remote-policy.enabled`
- `app.rbac.remote-policy.source-url`
- `app.rbac.remote-policy.refresh-interval`
- `app.rbac.internal-api-key`
- Fallback static rules in `application.yaml`

## 10. Error Handling

- Gateway authorization failure: HTTP `403`.
- Invalid internal key:
  - user-service internal policy API: `401` (`UnauthorizedException`).
  - gateway cache invalidate API: `401` (`ResponseStatusException`).
- Invalid policy update payload: `400` (`IllegalArgumentException` / bean validation).
- Unknown role names in policy: `400`.

## 11. Observability

Current logs:
- RBAC decision logs at debug in gateway.
- Remote policy refresh success/failure logs.
- Cache invalidation failure warnings in user-service.

Recommended additions:
- Counter metrics:
  - `rbac_authorize_allow_total`
  - `rbac_authorize_deny_total`
  - `rbac_policy_refresh_success_total`
  - `rbac_policy_refresh_failure_total`
- Gauge:
  - `rbac_policy_version_current`
- Structured audit event on policy change:
  - actor, version before/after, rule count delta.

## 12. Test Design

## 12.1 Gateway
- Unit tests:
  - allow-anonymous pass/fail paths
  - DENY precedence
  - hierarchy inheritance
  - default decision behavior
- Integration:
  - remote policy pull and cache invalidation endpoint.

## 12.2 user-service
- Service tests:
  - role validation
  - version bump
  - sort order persistence
  - replace-rules flush behavior
- Controller tests:
  - `SYSTEM_ADMIN` guard
  - internal API key guard.

## 12.3 End-to-End
- Through gateway:
1. login as admin.
2. `GET /user/api/v1/admin/rbac/policy` -> 200.
3. `PUT /user/api/v1/admin/rbac/policy` update -> 200.
4. verify affected business endpoint auth behavior changes without gateway restart.

## 13. Gaps and Next Enhancements
- Full role CRUD and role-permission matrix lifecycle (beyond policy rule editing).
- Policy change audit trail table and rollback snapshots.
- Multi-tenant policy partitioning (`policy_key` per tenant/environment).
- Change approval workflow for high-risk policy updates.

