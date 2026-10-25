# MedTrain LMS — Backend Foundation

Version 5 · Oct 24, 2026 · @Bangalore MedTrain LLP

The permanent engineering rules every module follows: the request path, the layers, tenancy and scope, authorization, data, the read path, API conventions, jobs, integrations, change control, and the module template. Product modules sit on top of this and never invent their own pipeline.

- This document says how every module is built. It changes rarely and only by review.
- plan.md says what Phase 1 builds (tables, flows, build steps, required tests) and holds the detailed security mechanisms (revocation protocol, sessions, audit chain).
- system-design.md holds capacity targets, infrastructure, environments, deployment, video, real-time, observability, operations, cost, client architecture and the accepted residual risks.
- If two documents disagree, stop and resolve it in the documents before writing code.

Merged from the AIMR clinic backend foundation (v1) and the LMS Phase 1 plan. Section 15 lists what was taken from AIMR and what was deliberately left out.

Changes in v5: MongoDB variant kept but frozen and local-only (§10); responses only through allowlisted response DTOs (§7.3); prototype-pollution and command-injection rules (§6.5); signed job payloads (§8); inbound webhook verification and replay protection (§9); rules 25–29.

Changes in v4: three Redis instances (BullMQ needs noeviction); internal global tables for the outboxes and the account directory; PlatformUnitOfWork.forOrganization(); platform admin sessions in their own global table; trusted-proxy rule; Cache-Control: no-store by default; input bounds on every DTO; one SSRF-safe HTTP client; tokens never in URLs; rich text and CSV export sanitizing; partitions created by the ops pipeline; two-person rule for dangerous platform actions; change control for humans and AI coding agents (§16).

## 1. Architecture at a glance

- Modular monolith. One NestJS 11 app, stateless replicas, no sticky sessions. Module boundaries follow Postgres schemas. The same API serves web, Android, iOS and desktop clients.
- PostgreSQL (one DataSource, schemas per module, RLS on every tenant table) through a central PgBouncer service in transaction mode.
- Three Redis instances, each with one job:
- redis-security: authorization state only (versions, deny entries, permission sets, support sessions, MFA challenges, stream tickets). noeviction, durable, HA. Cluster mode (system-design C2).
- redis-queue: BullMQ queues and per-organization job semaphores. noeviction (BullMQ loses jobs under any other policy).
- redis-cache: rate limits, response cache, real-time pub/sub. allkeys-lru; losing it never affects correctness or security.
- Object storage (private, organization-prefixed, versioned, one KMS key per organization), a write-once audit bucket in a separate account, and KMS for per-organization data keys.
- Workers (BullMQ) run the same codebase with a worker entry point. Heavy work (AI, export, scan, email, bulk invitations) never runs inside an HTTP request.
- LLM gateway module: the only code that talks to AI providers.

## 2. Request path

The order is fixed. Every HTTP request goes through these steps; nothing is skipped per module and there is no flag that disables a step in development.

NestJS runs a request in this order: middleware → guards → interceptors (before the handler) → pipes → handler → interceptors (after the handler) → exception filters. The table follows that order. Because interceptors run before validation, they see the raw request: the cache and idempotency interceptors build their keys from canonical input (§6.1, §7.5) and never store a 4xx response, so invalid input can't pollute them.

| # | Step | Nest mechanism, where | What it does |
|---|---|---|---|
| 1 | HTTP in | src/main.ts | URI versioning (/api/v1), global prefix, helmet, CORS allowlist (never `*` in production), per-route body size limits, X-Request-Id assigned or accepted (UUID format only). Client IP is taken only from the trusted proxy chain (§11 rule 17). |
| 2 | Auth context | Middleware: src/common/auth/auth-context.middleware.ts | Verifies the EdDSA access token (alg allowlist, kid, iss, aud, typ), security-state sentinel, freshness lease, pending-security barrier, redis-security checks, resolves permissions, fills AsyncLocalStorage. Fails closed (401/503). Details: plan Part 3. |
| 3 | Authorization | Guard (APP_GUARD, registered first): src/permissions/authorization.guard.ts | Applies the handler's single declaration: @Public, @Permissions, @PlatformAdmin or @Authenticated. Denies undeclared handlers. Applies @NoSupportAccess and @RequireRecentAuth. |
| 4 | Rate limits | Guard (APP_GUARD, registered second): @nestjs/throttler, redis-cache store | Per IP and per account identifier on public routes; per organization and concurrency caps on authenticated routes (§6.4). Limiter outage fails open and logs. |
| 5 | Idempotency (opt-in) | Interceptor: idempotency.interceptor.ts | Creating POSTs with Idempotency-Key. Replays a stored response for the same canonical body (§7.5). |
| 6 | Response cache (opt-in) | Interceptor: @Cacheable() | GET handlers that declare it. Key built by the framework from context and canonical input (§6.1). |
| 7 | Validation | Pipe: global ValidationPipe (src/utils/validation-options.ts) | whitelist, forbidNonWhitelisted, transform, and the input bounds of §6.5. Failure → 422 problem+json with field errors. |
| 8 | Controller | `<module>/<module>.controller.ts` | Thin. One explicit handler per route, one authorization declaration, maps DTO → service call → response DTO. No business logic, no repository access. |
| 9 | Service | `<module>/<module>.service.ts` | Business rules, scope selection (own/any), calls AuthorizationService.assert() for resource checks, opens units of work, enqueues jobs after commit. |
| 10 | Persistence | TenantUnitOfWork / PlatformUnitOfWork + scoped repositories | set_config('app.organization_id'), scoped queries, RLS, request-scoped BatchLoader. |
| 11 | Response | Interceptor (after the handler) + exception filter ProblemDetailsFilter | Real HTTP status. Success { data, meta? }; errors application/problem+json (§7). Security and caching headers (§7.7). Unhandled errors go to Sentry (scrubbed). |

## 3. Layers and what each may do

| Layer | Location | May call | Must not |
|---|---|---|---|
| Edge | src/main.ts, src/app.module.ts | Nest bootstrap, global pipes, filters, guards | Contain per-module logic. |
| Context | src/common/context, src/common/auth | Redis security client, signing-key service | Touch tenant tables (except one version read on a cache miss). |
| Authorization | src/permissions | AuthContextService | Ask "is the caller role Y". Only "does the caller hold permission X (on resource R)". |
| Controller | `<module>/*.controller.ts` | Its own module's service | Call repositories, other modules' services, TypeORM, Redis, or external SDKs. |
| Service | `<module>/*.service.ts` | Its own repositories (ports), other modules' exported services, units of work, AuthContextService, AuthorizationService, queues, gateways | Import another module's entity or repository. Call an external service inside a unit of work. Read the organization from body, params or query. |
| Domain | `<module>/domain/*.ts` | Nothing infrastructural | Import TypeORM, Nest, Redis. |
| Repository port | `<module>/infrastructure/persistence/*.repository.ts` | — | Expose TypeORM types. |
| Repository adapter | `<module>/infrastructure/persistence/relational/**` | TypeORM, the unit-of-work manager | Exist outside this folder. Run without a unit-of-work manager. |
| Jobs | `<module>/jobs/*.processor.ts` | Services, with an explicit TenantContext | Receive an access token. Skip the organization/actor re-check. |
| Integrations | src/ai/gateway, src/storage, src/mail, src/common/http/safe-fetch.ts | Provider SDKs, outbound HTTP | Be bypassed: no other module imports a provider SDK or makes outbound HTTP calls. |

Enforced by ESLint no-restricted-imports / dependency-cruiser rules in CI: a module may import another module only through its index.ts public exports; typeorm is importable only under infrastructure/persistence/relational/; provider SDKs only under their integration module; axios, node-fetch, undici and global fetch only inside safe-fetch.ts.

## 4. Tenancy and scope

### 4.1 Rules

- organization_id is on every tenant-owned row, and every tenant table is under forced RLS.
- Global tables have no organization_id or carry it only as information, have no RLS, and are listed in src/common/tenancy/global-tables.ts. There are three kinds, and nothing else may be global:
- Reference and platform tables: organizations, organization_slug_aliases, platform_admins, platform_admin_sessions, platform_admin_recovery_codes, platform_admin_webauthn_credentials, platform_approvals, platform_idempotency_keys, statuses, catalog tables.
- Internal queue tables: tenancy.security_outbox, tenancy.email_outbox. Read across organizations only by dedicated workers; the API role has column-level grants only (plan Part 2).
- Directory tables: identity.account_directory (hashes only), read only by the find-organization job.
- After login the organization comes only from the token (AuthContextService) or a job's TenantContext. Never from Host, body, params or query.
- Units of work. Every tenant query runs inside one of:
- TenantUnitOfWork.run(ctx, fn): one short transaction, set_config('app.organization_id', $1, true), commit, release. API units of work stay under 5 seconds (lms_app timeouts). Jobs connect as lms_worker (60-second cap), process in batches and commit per batch; a job that spans organizations opens one unit of work per organization.
- PlatformUnitOfWork.run(fn): sets app.platform = on only. For global tables and the platform audit chain. Reachable only from @PlatformAdmin() services.
- PlatformUnitOfWork.forOrganization(orgId, fn): sets both app.platform = on and app.organization_id. The only way a platform admin action reads or writes a tenant table (provisioning, suspend, support sessions, data requests). Its audit rows go on that organization's chain with actor_type = platform_admin.
- A tenant unit of work can never set app.platform; the unit-of-work classes are the only code allowed to call set_config, and a lint rule enforces it.
- No unit of work calls external services.
- Isolation is enforced three times: scoped repository (code), composite tenant-aware foreign keys (database), RLS with FORCE (database). See plan Part 4.
- Out-of-scope rows return 404, never 403, so existence never leaks.

### 4.2 Scope modes

Every scoped repository method takes a scope. The default is organization; the others are opt-in and checked at runtime.

| Scope | Filter added (besides RLS) | Use | Guard rail |
|---|---|---|---|
| organization | organization_id = ctx.org AND deleted_at IS NULL | Organization-wide records (roles, users, courses) | Default. |
| owner | organization + `<owner column>` = ctx.userId | A learner's own attempts, progress, certificates, notes | The entity declares its owner column with @OwnerColumn('learner_id'); using owner on an entity without one throws. Never used without the organization filter. |
| catalog | Catalog items licensed to ctx.org (join on learning.catalog_licenses) | MedTrain-authored shared content | Read-only for organizations; writes only from @PlatformAdmin() handlers. |
| global | deleted_at IS NULL only | Global tables | Throws unless the table is on the global allowlist. |

Shared content (from AIMR's core_or_owned). MedTrain's master library lives in the global catalog schema. An organization sees an item only through a license row. Editing a licensed item creates an organization copy ("fork on edit") with source_catalog_item_id. RLS policies never use OR organization_id IS NULL.

## 5. Authorization

- Declarations. Exactly one of @Public(), @Permissions(...), @PlatformAdmin(), @Authenticated() on every handler. The route scanner fails startup and CI otherwise. @Authenticated() only for routes on src/permissions/authenticated-allowlist.ts.
- Permission strings live in src/permissions/permission.ts as a string union: `<resource>.<action>` or `<resource>.<action>.<own|any>`. Role assignment is data; permission strings are code.
- Own vs any. When a resource has personal rows, it gets a pair (attempt.read.own, attempt.read.any). The handler declares both in @Permissions(...); the service picks the scope with authContext.resolveScope('attempt.read'), which returns organization for .any, owner for .own, and throws 403 otherwise.
- Resource checks. Services that act on a specific resource call authorizationService.assert(permission, { type, id }). In Phase 1 it checks the organization-level permission only. In Phase 2 it also reads tenancy.role_assignments (course- or batch-scoped roles). Guards never change when that lands.
- No escalation. Role and user writes follow the four rules in plan Part 1, each with a test.
- No impersonation. Nobody, including platform admins and support sessions, can act as another user. Support sessions act as the platform admin with the session's permissions, and every action is attributed to that admin.
- Two-person rule. Platform actions that can destroy or exfiltrate a whole organization (delete, export, reset the last Org Admin's MFA, reset a platform admin's MFA) need approval from a second platform admin through platform_approvals (plan Part 5). An admin can never approve their own request.
- Never cache authorization state inside a replica. Permission sets and versions are read from redis-security on every request.

## 6. Read path and performance

### 6.1 Response cache

AIMR's cache, made tenant-safe. A cache hit never reaches Postgres, so RLS does not protect it; the key is the isolation boundary.

- Opt-in per handler: @Cacheable({ ttl, scope: 'organization' | 'owner' }). No decorator, no cache. Stored in redis-cache.
- The interceptor runs after the authorization guard and builds the key itself, from the context only: c:{organizationId}:{module}:{gen}:{roleId}:{roleVersion}[:{userId}:{userVersion}]:{handler}:{sha256(canonical params+query)} Callers cannot pass an organization or override the key.
- {gen} is the module's generation counter cache:gen:{organizationId}:{module}. The interceptor reads it before the handler reads any data; writes call cache.bump(module) after commit. A read racing a write therefore stores its result under the old generation, which is already unreachable. Never delete by pattern (KEYS/SCAN). If redis-cache evicts a generation counter, it restarts from a value that includes the current time, so it never returns to an older generation.
- Owner-scoped results include the user's version, so a lost enrollment never serves cached content.
- Resource-checked handlers (those that call AuthorizationService.assert() with a resource) may only use scope: 'owner'. A static check fails on @Cacheable({ scope: 'organization' }) for such handlers.
- Canonical keys. The interceptor hashes only the query parameters declared on the handler's query DTO, with keys sorted and values normalized. Only 200 responses are stored; 4xx and 5xx never are.
- Never cached: anything encrypted with an organization data key (MFA, prompts, transcripts, health data), auth and session responses, and responses inside a support session. A static check fails if @Cacheable is on a handler whose response DTO is marked @Sensitive().
- TTL default 60 s, maximum 10 minutes.
- This is a server-side cache only. HTTP responses still carry Cache-Control: no-store (§7.7), so no browser, app or proxy keeps a copy.

### 6.2 Batching (no N+1)

- BatchLoader (DataLoader pattern) is request-scoped: created per request in AsyncLocalStorage (per job in workers), never shared across requests, so one batch never mixes organizations.
- Loads run inside the request's unit of work and the same scope as the caller.
- A test helper counts queries per request; list endpoints must stay within a fixed query budget (default 5).

### 6.3 Queries, indexes and search

- Every index on a tenant table starts with organization_id. Text search uses pg_trgm GIN indexes combined with btree_gin so the index can start with organization_id.
- High-volume tables (progress and xAPI events, AI usage, audit logs) are partitioned by time from their first migration (§10).
- List and reporting reads may use the read replica (same RLS) through TenantUnitOfWork.runReadOnly(). Never read-your-own-write from the replica.
- TypeORM query logging stays off. Slow queries are visible through pg_stat_statements, OpenTelemetry Postgres spans and a slow-query alert.

### 6.4 Fairness and load protection

One large or abusive organization, or a traffic spike, must not use up shared capacity.

- API: a per-organization request rate limit sized by plan, and concurrency caps on expensive endpoints (exports, search, bulk operations, later AI), in redis-cache. Over the limit → 429 org_rate_limited with Retry-After. Limits are configuration, not code.
- Shared IPs are normal. Mobile carriers in India put many users behind one IP (carrier-grade NAT), and hospitals use one outbound IP. Per-IP limits are therefore generous and only stop floods; account-level protection keys on organization plus account identifier.
- Jobs: each processor takes a per-organization semaphore in redis-queue before working; over the limit, the job is delayed and re-queued. Open-source BullMQ has no per-group concurrency, so this is done in our processor base class.
- Database: database time per organization is aggregated from OpenTelemetry spans; an alert fires when one organization exceeds its share.
- Load shedding: each replica caps in-flight requests (configuration, sized in system-design C5). Requests over the cap, or waiting longer than 2 seconds for a database connection, get 503 overloaded with Retry-After instead of queuing until they time out.
- Password hashing: bcrypt runs in a bounded worker pool per replica, so a login flood cannot block the event loop for everyone.
- AI (Phase 4): budget reservation per organization (plan "AI invariants") plus a per-user rate limit.

### 6.5 Input bounds

Every request is bounded before it reaches business logic.

- Every string field in every DTO has @MaxLength (default 1,000 unless the field says otherwise; emails 254; names 100; rich text 200 KB). Every array has @ArrayMaxSize (default 100). Nested objects have a fixed maximum depth (default 5).
- Body size limits per route: 100 KB default, larger only where declared (rich text, bulk imports go through file upload, never through the JSON body).
- No user-supplied regular expressions, and no validator that runs a regex with catastrophic backtracking on user input.
- Prototype pollution: the JSON body parser rejects any object key named `__proto__`, constructor or prototype (422), and code never merges request data into objects with deep-merge helpers; DTOs are created by class-transformer from whitelisted fields only.
- No shell, no eval: code never runs a shell with user input. External programs (media, image or scan tools) are started with execFile/spawn and an argument array, shell: false, inside isolated workers. eval, new Function and dynamic require of user-controlled paths are forbidden. Lint rules enforce both.
- No unsafe deserialization: only JSON is accepted from clients; no YAML, XML or serialized class formats are parsed from user input.
- A static check fails on a DTO string field without @MaxLength or an array without @ArrayMaxSize.

## 7. API conventions

### 7.1 URLs and methods

- `/api/v1/<plural-resource>[/{id}[/<sub-resource>]]`, kebab-case paths, camelCase JSON fields, UUIDv7 ids.
- GET reads, POST creates or runs a command (/files/{id}/complete), PATCH partial update, DELETE removes. No PUT.
- Self-service under /api/v1/me/...; platform admin under /api/v1/admin/... (served only on the admin hostname).
- Tokens and secrets never appear in URLs (paths or query strings). Email links put their token in the URL fragment (#token=…), which browsers never send to a server; the landing page posts it in the body. Streams use short-lived stream tickets (§7.7).

### 7.2 Status codes

The HTTP status always tells the truth. Never return 200 with an error body.

| Status | When |
|---|---|
| 200 | Read or update succeeded |
| 201 | Created (with Location) |
| 202 | Accepted for background work, or pending_enforcement |
| 204 | Deleted or no content |
| 400 | Malformed request, invalid cursor, generic token failures where the plan requires them |
| 401 | Not authenticated, token invalid or revoked, reauth_required, mfa_required |
| 403 | Authenticated but lacks the permission |
| 404 | Not found or out of scope |
| 409 | Conflict: already_exists, link_required, seat_limit_reached, last_org_admin, storage_limit_reached, email_taken, duplicate refresh |
| 413 | Body larger than the route allows |
| 422 | Validation failed, idempotency_key_reused |
| 426 | client_upgrade_required (client below the minimum supported version) |
| 429 | Rate limited (with Retry-After) |
| 500 | Unhandled error (generic body; details only in logs and Sentry) |
| 503 | Security state unavailable (Redis down, lease expired, sentinel missing) or overloaded, with Retry-After |

### 7.3 Bodies

- Success: { "data": ... }. Lists: { "data": [...], "meta": { "nextCursor": "..." | null } }. The request id is in the X-Request-Id response header.
- Responses only through response DTOs. Every handler returns a response DTO whose fields are listed explicitly (@Expose() with excludeExtraneousValues: true). Entities, domain objects and raw query rows are never returned. Fields such as password hashes, MFA secrets, authz_version, internal flags and other organizations' ids can therefore never leak by accident. A test compares every response with its DTO (plan test 78).
- Error: RFC 9457 application/problem+json:
```json
  { "type": "https://errors.med-train.com/seat_limit_reached", "title": "Seat limit reached",
    "status": 409, "code": "seat_limit_reached", "detail": "…", "requestId": "…",
    "errors": [{ "field": "email", "code": "invalid_email" }] }
```
- code is stable and machine-readable; clients branch on code, never on title or detail, and show their own localized message for each code. Every code lives in src/common/errors/error-codes.ts with its status. Adding a code is a reviewed change.
- Production errors never include stack traces, SQL, entity names or internal ids of other organizations.

### 7.4 Pagination

- Every list is cursor-paginated. ?limit= default 20, maximum 100. ?cursor= is opaque.
- Cursors are base64url JSON { sort keys, organizationId, expiresAt } signed with HMAC. The signing secret has a kid so it can rotate: the current and previous secrets both verify. A tampered, expired, or other-organization cursor returns 400 invalid_cursor.
- Sort is always stable (sort key + id).

### 7.5 Idempotency

- Every POST that creates something accepts Idempotency-Key (required on provisioning, invitations, file reservation, offline submissions from mobile, and later payments and AI generation).
- Stored in tenancy.idempotency_keys (organization_id, actor_id, key, request_hash, response_status, response_body, expires_at) for 24 hours. Same key and same body → the stored response. Same key and different body → 422 idempotency_key_reused.
- request_hash is SHA-256 of the canonical JSON body (keys sorted, whitespace removed), because the interceptor runs before validation.
- Only 2xx responses are stored. A 4xx is returned but not stored, so the client can fix the request and retry with the same key.
- Sensitive responses are never stored. When the response DTO is @Sensitive(), only the status and the created resource's id are kept, and a replay returns that status with { data: { id } }. Routes that return secrets (MFA enrollment, recovery codes) do not accept Idempotency-Key at all.

### 7.6 Clients and versions

- Dates are ISO 8601 UTC strings. Money is integer minor units plus currency code. Organizations store a time zone setting; due dates are stored in UTC and shown in that zone by clients.
- Clients send X-Client-Version (`<platform>/<semver>`). Below the published minimum for that platform, the API returns 426 client_upgrade_required.
- OpenAPI from @nestjs/swagger decorators on every DTO; disabled in production.
- A breaking change goes to a new version (/api/v2); the old version gets a Deprecation and Sunset header for at least 6 months, longer than the oldest mobile app still allowed.

### 7.7 Response headers and streams

- Caching: every response from the API carries Cache-Control: no-store by default, plus Pragma: no-cache. Only explicitly public resources (the JWKS document, the minimum-client-version document) may be publicly cacheable. CloudFront in front of the API never caches API responses (system-design C2).
- Security headers (helmet): Strict-Transport-Security (1 year, includeSubDomains), X-Content-Type-Options: nosniff, Referrer-Policy: no-referrer, X-Frame-Options: DENY, and a restrictive Content-Security-Policy on any HTML the API serves.
- Streams (SSE, later WebSocket): clients never put an access token in a URL. Native apps and modern browsers send Authorization on a fetch-based stream. Where a client must use a URL (browser EventSource, WebSocket), it first calls POST /api/v1/me/stream-tickets and receives a single-use ticket (random, stored hashed in redis-security, valid 30 seconds, bound to the session and the stream path), consumed atomically on connect. Long-lived streams then follow the re-check rule in plan Part 3.

## 8. Jobs and events

- One queue system: BullMQ on redis-queue (noeviction, startup fails otherwise). Queue names use a hash tag ({q:email}) so each queue's keys stay in one cluster slot. AIMR's second system (Sidequest) is not used.
- Signed jobs. Every job payload is signed with HMAC-SHA-256 (JOB_SIGNING_SECRET, with a kid for rotation) when it is enqueued. Processors verify the signature before anything else and reject, alert and dead-letter a job with a missing or invalid signature, so a job written directly into redis-queue cannot run.
- Jobs that must happen if and only if a transaction commits are written to an outbox table in that transaction (email_outbox, security_outbox, later job_outbox); a relay moves them to BullMQ. Outbox workers claim with FOR UPDATE SKIP LOCKED. Postgres is the source of truth: if redis-queue loses recent jobs in a failover, the relay re-enqueues anything not marked done.
- Every job payload carries TenantContext { organizationId, actorUserId, actorType } and ids only, never tokens, passwords or large data.
- Before working, a processor re-checks that the organization is active and the actor is not locked or deleted.
- Handlers are idempotent (safe to run twice), have explicit retry and backoff, and send exhausted jobs to a dead-letter queue with an alert.
- Request handlers enqueue and return (202 with a job or resource id). Clients poll or subscribe for the result.

## 9. Integrations

- LLM gateway (src/ai/gateway): the only code calling AI providers. Budget reservation, model allowlist, redaction, metering, prompt versions. See plan "AI invariants".
- Storage (src/storage): presigned upload and download only; keys start with the organization id; each organization's objects are encrypted with that organization's KMS key; quota reservations (plan Part 7).
- Mail (src/mail): only through email_outbox. Every message carries the organization id and outbox id as provider message tags, so bounce webhooks know which row to update.
- Outbound HTTP (src/common/http/safe-fetch.ts): the only way any code fetches a URL (OIDC discovery, breach-check API, webhooks, later AI tools and imports). It resolves DNS once and connects to that address, refuses private, loopback, link-local (including 169.254.0.0/16 cloud metadata) and multicast ranges for IPv4 and IPv6, refuses redirects to such addresses, enforces HTTPS, a timeout, and a response size limit, and uses a per-purpose host allowlist where one exists.
- Inbound webhooks (email provider now, payment provider and integrations later): every webhook route is @Public() but verifies the provider's signature with a constant-time comparison, rejects events whose timestamp is outside the provider's tolerance window (default 5 minutes), stores the provider's event or message id with a unique constraint so a replayed event is processed once, and handles each event idempotently. Webhook routes accept only the provider's published IP ranges where the provider publishes them.
- Sentry (SentryExceptionFilter, from AIMR): beforeSend removes request bodies, headers, cookies, tokens, emails and SQL parameters; tags only requestId, organizationId, subjectType; hosted in an approved region or self-hosted. Only 5xx and unexpected errors are sent.

## 10. Data rules

- One DataSource. All entities in one TypeORM DataSource; each declares its schema: @Entity({ schema: 'learning', name: 'courses' }). Never rely on search_path. AIMR's per-module named connections are not used.
- Postgres is the only supported database. The boilerplate's MongoDB (document) variant stays in the repository, frozen: selected only by DATABASE_TYPE=mongodb, allowed only in local development, never extended. It has none of the security in these documents (no RLS, no tenant-aware foreign keys, no tenant repository, no audit chain), so the app refuses to start with it in any other environment. Details and the startup check: plan Part 1.
- Schemas: identity, tenancy, catalog, storage, learning, progress, assessment, ai.
- Standard columns on every tenant table: id uuid (UUIDv7, generated in application code), organization_id uuid not null, created_at, updated_at, deleted_at (soft delete where the entity can be restored), plus version int where optimistic locking or cache versions apply.
- Constraints in the same migration that creates the table: UNIQUE (organization_id, id), composite foreign keys to every tenant parent, ENABLE and FORCE ROW LEVEL SECURITY with the standard policy, and indexes starting with organization_id.
- Partitioned tables (high-volume events and logs): Postgres requires the partition key in every unique key, so they use UNIQUE (organization_id, id, created_at) (and the same for any other unique key). They are leaf tables: nothing references them by foreign key. Partitions are created at least 3 months ahead by the ops pipeline as lms_migrator (creating or dropping a partition needs the table owner), an alert fires when fewer than 2 future months exist, and a default partition catches stray rows and alerts if it is ever non-empty.
- Null-organization rows in a tenant table exist only in the audit tables (platform and anonymous chains), with explicit per-role policies (plan, Audit log). Nowhere else.
- Never in tenant schemas: views without security_invoker = true, SECURITY DEFINER functions, synchronize: true outside local tests.
- Migrations: generated with npm run migration:generate, reviewed by hand, run by lms_migrator in the deploy pipeline, never at app startup. Each must be backwards compatible with the previous app version (expand, migrate, contract). Migrations set lock_timeout = 3s and retry; indexes on existing large tables use CREATE INDEX CONCURRENTLY in their own migration.
- Entities come from the project generators (generate:resource:relational, add:property:to-relational), then are adapted to the rules above. Never hand-write an entity from scratch.
- Encryption: columns holding secrets or sensitive personal or health data are encrypted with the organization's data key through DataKeyService and marked @Sensitive().
- User content is sanitized on write. Rich text is stored as a structured document (JSON blocks) and validated against an allowlist schema; any HTML is sanitized server-side with an allowlist (no script, no event handlers, no javascript: URLs, no inline styles beyond an allowlist). Clients never render stored content as raw HTML.
- Exports are escaped. CSV and spreadsheet exports prefix any cell starting with =, +, -, @, tab or carriage return with a single quote, so a value can never run as a formula. File names in Content-Disposition are RFC 6266 encoded.

## 11. Security rules (summary)

The mechanisms are in plan Parts 3–6. Every change must keep these true:

- No tenant data access outside a unit of work and a scoped repository.
- No handler without exactly one authorization declaration.
- The organization never comes from client input after login.
- Out-of-scope reads return 404.
- No authorization state cached in process memory.
- Security changes go through AuthzVersionService (outbox + versioned Redis write), never a direct DEL.
- Every security state change writes its audit row in the same unit of work.
- Never log, audit, cache or send to Sentry: tokens, passwords, hashes, MFA or reset codes, prompts, request bodies or raw emails.
- Never call external services inside a unit of work.
- Credential, MFA, session, token, export and deletion routes carry @NoSupportAccess() and, where the plan says so, @RequireRecentAuth().
- Secrets come only from the secret manager or env, validated at startup; startup fails on missing or weak values.
- Every token sent by email or link is single-use: random, stored hashed with purpose and expiry, consumed with one conditional update. Never a signed JWT that stays valid after use.
- Account-identifier changes (email, linked identities) need recent re-authentication, confirm the new value, notify the old one, keep the account status, and raise authz_version.
- Web clients keep tokens in memory only (never localStorage, sessionStorage or IndexedDB) until the httpOnly refresh cookie ships; mobile and desktop use the OS secure store.
- /health/ready never depends on shared dependencies or the lease; those fail per request with 503 and raise alerts.
- Every outbound HTTP request goes through safe-fetch.ts.
- The client IP is read only from the trusted proxy chain: Express trust proxy is set to the exact number of proxy hops (CloudFront, then the load balancer), never true. A client-supplied X-Forwarded-For can never change the IP used for rate limits or audit.
- API responses are Cache-Control: no-store unless explicitly public.
- Tokens never appear in URLs; email links use the fragment, streams use single-use tickets.
- User content is sanitized on write and escaped on export (§10).
- No impersonation of users; support sessions act as the admin.
- Platform actions that delete or export a whole organization, or reset an MFA factor of the last Org Admin or of a platform admin, need a second platform admin's approval.
- Every DTO field is bounded (§6.5).
- Only EdDSA is accepted for access tokens and only HS256 for refresh tokens; the algorithm is fixed in code, never read from the token header.
- Responses go only through allowlisted response DTOs; entities are never returned.
- No shell execution with user input, no eval, no unsafe deserialization; JSON bodies with prototype keys are rejected.
- Every job payload is signed and verified before it runs.
- Every inbound webhook is signature-verified, time-limited and replay-protected.
- The app runs on Postgres only outside local development; the frozen MongoDB variant is never extended.

## 12. Module template

Every product module looks like this. A hygen template (npm run generate:module, added in Phase 1a step 3) creates it, including the cross-tenant test.

```text
src/<module>/
  index.ts                         public exports: module, service interface, DTO types
  <module>.module.ts
  <module>.controller.ts           thin; one declaration per handler
  <module>.service.ts              business rules, scope selection, units of work
  domain/<entity>.ts               plain domain types
  dto/                             request and response DTOs (class-validator + swagger, bounded)
  infrastructure/persistence/
    <entity>.repository.ts         port (abstract class)
    relational/
      entities/<entity>.entity.ts  TypeORM entity with schema, organization_id, owner column
      repositories/<entity>.repository.ts   extends TenantScopedRepository
      mappers/<entity>.mapper.ts
  jobs/                            BullMQ processors (TenantContext in, re-check first)
  <module>.permissions.ts          this module's permission strings (merged into the union)
  test/<module>.cross-tenant.e2e-spec.ts    required
  test/<module>.authorization.e2e-spec.ts   required
```

## 13. Testing rules

- Unit tests for services and domain logic, with repository ports mocked.
- E2E tests against real PostgreSQL (with the real roles and RLS) and all three Redis instances, with redis-security in cluster mode (docker compose), never SQLite or mocks for persistence.
- Required for every module: the cross-tenant spec (org A cannot read, list, update or delete org B rows: 404), the authorization spec (each handler: 401 without a token, 403 without the permission, success with it), owner-scope tests where .own permissions exist, and a query budget assertion on list endpoints.
- Required for every concurrency rule (quotas, seats, last admin, refresh, token consumption): a test that runs the operation in parallel and asserts the invariant.
- Security regression suite: input-bound fuzzing of every DTO (oversized strings, arrays, depth), SSRF targets against safe-fetch.ts, token-in-URL and header checks on every route, and from Phase 4 a prompt-injection corpus against the LLM gateway.
- Tests are never weakened, skipped or deleted to make a build pass. A required test that cannot run is reported as a failure.
- Plan "Required tests" lists the Phase 1 suite; it is the minimum, not the whole suite.

## 14. Definition of done for any change

- npm run build, npm run lint, npm test and the relevant e2e specs pass.
- New handlers have one authorization declaration; the route scanner passes.
- New tenant tables pass the migration check (organization_id, composite FKs, RLS, index prefix, partition-key uniqueness).
- New lists use cursor pagination; new creates accept Idempotency-Key where §7.5 requires it.
- New error cases use a code from error-codes.ts.
- The cross-tenant and authorization specs cover every new route.
- No new log line or audit field contains a secret or personal data.
- Work that can exceed the API limits runs as a job in batches under lms_worker.
- New expensive endpoints have a per-organization concurrency cap (§6.4).
- New DTO fields are bounded (§6.5); new outbound calls use safe-fetch.ts.
- Security-owned paths changed only with a security owner's approval (§16).
- Docs updated when a rule or flow changed (this file, plan.md or system-design.md, by review).

## 15. What came from AIMR

| Taken | Changed on the way |
|---|---|
| Fixed request path, shared base pieces | Base pieces are the tenant repository, unit of work and interceptors; no generic execute_method dispatcher (it hides authorization from the route scanner). |
| Auth middleware with AsyncLocalStorage | Global for every route, fail closed, EdDSA via our own JWKS, plus revocation. |
| JWKS / issuer / audience verification | Used for our own tokens and for SSO login (Phase 5). External OIDC tokens are never accepted as API tokens. |
| ValidationPipe with 422 | Plus forbidNonWhitelisted and input bounds. |
| Scope modes | Renamed to organization / owner / catalog / global; bare user scope dropped (always combined with the organization). |
| core_or_owned shared master data | Global catalog schema plus license rows and fork-on-edit. |
| Response cache | Opt-in, tenant-safe keys, generation counters, never for sensitive data. |
| BatchLoader | Request-scoped. |
| Trigram search and partitions | Indexes lead with organization_id via btree_gin. |
| BullMQ for heavy work | Only queue system, on its own noeviction Redis; outboxes for transactional enqueue. |
| Sentry exception filter | Scrubbed, region-approved. |
| 404 for out-of-scope rows | Same. |
| Postgres schemas per module | One DataSource instead of named connections. |

| Not taken | Why |
|---|---|
| YAML route security | Drifts from code and can't be checked by the build; decorators plus route scanner instead. |
| Opt-in auth middleware list, ConditionalAuthGuard letting requests through | A forgotten controller would be public. Deny by default. |
| Open CORS | Production allowlist only. |
| {status_code, msg, data} envelope with HTTP 200 for errors and custom codes 1000/1500 | Breaks client refresh logic, monitoring, retries and the plan's 202/409/503 semantics. Real status + problem+json instead. |
| Named connections per module | Breaks atomic cross-schema changes and exhausts connections. |
| Sidequest as a second queue | One queue model. |

## 16. Change control (humans and AI coding agents)

Security does not depend on whoever writes the code being careful. The repository enforces it.

- Branch protection: main accepts changes only through pull requests with passing CI and at least one human approval. Nobody, human or agent, pushes to main directly. Force pushes are disabled.
- CODEOWNERS: changes to these paths need approval from a named security owner: `src/common/auth/**`, `src/common/tenancy/**`, `src/permissions/**`, `src/security/**`, `src/auth/**`, `src/audit/**`, `src/redis/**`, src/common/http/safe-fetch.ts, `src/database/migrations/**`, `.github/**`, CLAUDE.md, `.claude/**`, package.json and the lockfile.
- CI is the enforcer, not the author. Every rule in this document that can be checked is checked in CI: route scanner, migration check, lint boundaries, input-bound check, cache checks, set_config restriction, secret scan, dependency scan, the full e2e suite. An author, human or AI, cannot pass review by claiming a rule was followed.
- AI coding agents (Claude Code and others) work under CLAUDE.md: they run with development credentials only, have no access to staging or production secrets, never run migrations outside local and CI databases, never add @Public, entries to authenticated-allowlist.ts, new dependencies, or changes to security-owned paths without saying so explicitly in their report, and never weaken, skip or delete a test.
- Dependencies: new runtime dependencies need review; the lockfile is committed; CI installs with npm ci; install scripts of new dependencies are reviewed; images are built in CI only, scanned, and signed, and production runs only signed images.
