Multi-tenant AI LMS — Phase 1 Security Foundation Plan

Oct 21, 2026 · @Bangalore MedTrain LLP · Revised Oct 25, 2026 (v7)

This plan says what Phase 1 builds. The permanent rules for how every module is built are in foundation.md. Infrastructure, capacity, video, real-time, operations, client architecture and accepted residual risks are in system-design.md. Read all three before coding.

Revision notes v7 (Oct 25, 2026)
Build order reordered so every step uses only what earlier steps built: Redis clients, signed jobs, the email outbox and the audit chain come before identity; sessions come with login; the sentinel check comes with the rebuild job; bulk invitations move to Phase 1b with storage.files. Phase 1a is steps 1–17, Phase 1b steps 18–23.
User and platform admin session ids are UUIDv7 like every other id.
Failed-login counters live in redis-cache, keyed by an HMAC of organization and email, and apply equally to unknown accounts, so the progressive delay never reveals which accounts exist.
Passwords are limited to 72 UTF-8 bytes, because bcrypt ignores the rest.
Media cookies are scoped to one course, not the whole organization.
Required tests are numbered.
Revision notes v6 (Oct 24, 2026)
DATABASE_TYPE selects the database; postgres is the only supported value outside local development, and the app refuses to start otherwise. New code is Postgres only (Part 1).
Payment invariants added (binding when billing or paid courses are built).
LMS content invariants added: course access, media URLs, signed certificates.
Response DTO allowlists, no shell execution with user input, prototype-pollution protection, signed job payloads, inbound webhook replay protection, no secrets in mobile app binaries (foundation v5, system-design v3).
Tests 77–83 added.
Revision notes v5 (Oct 24, 2026) — final review

Flows that run without an active organization, and infrastructure conflicts:

security_outbox and email_outbox become internal global tables with column-level grants; the pending-security watcher reads them through its own role, lms_security_watcher. Under forced RLS it would have seen no rows.
Platform admin sessions move to identity.platform_admin_sessions (global). identity.sessions holds user sessions only, with organization_id NOT NULL. Refresh tokens carry the organization so refresh can open the right unit of work.
PlatformUnitOfWork.forOrganization() sets both the platform flag and the organization, so provisioning, suspend, support sessions and data requests can write tenant tables under RLS.
Organization discovery reads a new hash-only global table, identity.account_directory.
Redis splits into three: redis-security, redis-queue (BullMQ, noeviction), redis-cache (eviction allowed).
redis-security is cluster mode from day one (MemoryDB): the middleware reads keys per node instead of one cross-slot pipeline.
A security-state sentinel (sec:epoch) protects against Redis data loss: without it, authenticated requests get 503 until deny keys are rebuilt from Postgres.
One KMS key per organization for its stored files, multi-Region for disaster recovery, so crypto-shredding actually covers objects and backups.
Audit fixes: the (chain_id, seq, created_at) key is no longer called a guard; partitions are created and dropped only by the ops pipeline; lms_audit_writer has no SELECT on audit logs.
PgBouncer runs as one central service.

New security items from the final pass: MFA challenge and attempt limits, WebAuthn credentials table, account recovery for lost MFA, two-person rule for destructive platform actions (platform_approvals), user and Org Admin security notifications, DPDP consent records (user_consents), bulk invitations, progressive login delays, CAPTCHA thresholds, trusted-proxy IP handling, no-store responses, tokens never in URLs (email links in the fragment, stream tickets), SSRF-safe outbound HTTP, input bounds, rich text and CSV sanitizing, file name sanitizing and image metadata stripping, short-lived media cookies, algorithm pinning and clock skew for tokens, graceful shutdown, AI hardening (tools, exfiltration, injection test corpus) and change control for AI coding agents (foundation §16). Tests 53–76 added.

Earlier revisions: v2 moved to EdDSA tokens, split Redis, added per-organization data keys, MFA, re-auth, AI invariants and the 1a/1b split. v3 merged the AIMR foundation (foundation.md, problem+json, one DataSource, scope modes, catalog schema, idempotency, tenant-safe cache). v4 fixed partition uniqueness, moved the watcher into 1a, added audit RLS policies, the real NestJS order, lms_worker, single-use email tokens, change of email, local-only readiness, per-organization fairness, the web refresh cookie, slug aliases and email deliverability.

Goal and what stays

Adapt this NestJS boilerplate (brocoders nestjs-boilerplate) into the auth layer of a multi-tenant AI LMS product. Any number of customer organizations (hospitals, colleges, companies) sign up and run their own isolated workspace, from web, Android, iOS and desktop clients through one API. Security comes first: every tenant boundary is enforced by code and by the database, not by developer memory. Login, refresh and sessions are issued by src/auth/auth.service.ts. Permissions live in code and on handlers, never in YAML.

Account model. One account per organization. A user account belongs to exactly one organization. The same email can exist in many organizations as completely separate accounts, each with its own password, sessions and role. The organization owns its accounts: it can create, lock or delete them without affecting any other organization. MedTrain team members who run the product are platform admins, stored separately, with their own login.

Keep these from the boilerplate, adapted to the account model above:

Email login, social login (Google, Facebook, Apple), confirm, forgot/reset, logout and refresh in src/auth/auth.controller.ts. Each now takes the organization at login time.
Refresh still rotates the session hash.
No Postgres query on the authenticated request path, except one read on a permission or version cache miss. redis-security is the only shared check. Rate limits live in redis-cache.

Work step by step. After each numbered step, run the build, lint and tests, and stop to report what changed before moving on.

Do not add a generic base repository, entity, service or controller (the boilerplate's empty src/common/base/ files are already removed). A generic base repository would bypass TenantScopedRepository; tenant data access goes only through Part 4.

Scale invariants (do not build the AI product in this task)

This slice is auth and tenancy only. The product around it is a multi-tenant AI LMS (courses, enrollments, files, chat, embeddings, generation, grading). These rules are binding so that work can be added later without a rewrite.

Shard key. organization_id is on every tenant-owned row, including users and user sessions. Only the global tables in foundation §4.1 have none. After login the access token carries the organization, so any API replica can serve any tenant with no sticky sessions.
Index rule. Every index on a tenant table starts with organization_id (including the primary-key lookup index (organization_id, id)), so RLS filters never scan across tenants.
Organization at login only. The login, register, forgot and reset requests carry the organization slug in the body. After login the organization comes only from the token, never from the Host header, body, params or query.
Ids. Organization, OrganizationRole, User, PlatformAdmin, AuditLog, user and platform admin sessions, and every later tenant entity use UUIDv7 generated in application code. UUIDv7 reveals its creation time, which is accepted (system-design C13). A session id identifies a session (it appears in tokens and in /me/sessions); it never authenticates one, which only the refresh token's hash does.
Identity is not email. A user is identified by users.id. External logins map through identity.user_identities (organization_id, provider, external_subject), and are never linked by email alone.
Request vs worker. AsyncLocalStorage exists only for the HTTP request. Jobs receive an explicit TenantContext: { organizationId, actorUserId, actorType }. Workers never receive an access token. Before doing any work, a job re-checks that the organization is active and the actor is not locked or deleted; a job queued before a suspension never runs after it.
AI data. Embeddings, object-storage keys, prompts, and model logs are keyed by organization_id and protected with the organization's keys (Part 6).
Postgres, with RLS. Isolation is enforced by the tenant repository, tenant-aware foreign keys and RLS (Part 4).
Lists. Every list endpoint is cursor-paginated.
Permission strings stay in code. Role assignment is data. Every protected handler declares its authorization explicitly (Part 3).
No authorization state cached inside a replica. Permission sets, versions and support sessions are read from redis-security on every request. The only in-memory security state is the pending-security set, and it is protected by the freshness lease.
Revocation guarantee, stated precisely. After a security change commits, every replica either enforces it or refuses all authenticated requests. Normally enforcement is immediate (the Redis write succeeds before the API confirms). If Redis refuses the write, every replica that is still serving enforces it within the freshness lease (default 2 seconds, SECURITY_LEASE_MS), because a replica whose pending-security set is older than the lease stops serving (Part 5). If redis-security loses data, the missing sentinel makes every replica refuse authenticated requests until deny state is rebuilt (Part 5). There is no state in which a replica serves a revoked token beyond the lease.
Availability trade-off, accepted. Because the lease depends on Postgres, a Postgres failover (typically 30–120 s on managed services) makes every replica refuse authenticated requests for its duration. Most authenticated requests need Postgres anyway, so this is accepted. The lease duration is configurable and lease expiry is alerted.
Three Redis instances (foundation §1).
redis-security: versions, deny entries, permission sets, support sessions, MFA challenges, stream tickets, and the sec:epoch sentinel. Security state, not a cache you can lose: noeviction, durable, HA, memory alerts well before the limit. Nothing else is stored here. It runs in cluster mode (MemoryDB), so every multi-key read is split per node and every Lua script touches one key.
redis-queue: BullMQ and job semaphores. noeviction; startup fails otherwise.
redis-cache: rate limits, response cache, real-time pub/sub. Eviction allowed. Filling it cannot affect authorization or jobs.
Token signing. Access tokens are signed with Ed25519 (EdDSA, via the jose library) and carry kid, iss, aud, iat, nbf, exp. Verification accepts only EdDSA (never the alg from the header), allows 30 seconds of clock skew, and rejects unknown kid. Public keys are published at /.well-known/jwks.json. Refresh tokens use HS256 with their own secret, are verified only by the auth module, and carry sub, subType, sessionId, organizationId (users only) and typ: "refresh".
Cross-platform clients. One API serves web, Android, iOS, and desktop. Refresh logic is transport-neutral (body for apps, httpOnly cookie for web from step 20). Client rules are in system-design C12.
One DataSource, one queue. All entities live in one TypeORM DataSource with per-entity schemas. BullMQ on redis-queue is the only job system.
Honest HTTP. The HTTP status always reflects the outcome; errors are problem+json with a stable code (foundation §7).
Ownership inside an organization. Personal rows (attempts, progress, certificates) declare an owner column and are read with owner scope unless the caller holds the .any permission (foundation §4.2, §5).
Shared content. MedTrain's library lives in the global catalog schema and reaches an organization only through a license row (foundation §4.2).
Part 1 — Database policy: PostgreSQL only

The product runs on PostgreSQL through TypeORM. Every security mechanism in this plan (RLS, tenant-aware foreign keys, the tenant repository, units of work, the audit chain, revocation) exists only for Postgres.

MongoDB variant: kept, frozen, local only. The boilerplate's MongoDB (document) variant stays in the repository, frozen: selected only by DATABASE_TYPE=mongodb, allowed only in local development, never extended. It has none of the security in these documents (no RLS, no tenant-aware foreign keys, no tenant repository, no audit chain), so the app refuses to start with it in any other environment.

DATABASE_TYPE is validated at startup. Allowed values: postgres everywhere; mongodb only when NODE_ENV=development and ALLOW_DOCUMENT_DB=true. Any other combination, and any mongodb value in staging or production, stops the app with a clear error.
When the app runs with mongodb, it logs a warning at startup and every hour that tenant isolation is not enforced.
New modules, tables and properties are generated with the relational generator only (generate:resource:relational, add:property:to-relational), never all-db or document. Nobody, human or agent, edits or extends files under document/ folders.
The document code will fall behind as Phase 1 adds tables (organizations, sessions, audit logs and so on exist only in Postgres). That is expected; Mongo mode is a frozen local copy, not a second supported database.
If a future feature truly needs a document store, it is added as one new module for that data, with its own tenant rules and review, behind the same repository ports. The whole app is never switched to Mongo.
The mongoose dependencies stay installed and are kept patched like any other dependency.

Tables live in Postgres schemas by module (identity, tenancy, catalog, storage, learning, progress, assessment, ai), all in one TypeORM DataSource. Each entity declares its schema (@Entity({ schema: 'identity', name: 'users' })). The first migration runs CREATE SCHEMA IF NOT EXISTS for all eight. Never rely on search_path. The table-by-table design is in the Database Architecture workbook.

Portability rule: keep the boilerplate's hexagonal layout. Services depend only on the abstract repository ports in infrastructure/persistence/*.repository.ts, never on TypeORM classes. All TypeORM code stays inside infrastructure/persistence/relational/.

Connection settings come only from env or the secret manager. In production, require DATABASE_SSL_ENABLED=true with certificate verification, and fail startup if it is missing. Never log connection strings or credentials.

Database roles. Each role has its own password in the secret manager, rotated automatically (system-design C2).

lms_migrator owns the tables, runs migrations, and is the only role that creates or drops partitions. Used only by the deploy and ops pipelines.
lms_app is what the API uses: not the owner, no BYPASSRLS, no DDL. Role-level settings: statement_timeout = 5s, idle_in_transaction_session_timeout = 5s, lock_timeout = 2s.
lms_worker is what background jobs use: the same rights as lms_app, with statement_timeout = 60s, idle_in_transaction_session_timeout = 30s, lock_timeout = 5s. Jobs work in batches and commit per batch. Jobs that span organizations loop over organizations with one unit of work each, so RLS still applies.
lms_security_watcher is used only by the pending-security watcher, over its own direct connection: SELECT (subject_key, processed_at) on tenancy.security_outbox, LISTEN, nothing else.
lms_audit_writer writes anonymous audit rows (Audit log): INSERT on tenancy.audit_logs, the matching rights on tenancy.audit_chain_heads, no SELECT on audit logs.
lms_retention is the only role that may delete individual expired audit rows (offboarding, per-organization policies shorter than a month). Whole partitions are dropped by the ops pipeline as lms_migrator.
lms_platform_reader is read-only with BYPASSRLS, used only for cross-tenant platform reporting against a read replica, never by the API or workers. Defined now so nobody adds an ad-hoc bypass later.
Roles

There are two kinds of people, stored in two tables, with two separate logins.

Platform admin (identity.platform_admins): the team that runs the product. Global, not in any organization. Must use MFA (WebAuthn required by the end of Phase 1b). Creates and suspends organizations. Enters a customer organization only through a time-limited, audited support session (Part 5). Actions that delete or export a whole organization, or reset an MFA factor of the last Org Admin or of another platform admin, need a second platform admin's approval (Part 5).
Organization user (identity.users, scoped by organization_id): everyone inside a customer organization. Each has exactly one OrganizationRole in that organization.

Nobody can impersonate a user. Support sessions act as the platform admin, with the session's permissions.

The boilerplate's platform RoleEnum (admin, user) and the roles table are removed. @Roles(RoleEnum.admin) on the users admin API is replaced by @PlatformAdmin().

Future multiple roles. One role per user keeps Phase 1 simple. Guards and services only ever ask "does this caller hold permission X", through AuthContextService.getPermissions(), never "is this caller role Y". A later user_roles table then changes only how the permission set is built, not any guard.

Organizations are provisioned in one workflow (Part 5). It seeds three roles and sends an invitation to the first Org Admin; no platform admin ever sets a customer's password:

Org Admin (slug org-admin, allPermissions: true, system: true). Satisfies every current and future organization permission. Cannot be edited or deleted. An organization can never drop to zero active users with this role. Users holding a role with allPermissions or user.write/role.write must have MFA enrolled (Part 5).
Instructor and Learner are editable starter roles. Instructor starts with organization.read and user.read. Learner starts with organization.read. The Org Admin can change or delete them and create more roles (for example "Nurse educator") from the assignable permission list.

Assignable permissions in this task: organization.read, organization.write, user.read, user.write, role.read, role.write, audit.read. A custom role cannot set allPermissions. Platform-only actions are not assignable. Unknown permission strings are rejected.

No privilege escalation. These rules apply to every role and user write, and each has an e2e test:

A caller can only put permissions on a role that the caller already holds.
A caller cannot edit the role they currently hold.
Only an Org Admin or a platform admin can assign the Org Admin role, or any role whose permissions exceed the caller's own.
A user cannot change their own role or lock or delete their own account through the user-management API.
Part 2 — Data model

Add entities with the project generator, one at a time, then properties one at a time. Do not hand-write entity files. After generation, change the primary key to a UUIDv7 generated in application code. Every tenant table gets UNIQUE (organization_id, id), a tenant-aware foreign key for each parent, and its RLS policy in the same migration that creates it. Global tables are listed in global-tables.ts (foundation §4.1). Full column lists are in the Database Architecture workbook.

Columns marked (encrypted) are encrypted in the application with the organization's data key (Part 6), or with the platform key for global tables.

Entity	Key fields	Constraints
tenancy.organizations (global)	name, slug, status (provisioning, active, suspended, deleted), settings (includes allowSelfRegistration, requireMfaForAll, timeZone), provisioning_key, data_key_id, data_key_wrapped, storage_kms_key_arn, deleted_at, slug_reserved_until	Unique slug (a deleted organization's slug stays reserved for 90 days). Unique provisioning_key. After deletion only a tombstone remains: id, status, dates.
tenancy.organization_slug_aliases (global)	slug, organization_id, expires_at	Unique slug across organizations, aliases and reserved deleted slugs. Kept 90 days after a rename.
tenancy.organization_plans	plan_name, seat_limit, storage_limit_bytes, ai_monthly_budget_usd, api_rate_limit, features	One active plan per organization. Locked with FOR UPDATE whenever a seat may be consumed.
tenancy.organization_storage_usage	organization_id (PK), used_bytes, reserved_bytes, reconciled_at	One row per organization. Locked with FOR UPDATE for every reservation.
tenancy.organization_roles	name, slug, all_permissions, system, permissions (text[]), version	Unique (organization_id, slug). version only ever increases.
identity.users	UUIDv7 id, organization_role_id, email, email_verified_at, password (nullable), password_changed_at, names, status_id, authz_version, mfa_secret (encrypted), mfa_enabled_at, mfa_last_used_step, deleted_at (failed-login counters live in redis-cache, Part 5)	Unique (organization_id, lower(email)) ignoring deleted rows.
identity.user_recovery_codes	user_id, code_hash, used_at	Hashed; each usable once.
identity.user_identities	user_id, provider, external_subject, email_at_link, linked_at, last_used_at	Unique (organization_id, provider, external_subject).
identity.identity_link_tickets	provider, external_subject, email_reported, ticket_hash, expires_at, consumed_at, consumed_by_user_id	A pending link, valid 10 minutes. Consumed once.
identity.email_action_tokens	user_id, purpose (verify_email, reset_password, change_email, revert_email), token_hash, new_email, previous_email, expires_at, used_at, revoked_at	Single-use (Part 5). Issuing a new token for the same user and purpose revokes older ones.
identity.user_consents	user_id, notice_version, purpose, consented_at, withdrawn_at, ip, user_agent	Append-only history. The current notice version must be accepted before tokens are issued.
identity.account_directory (global, directory)	email_hash (HMAC-SHA-256 with a platform secret), organization_id, user_id	Unique (email_hash, organization_id). Maintained in the same unit of work as every user create, email change, lock and delete. Read only by the find-organization job.
identity.identity_providers (reserved, Phase 5)	type, status, config (encrypted), allowed_domains, jit_provisioning	Designed now, built with SSO.
identity.sessions	user_id, organization_id (NOT NULL), hash, previous_hash, rotated_at, revoked_at, revoke_reason, reauth_at, expires_at, device_label, client_platform, last_seen_at, ip_first, user_agent_first	User sessions only. Tenant-aware FK to the user. Revoked rows are kept until the cleanup job removes them after expires_at, so replay detection still works.
identity.platform_admins (global)	UUIDv7 id, email, password, names, status_id, authz_version, totp_secret (encrypted), totp_last_used_step	Unique lower(email). No tokens until MFA is set up.
identity.platform_admin_webauthn_credentials (global)	platform_admin_id, credential_id, public_key, sign_count, transports, name, created_at, last_used_at	Unique credential_id. A sign count that does not increase is rejected and alerted.
identity.platform_admin_recovery_codes (global)	platform_admin_id, code_hash, used_at	Hashed; each usable once.
identity.platform_admin_sessions (global)	platform_admin_id, hash, previous_hash, rotated_at, revoked_at, revoke_reason, reauth_at, expires_at, ip_first, user_agent_first, last_seen_at	Same rotation rules as user sessions. Shorter lifetime (refresh 12 hours).
tenancy.platform_approvals (global)	action (org_delete, org_export, reset_last_org_admin_mfa, reset_platform_admin_mfa), target_type, target_id, reason, requested_by, approved_by, status (pending, approved, rejected, expired, executed), expires_at	approved_by must differ from requested_by (CHECK). Expires after 24 hours.
tenancy.support_sessions	platform_admin_id, organization_id, reason, ticket_ref, granted_permissions, approval_status, approved_by_user_id, approved_at, status (pending_approval, active, ended, revoked, expired), version, starts_at, expires_at, ended_at	Max 60 minutes. version increases on every change.
tenancy.invitations	email, organization_role_id, token_hash, expires_at, accepted_at, accepted_user_id, revoked_at, bulk_import_id, delivery_status	One open invite per email per organization.
tenancy.bulk_imports	kind (invitations), file_id, status, total_rows, accepted_rows, rejected_rows, report_file_id	One running import per organization.
tenancy.audit_logs	chain_id, seq, actor_type, actor_id, support_session_id, organization_id, action, outcome, target_type, target_id, ip, user_agent, request_id, email_hash, prev_hash, row_hash, created_at	Partitioned by month on created_at, so every unique key includes it: Unique (chain_id, seq, created_at), Unique (organization_id, id, created_at). Nothing references this table by foreign key. organization_id is null only for the platform and anonymous chains. Append-only, with per-role RLS policies (Audit log).
tenancy.audit_chain_heads	chain_id (PK: org:{id}, org:{id}:anonymous, anonymous, or platform), chain_kind, organization_id (null for anonymous and platform), last_seq, last_hash, updated_at	Not partitioned. Locked per insert to serialize each chain. Per-role SELECT, INSERT and UPDATE policies (Audit log).
tenancy.data_requests	kind (export, delete, user_export, user_delete), subject_user_id, requested_by_type, requested_by_id, platform_approval_id, status (pending, awaiting_approval, grace_period, running, completed, cancelled, failed), grace_until, result_file_id, certificate_file_id, created_at, completed_at	One open request per kind per organization (per user for user_* kinds). Organization export and delete require an approved platform_approvals row.
tenancy.retention_policies	data_class (audit_security, sessions, outbox, ai_prompts, model_logs, …), retain_days, action (delete, anonymize)	Unique (organization_id, data_class). audit_security retain_days ≥ 365.
storage.files	storage_key, original_name (sanitized), mime_type, detected_mime_type, declared_size_bytes, size_bytes, status (pending_upload, uploaded, scanned_clean, infected, expired), reservation_expires_at, checksum, scan_status	size_bytes set only after the upload is verified.
tenancy.security_outbox (global, internal queue)	organization_id (information only), kind, subject_key, payload, attempts, next_attempt_at, processed_at	Same transaction as the security change. Grants: lms_app INSERT, SELECT (id), UPDATE (attempts, next_attempt_at, processed_at); lms_worker (the outbox workers) SELECT, INSERT and the same UPDATE columns; lms_security_watcher SELECT (subject_key, processed_at). Partial index on subject_key WHERE processed_at IS NULL.
tenancy.email_outbox (global, internal queue)	organization_id (information only), template, to_email, payload (encrypted), attempts, next_attempt_at, sent_at, last_error, provider_message_id, bounced_at, bounce_type, complained_at	Same transaction as the invitation or reset. Grants as for security_outbox: the API can insert but never read to_email or payload; only the email worker (lms_worker) reads them. Bounce and complaint webhooks update it (Part 8).
tenancy.idempotency_keys	actor_type, actor_id, key, request_hash, response_status, response_body, expires_at	Unique (organization_id, actor_id, key). Kept 24 hours (foundation §7.5). Platform-level keys use the global sibling tenancy.platform_idempotency_keys.
tenancy.role_assignments (reserved, Phase 2)	user_id, organization_role_id, scope_type (organization, course, batch), scope_id	Unique (organization_id, user_id, organization_role_id, scope_type, scope_id). Designed now; AuthorizationService.assert() reads it in Phase 2.
catalog.catalog_items (global, reserved, Phase 2)	type, title, version, status, content_ref	Written only by platform admins. No RLS; reachable by organizations only through licenses.
learning.catalog_licenses (reserved, Phase 2)	catalog_item_id, licensed_at, expires_at, seat_limit	Unique (organization_id, catalog_item_id). Under RLS. Fork-on-edit copies carry source_catalog_item_id.

The boilerplate's roles table and RoleEnum are removed. statuses stays. Scheduled jobs delete expired sessions and processed outbox rows. Outbox workers claim rows with SELECT … FOR UPDATE SKIP LOCKED so several workers run in parallel. Generate migrations with npm run migration:generate and review each by hand. Never use synchronize: true except in local tests.

Part 3 — Request flow and new files

The access token carries sub (the user or platform admin id), subType (user or platform_admin), sessionId, organizationId, organizationRoleId (users only), supportSessionId (support tokens only), ver, typ, iss, aud, iat, nbf, exp, and the kid header.

Every request goes through these steps in order:

The middleware assigns a requestId (or accepts a valid incoming UUID) and reads the Authorization: Bearer header. The client IP comes only from the trusted proxy chain (foundation §11 rule 17).
No header: continue with an empty context.
Header present: verify the signature with EdDSA only, against the key named by kid, and require typ: "access", the expected iss and aud, and valid exp and nbf with 30 seconds of skew. Invalid, expired, unknown kid, or wrong type returns 401.
Security-state sentinel. If this replica's cached view of sec:epoch (re-checked once per second) shows it missing, return 503. See "Redis data loss" in Part 5.
Freshness lease. If this replica's pending-security set was last synced more than SECURITY_LEASE_MS ago (monotonic clock), return 503.
Pending-change barrier (in memory): if the token's user, session, organization, role or support session is in the pending-security set, return 401.
redis-security checks, read per cluster node in parallel, overall timeout about 50 ms. Any error or timeout returns 503 (fail closed).
deny:org:{organizationId} exists → 401.
deny:session:{sessionId} (or deny:psession:{id} for platform admins) exists → 401.
Token ver is less than authz:ver:{subType}:{sub} → 401.
For a support token: read support:{supportSessionId} (Part 5). Missing, not active, expired, or its platformAdminId or organizationId differs from the token → 401.
A missing version key is never treated as 0. Read the durable value from Postgres, write it back with the compare-and-set script, then compare.
Resolve permissions, always from redis-security, never from the token or a local cache. For a user: role:perms:{organizationRoleId} ({ allPermissions, permissions, version }), filled on a miss with the compare-and-set script. For a support session: the permissions inside the support:{id} entry. A platform admin outside a support session has no organization permissions.
Store requestId, subject, session, organization, role, support session and permissions in AsyncLocalStorage, and set request.user.
The global AuthorizationGuard applies the handler's declared authorization (below).
The controller runs. Services read tenant data only through short units of work (Part 4).

After the guard, the throttler, idempotency, response cache, validation and response formatting run as defined in foundation §2.

Version-aware cache writes. Every Redis write of a version, permission set or support session goes through one Lua compare-and-set script on one key: write only when the key is missing or its stored version is lower. Security changes never DEL a key to invalidate it; they write the new value with its new version.

Why a refilled cache can't be stale. A cache miss refills from Postgres, which already holds every committed change. A refill whose read started before a change can only land if the key is missing; the change's own write (newer version) then overwrites it. If that newer write failed, the role is in the pending-security set, so its users are refused (step 6) until the outbox lands the new version. No replica keeps its own copy, so no replica can hold an older set than Redis.

Long-lived connections. SSE and WebSocket connections (later: tutor chat, live sessions) authenticate with the Authorization header or a single-use stream ticket (foundation §7.7), pass the same checks at connect time, then repeat steps 4–8 every 30 seconds and on every security NOTIFY that matches the connection's subject. The connection is closed when any check fails.

MFA challenges. A login that needs MFA returns { challengeId } instead of tokens. The challenge is stored in redis-security as mfa:{hash(challengeId)} with the subject, organization, a 5-minute TTL and an attempt counter. Five wrong codes delete the challenge; ten failed MFA attempts for one account within an hour add a 15-minute delay to that account's MFA and alert. The challenge is consumed atomically on success.

Every protected handler declares its authorization. Exactly one of these decorators is required on every route handler:

@Public() — no login needed. Only the routes listed below.
@Permissions('user.read', ...) — needs at least one listed permission in the active organization.
@PlatformAdmin() — platform admin only, outside any organization.
@Authenticated() — any logged-in subject, for self-service routes only (GET /me, change own password, MFA enrollment, my sessions, stream tickets, consent, logout). Every use is listed in src/permissions/authenticated-allowlist.ts, which changes only through code review.

A handler with none of them is denied at runtime (403) and logged as an error. At startup, and in CI, a route scanner lists every handler and fails the build when one has no declaration, more than one, or an @Authenticated() not on the allowlist.

@NoSupportAccess() can be added to any handler, and is required on credential, MFA, session, invitation-token, stream-ticket, export and deletion routes. Support sessions are always refused there.

@RequireRecentAuth(minutes) can be added to any handler and is required on role and permission writes, invitations, user lock and delete, identity link and unlink, MFA changes, email changes, exports and deletion requests. The caller must have re-entered their password (and MFA code when enrolled) within the window, through POST /api/v1/me/reauth, which records reauth_at on the session.

The middleware is the only place that verifies the access token. The Passport jwt guard must not verify it again; make it read request.user only. Keep the jwt-refresh strategy for the refresh route (HS256 only), and exclude refresh from the middleware with MiddlewareConsumer.exclude({ path: 'auth/refresh', method: RequestMethod.POST }).

Refresh opens the right unit of work. The refresh route verifies the refresh token, then reads organizationId and sessionId from its signed claims and opens TenantUnitOfWork for that organization to find the session. Platform admin refresh uses /api/v1/admin/auth/refresh and platform_admin_sessions in a PlatformUnitOfWork.

New files:

src/common/context/request-context.ts — AsyncLocalStorage store. run() wraps next().
src/common/auth/auth-context.middleware.ts — global middleware described above.
src/common/auth/auth-context.service.ts — getSubjectId(), isPlatformAdmin(), getUserId(), getOrganizationId() (400 when none is active), getOrganizationRoleId(), getSupportSession(), getPermissions(), resolveScope().
src/permissions/decorators.ts — @Public(), @Permissions(), @PlatformAdmin(), @Authenticated(), @NoSupportAccess(), @RequireRecentAuth().
src/permissions/authorization.guard.ts — global APP_GUARD enforcing the declared authorization. Denies any handler without a declaration.
src/permissions/route-authorization.scanner.ts — startup and CI check that every handler has exactly one declaration.
src/permissions/authenticated-allowlist.ts — the reviewed list of @Authenticated() routes.
src/permissions/permission.ts — the Permission string union, including the support.* permissions.
src/permissions/authorization.service.ts — assert(permission, resource?); organization-level in Phase 1, reads role_assignments in Phase 2.
src/common/tenancy/tenant-unit-of-work.ts and platform-unit-of-work.ts — the only code allowed to call set_config (Part 4).
src/common/tenancy/global-tables.ts — the allowlist of global tables (foundation §4.1).
src/common/tenancy/scope.ts — scope modes and the @OwnerColumn() decorator.
src/redis/redis.module.ts — three named clients, SECURITY_REDIS (cluster), QUEUE_REDIS, CACHE_REDIS; src/redis/versioned-cas.lua; src/redis/node-split-reader.ts (per-node parallel reads).
src/auth/keys/signing-key.service.ts — loads the active and previous Ed25519 keys from the secret manager, signs with the active key, serves /.well-known/jwks.json.
src/auth/authz-version.service.ts — the only code that makes security changes (Part 5).
src/auth/mfa/mfa-challenge.service.ts — challenges and attempt limits.
src/security/pending-security.watcher.ts — holds the pending-security set and its lastSyncedAt (monotonic). Connects directly to Postgres as lms_security_watcher (not through PgBouncer, because transaction pooling drops LISTEN), listens on security_outbox, reconnects on failure, and re-reads all unprocessed keys every 500 ms as the backstop.
src/security/security-sentinel.service.ts — checks sec:epoch once per second.
src/security/security-outbox.worker.ts — retries Redis writes for unprocessed outbox rows.
src/security/security-state-rebuild.job.ts — rebuilds deny state from Postgres and writes sec:epoch (Part 5).
src/security/crypto/data-key.service.ts — per-organization envelope encryption (Part 6).
src/auth/identity/identity-provider.interface.ts — one interface for every external login.
src/audit/audit-chain.service.ts — the only writer of audit rows, with per-chain serialization (Audit log).
src/storage/storage-quota.service.ts — reserves, commits and releases storage capacity (Part 7).
src/health/health.controller.ts — /health/live and /health/ready (Part 8).
src/common/http/response.interceptor.ts, src/common/errors/problem-details.filter.ts, src/common/errors/error-codes.ts — response shape, headers and error codes (foundation §7).
src/common/http/cursor.ts, src/common/http/idempotency.interceptor.ts, src/common/http/safe-fetch.ts.
src/common/cache/cacheable.decorator.ts and cache.interceptor.ts — tenant-safe response cache (foundation §6.1).
src/common/batch/batch-loader.ts — request-scoped batching (foundation §6.2).
src/common/content/sanitize.ts and src/common/content/csv-escape.ts — rich text sanitizing and export escaping (foundation §10).
src/common/observability/sentry.filter.ts — scrubbed error reporting.

Wire the middleware and guard in src/app.module.ts. There is no per-module auth and no flag that turns authorization off in development.

Mark @Public() on: GET /, email register, login, confirm, forgot and reset, refresh (user and admin), invitation accept, organization discovery, the Google, Facebook and Apple login routes, user MFA verify, email-change confirm and revert, admin login and admin MFA, /.well-known/jwks.json, the minimum-client-version document, and the email provider's bounce webhook (signature-verified). Nothing else.

Part 4 — Tenant isolation: three layers

A missing organization filter is a data leak between customers. Isolation is enforced three times, so one mistake is never enough.

Layer 1 — Tenant repository (code)
src/common/tenancy/tenant-context.ts and src/common/tenancy/tenant-scoped.repository.ts. Find, update and delete always add organization_id and exclude soft-deleted rows. A missing organization throws.
Every repository method takes a scope mode (organization by default, owner, catalog, global) with the guard rails in foundation §4.2. global on a table outside global-tables.ts throws; owner on an entity without @OwnerColumn throws.
Every tenant entity's repository extends it. Services never call raw TypeORM Repository methods on tenant data.
The organization comes from AuthContextService (HTTP) or the job's TenantContext (workers), never from the body, params or query.
Reading another organization's record returns 404, not 403.
A lint rule fails if a tenant module imports Repository from typeorm outside infrastructure/persistence/relational/, or if any file other than the unit-of-work classes calls set_config.
Layer 2 — Tenant-aware foreign keys (database)
Every tenant table has UNIQUE (organization_id, id). Partitioned tables (audit logs, and later progress, xAPI and AI usage events) are the exception: every unique key also includes the partition key (UNIQUE (organization_id, id, created_at)), and they are leaf tables that nothing references by foreign key.
Every foreign key to a tenant table is composite: FOREIGN KEY (organization_id, course_id) REFERENCES learning.courses (organization_id, id). The database then refuses any link between two organizations.
Layer 3 — Row Level Security (database, from the first migration)
The API connects as lms_app and jobs as lms_worker (Part 1): not the owner, no BYPASSRLS, no DDL.
Each tenant table's migration enables and forces RLS with one policy: USING (organization_id = current_setting('app.organization_id', true)::uuid) WITH CHECK (same). A missing setting matches nothing, so it fails closed. The only exceptions are the audit tables, whose per-role policies allow the null-organization platform and anonymous chains (Audit log). Global tables have no RLS and are protected by grants.
Every view is created WITH (security_invoker = true). SECURITY DEFINER functions are forbidden in tenant schemas. The migration check rejects both.
Short units of work, not one transaction per request. TenantUnitOfWork.run(tenantContext, async (manager) => { ... }) borrows a connection, starts a transaction, runs SELECT set_config('app.organization_id', $1, true), runs the callback's queries, then commits or rolls back and releases the connection immediately. A service method that changes several rows uses one unit of work, so the change is atomic.
Platform work. PlatformUnitOfWork.run(fn) sets only app.platform = on (global tables and the platform audit chain). PlatformUnitOfWork.forOrganization(orgId, fn) sets both settings and is the only way a platform admin action touches a tenant table: provisioning, suspend and reactivate (which revoke sessions), support sessions, data requests, MFA resets. Both are reachable only from @PlatformAdmin() services.
Tenant repositories only accept a manager from a unit of work, so a query cannot run without the tenant set.
Never call an external service (AI provider, email, file storage, Redis, KMS) inside a unit of work. Read, commit, then call. Write results back in a new unit of work.
The setting is transaction-local, so this works with PgBouncer in transaction mode. Keep the pool small per instance. The pending-security watcher's connection is the one exception and connects directly (Part 3).
Global tables have no RLS. Login resolves the slug from organizations (and organization_slug_aliases) first, then opens a unit of work for that organization to look up the user.
Part 5 — Login, invitations and sessions

User login. POST /api/v1/auth/email/login with { organization, email, password }, where organization is the slug or an alias. Resolve it to an active organization, then find the user by (organization_id, lower(email)). Any failure (unknown organization, unknown email, wrong password, locked account) returns the same generic 401 and takes the same time (run bcrypt against a dummy hash when the user is not found). The current slug is returned on success so clients update what they store.

On success: if the current consent notice has not been accepted, return consent_required with a short-lived consent challenge (Part 6). If the user has MFA enabled or is required to enroll, return an MFA challenge instead of tokens. Otherwise create a session with the organization and issue tokens carrying organizationId, organizationRoleId and the current ver.

Brute-force protection, without letting attackers lock people out. Rate limits per IP (generous, because of carrier NAT) and per organization plus email. After 5 failed passwords for one account, each further attempt for that account waits an increasing delay (1 s, 2 s, 4 s … capped at 60 s), and the account owner is emailed after 10 failures in an hour. The failure counter is kept in redis-cache under an HMAC of (organization id, lowercased email) and is counted and applied the same way whether or not the account exists, so the delay never reveals which accounts exist; it is never stored on the users row. Accounts are never hard-locked by failures, so an attacker cannot lock a real user out. After repeated failures from an IP or for an account, public auth routes require a CAPTCHA token (a provider with web and mobile SDKs, system-design C2); the CAPTCHA is an extra hurdle, never a replacement for the limits.

User MFA (Phase 1). TOTP with hashed one-time recovery codes. Optional for every user, required for any user whose role has allPermissions, user.write or role.write, and for everyone when the organization sets requireMfaForAll: such a user without MFA can only call MFA enrollment until enrolled. A used TOTP time step is stored (mfa_last_used_step) and a code from the same or an earlier step is rejected. Challenge attempts are limited (Part 3). The secret is encrypted with the organization's data key.

Account recovery when MFA is lost.

A user who lost their device uses a recovery code. Without one, an Org Admin can reset that user's MFA (user.write, recent re-authentication, audited); the user is emailed and must enroll again at next login, and all their sessions are revoked.
An Org Admin cannot reset their own MFA. Another Org Admin can.
If the organization's last Org Admin loses MFA and recovery codes, a platform admin opens a request after verifying the person out of band (signed request from the customer's authorized contact on file). It needs a second platform admin's approval through platform_approvals, waits 24 hours, and both the Org Admin's email and the organization's other contacts are notified before it runs.
A platform admin's lost factors are reset only by another platform admin, with two-person approval, and the admin must enroll WebAuthn again.

Organization discovery. Users often don't know their organization slug. POST /api/v1/auth/find-organization with { email } always returns the same 202 and enqueues a job. The job computes the email's HMAC, reads identity.account_directory (a global table holding only hashes), and emails the list of organizations where that email has an active account, or nothing. The response never reveals memberships. Rate-limited per IP and per email.

Platform admin login, with MFA. POST /api/v1/admin/auth/login with { email, password } returns an MFA challenge, never tokens. POST /api/v1/admin/auth/mfa with { challengeId, ... } accepts a WebAuthn assertion (required for every platform admin by the end of Phase 1b), a TOTP code, or a one-time recovery code, and returns tokens with subType: platform_admin, no organization, and a session in platform_admin_sessions. An admin without MFA set up can only call the MFA enrollment route. A WebAuthn sign count that does not increase is rejected and alerted. Admin routes are served only on a separate hostname (admin-api.) behind an IP allowlist or VPN, with their own CORS allowlist. A platform admin can never log in through the user login, and a user never through the admin login.

Two-person rule. These platform actions create a platform_approvals row and run only after a different platform admin approves it within 24 hours: organization export, organization delete, resetting the last Org Admin's MFA, resetting another platform admin's MFA. Both admins need recent re-authentication. Requests, approvals, rejections and execution are audited, and the affected organization's Org Admins are emailed.

Support sessions, with explicit permissions. HTTP methods never decide support access; permissions do.

POST /api/v1/admin/support-sessions with { organizationId, reason, ticketRef, permissions }. permissions comes from a fixed support allowlist, each mapping to exactly one organization permission:
Support permission	Grants
support.organization.read	organization.read
support.user.read	user.read
support.role.read	role.read
support.audit.read	audit.read
support.user.write	user.write
support.role.write	role.write
support.organization.write	organization.write
Read-only permissions start at once. Any write permission needs approval from one of the organization's Org Admins: the session stays pending_approval until approved or rejected in the product. Every support session start is emailed to the organization's Org Admins. The approval is audited.
Server-side binding. The authoritative state is the support_sessions row, written through PlatformUnitOfWork.forOrganization(). On every change (create, approve, reject, end, revoke, expire) the row's version increases and the full state is written to Redis as support:{id} = { platformAdminId, organizationId, permissions, status, version, expiresAt }, through the compare-and-set script and the outbox. The token carries only supportSessionId, organizationId and the admin's id. The middleware requires that entry to exist, be active, be unexpired, and match the token's admin and organization, and it takes the permissions from the entry. A token can never carry or widen its own permissions.
Inside a support session, the effective permissions are exactly the entry's support.* permissions mapped by the table above. Nothing else, even for a platform admin.
Always refused: routes marked @NoSupportAccess() (passwords, MFA, recovery codes, sessions, invitation and link tokens, stream tickets, data export and deletion, plan and billing changes).
The session lasts at most 60 minutes; the Redis entry expires with it. Ending, revoking or expiry stops the next request. Every request inside it is audited with support_session_id, and Org Admins can list and end any support session on their organization.

Social login, through the identity-provider interface. The request includes the organization. Each provider (Google, Facebook, Apple now; SAML and OIDC later) implements IdentityProvider and returns { provider, externalSubject, email, emailVerified }. Provider tokens are verified against the provider's published keys with the expected audience (our client ids per platform), issuer and nonce.

Known identity: look up user_identities by (organization_id, provider, external_subject). Found: log that user in (through MFA when enabled). A changed email at the provider still finds the same user.
Unknown identity, email matches an existing account: never link automatically, even when the provider says the email is verified. Store an identity_link_tickets row (valid 10 minutes, stored hashed) and return 409 link_required with the ticket. The user must sign in with their existing method (password, plus MFA when enabled), then call POST /api/v1/me/identities/link with the ticket. Only then is the user_identities row created, and an audit row and a notification email are written.
Accepting an invitation with a provider: allowed to link, because the invitation proves the email and the user chose it. The provider's verified email must equal the invitation email.
Unknown identity, no matching account: create an account only if the organization allows self-registration or a pending invitation exists.
Organization SSO (Phase 5): an organization may enable auto-linking only for its own SAML/OIDC provider and only for its verified allowed_domains. Public social providers can never auto-link. Metadata URLs are fetched through safe-fetch.ts.
Unlinking is a self-service route that requires recent re-authentication, and a user can never remove their last way to log in.

Self-registration. POST /api/v1/auth/email/register with the organization slug. When settings.allowSelfRegistration is false, or the slug is unknown, the response is the same generic 202 as a successful registration, and nothing is created, so the endpoint reveals neither organizations nor accounts. When the email already has an account, the response is the same 202 and the owner receives an "you already have an account" email. All email work is queued, so every branch takes the same time. New users get the Learner role, record their consent, and cannot log in until email_verified_at is set through the confirm link.

Invitations.

POST /api/v1/invitations with { email, organizationRoleId } (needs user.write and recent re-authentication; the no-escalation rules apply).
Generate a random 32-byte token. Store only its SHA-256 in token_hash. Expires after 7 days. The invite email is written to email_outbox in the same transaction, so a mail outage never loses it.
POST /api/v1/auth/invitations/accept (public) with { token, password, firstName, lastName, consentNoticeVersion }. In one unit of work: lock the organization's plan row with SELECT … FOR UPDATE, count active users, refuse with 409 if the seat limit is reached, create the user with email_verified_at set, record consent, add the directory entry, and mark the invitation accepted. Two people accepting at once can never exceed the limit.
A used, expired or revoked token returns the same generic 400. Re-sending revokes the old invitation and creates a new one.
Bulk invitations. POST /api/v1/invitations/bulk (user.write, recent re-authentication, Idempotency-Key) takes an uploaded CSV file id (at most 10,000 rows), and a job validates each row (email format, role in this organization, no escalation, no duplicate), creates invitations in batches, stops creating once the seat limit would be exceeded, and writes a result report. The report is CSV-escaped (foundation §10). One bulk import runs per organization at a time.
An email worker sends email_outbox rows with retries and backoff. Failures are logged and alerted, never shown to the user as a security error.

Seat limit, every path. Every path that can make a user active takes the plan row lock and checks the seat limit in the same unit of work: invitation accept, self-registration confirm, unlock, undelete, bulk import, and later SSO just-in-time provisioning.

Passwords. Minimum 12 characters, maximum 72 bytes in UTF-8 (bcrypt ignores everything after 72 bytes, so a longer password is refused with 422 password_too_long rather than silently truncated), no composition rules. Moving to argon2id, which has no such limit, needs a new dependency and is a separate, approved change. Every new password is checked against the Have I Been Pwned range API through safe-fetch.ts (k-anonymity, only a 5-character hash prefix leaves the server); a breached password is refused. If the API is unreachable the check is skipped and logged. bcrypt cost 12, run in a bounded worker pool (foundation §6.4).

Forgot and reset password. Both include the organization slug. Forgot always returns the same 202 and queues the email, so it takes the same time whether or not the account exists. Reset increments the user's authz_version, revokes all their sessions (sets revoked_at, revoke_reason = password_reset; rows are kept for replay detection), and sends a "your password was changed" email.

Emailed tokens are single-use. Every token sent by email (verify email, reset password, change email, revert email) is 32 random bytes, stored only as its SHA-256 in identity.email_action_tokens with a purpose and expiry; invitation tokens follow the same rule in tenancy.invitations. Links carry the token in the URL fragment (https://app.…/reset#token=…), never in the path or query, and the landing page posts it in the body; landing pages send Referrer-Policy: no-referrer. On mobile, the same links open the app through verified App Links and Universal Links. A token is consumed with one conditional update (… SET used_at = now() WHERE token_hash = :h AND purpose = :p AND used_at IS NULL AND revoked_at IS NULL AND expires_at > now()), so it works exactly once, even under concurrent clicks. Issuing a new token revokes older ones of the same purpose. Lifetimes: verify email 24 hours, reset password 1 hour, change email 1 hour, revert email 7 days. Any invalid, used, revoked or expired token returns the same generic 400. This replaces the boilerplate's JWT links.

Change of email. The boilerplate's email change inside PATCH /auth/me is removed. The new flow:

POST /api/v1/me/email with { newEmail }: @Authenticated(), @NoSupportAccess(), @RequireRecentAuth(5). It always returns the same 202. It issues a change_email token to the new address through email_outbox. If the address already belongs to another account in the organization, the new address instead receives an "already in use" email.
The old address immediately receives a notice with a revert_email link valid for 7 days.
Confirm (POST /api/v1/auth/email/change/confirm, public, with the token), in one unit of work: lock the user row, re-check that no other active account in the organization uses the address (409 email_taken), set email and email_verified_at, leave status unchanged, update the directory entry, increment authz_version, revoke every other session, write the audit row, consume the token, and revoke any other outstanding change_email tokens.
Revert (POST /api/v1/auth/email/change/revert, public, with the token): restore the previous address and directory entry, revoke all sessions, increment authz_version, require a password reset at next login, write the audit row, and notify both addresses.
Linked social identities are unaffected (identity is not email). Org Admins cannot change another user's email through the user-management API.

Security notifications. Users are emailed when a new device signs in, their password or email changes, MFA is enabled, disabled or reset, an identity is linked or unlinked, or their sessions are revoked by someone else. An organization's Org Admins are emailed when a support session starts, a write support session is requested, a new Org Admin is assigned, an export or delete is requested, or the last Org Admin's MFA reset is requested. Notifications never include secrets or links that act without a further login.

My sessions. GET /api/v1/me/sessions lists the caller's active sessions (device label, platform, first IP, last seen). DELETE /api/v1/me/sessions/{id} and DELETE /api/v1/me/sessions (all others) revoke them. Both are @Authenticated() and @NoSupportAccess(). Each user has at most 20 active sessions; a new login beyond that revokes the least recently used one.

Refresh, safe under concurrency. Mobile apps often send two refreshes at once. Rotation is one conditional update: UPDATE sessions SET previous_hash = hash, hash = :new, rotated_at = now() WHERE id = :id AND hash = :presented AND revoked_at IS NULL. Hashes are compared in constant time. Then:

One row updated: issue new tokens.
No row updated, but the presented hash equals previous_hash and rotated_at is under 30 seconds ago: a concurrent duplicate. Return 409 with a retry hint; the client retries with the new token it already received. Do not revoke.
The presented hash matches previous_hash after the grace window, or matches nothing: replay. Revoke the whole session (set revoked_at, deny it in Redis through the outbox), audit it, alert, and return 401.
Refresh also reloads the user's status, role and authz_version, and returns 401 if the user is locked or deleted, the organization is suspended, or the version is stale.

Clients refresh single-flight: one refresh in progress, other requests wait for it (system-design C12).

Revocation protocol, with a consistency barrier. Postgres is the source of truth. A security change is reported as effective only once the authorization-serving state reflects it.

Commit. In one Postgres transaction: make the change, revoke affected sessions, write the audit row, and insert a security_outbox row with a subject_key (user:{id}, session:{id}, psession:{id}, org:{id}, role:{id}, support:{id}, padmin:{id}). A trigger sends NOTIFY security_outbox with the key on commit.
Apply. Still inside the API request, write redis-security with the compare-and-set script. Retry up to 3 times within about 500 ms.
Confirm. If Redis accepted the write, mark the outbox row processed and return success: every replica reads Redis on every request, so the change is enforced everywhere from that moment.
If Redis could not be written: return 202 Accepted with status: "pending_enforcement". The outbox worker keeps retrying.
Barrier on every replica. PendingSecurityWatcher adds the key to the replica's pending-security set the moment the NOTIFY arrives, and every 500 ms re-reads all unprocessed keys from security_outbox. Each successful re-read updates lastSyncedAt. The middleware refuses any token whose user, session, organization, role or support session is in the set.
Freshness lease. If lastSyncedAt is older than SECURITY_LEASE_MS (default 2 s; database down, network split, stuck process, lost LISTEN connection), the replica refuses every authenticated request with 503 and raises the lease-expiry metric and alert. Readiness is not affected (Part 8), so public routes such as login and JWKS keep working. It resumes only after a successful full re-read.

Why the bound holds. Let a change commit at time T. Any replica that serves an authenticated request at a time later than T + lease has completed a full re-read after T, because its lease would otherwise be expired. That re-read sees the committed outbox row (or its processed result in Redis), so the replica rejects the old token. A replica that cannot complete a re-read stops serving. No path lets a replica serve a revoked token after T + lease. NOTIFY makes the normal delay milliseconds; the 500 ms poll is the backstop; the lease is the proof.

Redis data loss. Version keys repair themselves from Postgres, but a missing deny key means "allowed". So:

sec:epoch is a sentinel key in redis-security, written only by SecurityStateRebuildJob. Every replica checks it once per second; while it is missing, authenticated requests get 503.
The rebuild job runs at startup of the security worker and whenever the sentinel is missing. In order: it rewrites deny:org for every suspended organization, deny:session and deny:psession for every session revoked within the last access-token lifetime, support:{id} for every active support session, and the current role:perms and authz:ver keys for recently changed roles and accounts, all with the compare-and-set script; then it writes sec:epoch.
MemoryDB's durable log makes this rare; the sentinel makes it safe when it happens.

What each change writes to redis-security:

Logout one device: deny:session:{sessionId} (or deny:psession:{id}) with TTL equal to the remaining access-token life.
Lock, delete, role change, MFA reset or password reset: authz:ver:user:{userId} to the new version.
Suspend an organization: deny:org:{organizationId} until reactivated.
Edit a role: role:perms:{roleId} with its new version and permission set. Delete a role: an empty set with the new version.
Platform admin lock, reset or MFA reset: authz:ver:platform_admin:{id}.
Any support session change: support:{id} with its new version and full state.

A reconciliation job compares recent Postgres versions with Redis every few minutes and repairs drift. It is a repair tool, not part of the guarantee.

Last Org Admin rule. Removing, locking or demoting an Org Admin runs in one transaction that first takes SELECT … FOR UPDATE on the organization's plan row (the one lock row per organization), counts remaining active Org Admins, then writes. Zero remaining returns 409 last_org_admin.

Organization provisioning. POST /api/v1/admin/organizations with { name, slug, plan, firstAdminEmail, authorizedContact } and an Idempotency-Key header. Before the transaction: generate the organization's UUIDv7, create its data key and its storage KMS key (multi-Region, system-design C2). Then one transaction through PlatformUnitOfWork.forOrganization(newId): insert the organization (status provisioning, provisioning_key = the idempotency key), its plan, its storage usage row, its default retention policies, the three seeded roles, an Org Admin invitation, the invite email in email_outbox, and an audit row on the organization's chain; then set status active. Repeating the request with the same key returns the same organization instead of creating a second one. Keys created for a failed attempt are scheduled for deletion by a cleanup job. The platform admin never sets or sees the Org Admin's password.

Slug changes. Only a platform admin can change an organization's slug, and the change is audited. The old slug is written to tenancy.organization_slug_aliases for 90 days. Login, register, forgot, reset and find-organization resolve aliases and return the current slug in the response, so mobile apps update what they store. Aliased slugs and reserved deleted slugs cannot be claimed by another organization until they expire.

Part 6 — Security baseline

Tokens and secrets:

Access token lifetime 15 minutes or less (AUTH_JWT_TOKEN_EXPIRES_IN). User refresh lifetime from env (proposed 30 days, sliding); platform admin refresh 12 hours.
Access tokens: Ed25519 key pairs loaded from the secret manager, identified by kid. The active key signs; the previous key stays in the JWKS for one access-token lifetime after rotation. Rotate at least every 90 days and immediately on suspected exposure (runbook in system-design C10).
AUTH_REFRESH_SECRET (refresh tokens, HS256 only) is at least 32 random bytes. CURSOR_SECRET and the directory HMAC secret likewise, each with a kid for rotation. Validate all key material at startup and refuse to boot otherwise.
Never log tokens, passwords, hashes, invite tokens, reset codes, MFA codes, or request bodies on auth routes.
typ and separate keys make access and refresh tokens impossible to swap. Algorithms are fixed in code (foundation §11 rule 24).
Web clients: until the cookie flow ships, the web app keeps access and refresh tokens in memory only and is served with a strict Content-Security-Policy. The httpOnly refresh cookie (Secure, SameSite=Strict, __Host- prefix where the host allows, path limited to the refresh route, with an Origin check and a CSRF double-submit token) is built in Phase 1b step 20. Mobile and desktop apps keep the refresh token in the OS secure store (Keychain, Android Keystore).

Encryption at rest and crypto-shredding:

Every organization has a data key, generated by KMS and stored wrapped (data_key_wrapped). DataKeyService unwraps it on demand and keeps the plaintext only in memory with a short TTL (5 minutes).
Every organization also has its own storage KMS key (storage_kms_key_arn). Its files are written with SSE-KMS under that key; presigned URLs and CloudFront origin access work with it because the key policy grants the API role and CloudFront.
Encrypted with the organization's data key: MFA secrets, prompts, model logs, chat transcripts, email outbox payloads, and later personal or health data columns.
Global secrets (platform admin TOTP secrets, identity provider configs, directory HMAC secret) use a platform key in KMS.
Both per-organization keys are multi-Region keys with a replica in the disaster-recovery region, so restored backups and replicated objects stay decryptable after a regional failover (system-design C2).
Deleting an organization schedules both of its keys, and their replicas, for deletion. Its encrypted data in backups, replicas and object versions then becomes unreadable.
RDS storage, snapshots, cross-region backup copies, Redis and S3 are encrypted at rest with KMS; all connections use TLS.

HTTP hardening (tighten what exists in src/main.ts and src/utils/validation-options.ts; do not add a second copy):

Production startup fails when APP_CORS_ORIGINS is missing or contains *. The admin hostname has its own allowlist.
Express trust proxy is set to the exact proxy hop count (CloudFront, then the load balancer). A client-supplied X-Forwarded-For never changes the IP used for rate limits, audit or sessions.
Per-route body size limits and DTO input bounds (foundation §6.5).
ValidationPipe with whitelist: true and forbidNonWhitelisted: true.
Cache-Control: no-store and the security headers of foundation §7.7 on every API response.
Swagger UI and the OpenAPI JSON are disabled in production.
@nestjs/throttler with a redis-cache store on login, MFA verify, register, find-organization, forgot, reset, refresh, invitation accept, re-auth, stream tickets, and admin login. Limiter keys combine IP with organization slug plus email. A limiter outage fails open and is logged; revocation still fails closed (separate Redis).
Per-organization fairness and load shedding (foundation §6.4).
Bots: CAPTCHA after thresholds on public auth routes (Part 5). Mobile apps may also send a platform attestation token (Play Integrity, App Attest); it is a risk signal for rate limits, never an authorization input.
Error responses never include stack traces, SQL, or entity internals in production. Every error is application/problem+json with the real HTTP status and a code from error-codes.ts.
Sentry receives only 5xx and unexpected errors, scrubbed, from an approved region.

Outbound requests: only through safe-fetch.ts (foundation §9). Compute uses IMDSv2 or the container metadata endpoint, which the client also blocks.

Passwords: see Part 5 (length, breach check, bcrypt cost 12 in a bounded pool, progressive delays), one generic login error, one generic forgot response.

Database and Redis:

lms_app and lms_worker have no superuser, DDL or BYPASSRLS rights, and have statement, idle-in-transaction and lock timeouts (Part 1). Migrations run as lms_migrator.
Only parameterized queries.
All three Redis instances use TLS and ACL users in production; each service's Redis user can run only the commands it needs (for example, the API user on redis-security cannot run FLUSHALL, KEYS, CONFIG or SCRIPT other than loading the CAS script). Startup fails if redis-security or redis-queue is not noeviction.

Compliance (India first).

Digital Personal Data Protection Act 2023 and Rules: record each user's consent and notice version in identity.user_consents before tokens are issued; a new notice version requires acceptance at next login; withdrawal of consent is a self-service action that blocks further processing for that purpose and offers deletion. Support per-user access, correction and erasure through data_requests (user_export, user_delete). Breach handling follows system-design C10. The organization (hospital, college) is the data fiduciary and MedTrain the processor; the DPA states this.
Data residency: Postgres, Redis, object storage, backups, logs and traces in India regions. AI providers are used only in regions and under contracts (zero data retention) the DPA lists.
Before the first hospital contract: a written threat model and an external penetration test, repeated yearly and after major changes. HIPAA controls only if US customers are signed.
Part 7 — First protected APIs

These routes exist so every Phase 1 table is used and every permission is enforced by a real handler.

Platform admin (@PlatformAdmin(); tenant writes through PlatformUnitOfWork.forOrganization()):

Provision an organization (Part 5) in one idempotent transaction.
List, suspend and reactivate organizations. Set or change a plan. Change a slug.
Open, list and end support sessions.
Request, approve and reject platform_approvals.
Create data requests (organization export or delete), which wait for approval.

Inside the active organization:

GET /organization returns the current organization (organization.read). PATCH changes name and settings (organization.write).
Roles: list (role.read); create, edit, delete custom roles (role.write, recent re-authentication). The system Org Admin role cannot be edited or deleted. A role with users cannot be deleted (409).
Users: list with cursor pagination (user.read); change role, lock, unlock, delete, reset MFA (user.write, recent re-authentication). No-escalation rules apply. Unlock checks the seat limit. A user cannot change, lock, delete or reset MFA on their own account here.
Invitations: create, bulk create, list, resend, revoke (user.write).
Audit log: read the organization's own audit log (audit.read).
Support sessions on this organization: list, approve, reject, end (Org Admin through allPermissions).
Data requests for one user: export or delete a single user's data (user.write, recent re-authentication), for DPDP requests.

Self-service (@Authenticated()):

GET /me, change own password, MFA enroll and disable, recovery codes, re-auth, my sessions, stream tickets, consent view and withdraw, change email, link and unlink identities, export my own data.

Files (storage.files):

Reserve. POST /api/v1/files with { name, mimeType, sizeBytes } and Idempotency-Key. In one unit of work: lock the organization's organization_storage_usage row with FOR UPDATE, check used_bytes + reserved_bytes + sizeBytes against the plan's storage_limit_bytes (409 if over), add sizeBytes to reserved_bytes, and create the files row as pending_upload with declared_size_bytes and reservation_expires_at (15 minutes). Only allowlisted MIME types are accepted. original_name is sanitized (no path separators, control characters or leading dots; at most 200 characters) and is never part of the storage key.
Upload. Return a signed upload URL (a presigned POST with a content-length-range condition and the organization's SSE-KMS key) that accepts exactly declared_size_bytes, to a storage_key that starts with the organization id and ends in a random id. Files over 5 GB (video, Phase 2) use multipart upload (system-design C6). Buckets are private.
Complete. POST /api/v1/files/{id}/complete: read the real object size from storage. If it exceeds the declared size, delete the object and release the reservation. Detect the real type from the file's magic bytes into detected_mime_type; a mismatch with an allowlisted type is rejected the same way. Otherwise, in one unit of work, move the size from reserved_bytes to used_bytes, set size_bytes, and mark the file uploaded, then queue the virus scan.
Scan and process. Scanning and any file processing (image resizing, metadata stripping) run in an isolated worker with no access to the database or other organizations' prefixes. Images have EXIF and GPS metadata removed before they are served.
Expire. A job releases reservations past reservation_expires_at, marks those files expired, and deletes any object uploaded without completing.
Delete. Deleting a file subtracts its size_bytes from used_bytes in the same unit of work.
Reconcile. A nightly job compares used_bytes with the sum of size_bytes and the storage provider's own totals, repairs drift, and alerts on any difference.
Download. The API checks the caller may see the file, then returns a signed URL valid for 5 minutes, on a separate download domain, with Content-Disposition: attachment (RFC 6266 file name) and X-Content-Type-Options: nosniff. SVG, HTML and other active content are never served inline. Files are never served unless scanned_clean.
Remove the boilerplate's multer/local upload path once the presigned flow exists.

Offboarding and retention:

Export. Needs an approved platform_approvals row (or, for a single user, the user-level request). A job runs with the organization's TenantContext in batches, writes one encrypted archive to storage.files, and records it on the data request. The download link needs recent re-authentication and expires after 24 hours.
Delete. Needs an approved platform_approvals row, then waits a grace period (30 days, cancellable by the organization's Org Admins). Before deleting, the job confirms that every audit row for the organization has been exported to the write-once store. Then it deletes every row with that organization_id (one batched unit of work per table), every file under its storage prefix, the directory entries, and finally schedules destruction of the organization's data key and storage key in KMS (minimum KMS waiting period).
What remains, on purpose:
The write-once audit export, kept only until its retention period ends, then expired automatically by the storage object lock. It holds no passwords or tokens, emails are hashed, and actor ids no longer resolve to any person.
An organization tombstone: id, status deleted, creation and deletion dates. Its slug stays reserved for 90 days.
Database backups and object versions until their retention window ends. Their encrypted contents are unreadable once the keys are destroyed; plaintext columns in backups (names, emails) expire with the backup retention.
Deletion certificate. The customer receives a record listing what was deleted, what was retained, why, the date each will expire, and the date the keys were destroyed. The customer contract (DPA) must state this retention.
Single-user deletion. A user_delete request removes or anonymizes that user's rows in the organization and their directory entry, keeps their audit rows with the actor id no longer resolving to a person, and is itself audited.
Retention. A scheduled job reads tenancy.retention_policies and deletes or anonymizes old rows for each organization. Security audit rows are never deleted from the live database before they are exported.
Part 8 — Production operations

Observability.

Every log line is structured JSON with requestId, organizationId, subjectType, subjectId and supportSessionId where known. Never log secrets, request bodies, URLs with query strings on auth routes, or email addresses (use the email hash).
OpenTelemetry traces across HTTP, Postgres, Redis and workers, carrying requestId.
Security metrics: login failures per organization, progressive-delay activations, CAPTCHA challenges, MFA failures, refresh replays, 401/403/503 rates, outbox lag, freshness-lease expiries, sentinel missing, RLS denials, rate-limit hits, breach-check refusals, WebAuthn sign-count anomalies, support sessions opened, two-person approvals.
Alerts: login-failure spike, any refresh replay, any WebAuthn sign-count anomaly, outbox row older than 1 minute, any lease expiry, sentinel missing, LISTEN connection lost, any Redis instance unreachable or above 70% memory, any RLS or tenant-aware FK violation (these should never happen; one means a bug), default audit partition non-empty, fewer than 2 future audit partitions, failed audit insert.

Health. /health/live returns 200 while the process runs and its event loop responds. /health/ready returns 200 once startup is complete (configuration validated, signing keys loaded, first pending-security sync done) and stays 200 afterwards. It never checks shared dependencies, the lease or the sentinel: during a Postgres failover or Redis outage every replica would turn not-ready at once and public routes would fail with gateway errors. Shared-dependency failures are handled per request (503 with Retry-After) and by alerts. An internal /health/deps (not public, not used by the orchestrator) reports each dependency for dashboards. None of them reveals versions or internals.

Graceful shutdown. On SIGTERM a replica stops accepting new connections, finishes in-flight requests within 25 seconds, tells SSE clients to reconnect, closes its database and Redis connections, and exits. Workers finish or release their current job. Deploys rely on this to lose no requests.

Email delivery. The sending domain publishes SPF, DKIM (2048-bit) and DMARC (p=quarantine, then p=reject). The provider's bounce and complaint webhook (signature-verified, @Public) uses the organization and outbox ids in the message tags to update email_outbox and the invitation. A hard bounce on an invitation shows as bounced and notifies the inviter; a complaint suppresses the address. Delivery and bounce rates are metrics with alerts.

Backup and recovery.

Postgres: managed automated backups with point-in-time recovery, copied to the disaster-recovery region in India. Target RPO 15 minutes and RTO 4 hours, adjusted to contracts. Backup retention is stated in the DPA.
redis-security: durable log plus a replica. After a data loss, SecurityStateRebuildJob restores deny state and the sentinel (Part 5); until then authenticated requests get 503. redis-queue: Postgres outboxes re-enqueue anything lost. redis-cache needs no backup.
File storage: versioning on the bucket and replication to the disaster-recovery region.
KMS: data keys are never exported; multi-Region replicas allow recovery; key destruction uses the KMS waiting period so a mistaken deletion can be cancelled.
A restore drill into a separate environment every quarter, with the result written down.

Partitions. The ops pipeline (as lms_migrator) creates audit and event partitions at least 3 months ahead, and drops expired months only after that month's write-once export is confirmed.

CI/CD security.

Every pull request runs: lint, type check, unit and e2e tests (including the cross-tenant matrix), the route scanner, the input-bound check, the cache checks, dependency vulnerability scan, secret scan, and a migration check that fails if a new tenant table lacks organization_id, a tenant-aware FK, an RLS policy, or an index starting with organization_id; if a partitioned table has a unique key without the partition key; or if a migration adds a view without security_invoker or a SECURITY DEFINER function.
Branch protection and CODEOWNERS (foundation §16). Container images are built in CI only, scanned, and signed; production runs only signed images.
Production secrets and signing keys only from the secret manager, never from the repository. No human has standing access to production data (system-design C10 break-glass).
Migrations run as lms_migrator in the deploy pipeline, never from the app at startup.
Audit log

What is audited.

Login success and failure, MFA success and failure, re-authentication, logout, password reset, refresh replay, session revoke, consent accepted and withdrawn, invitation sent, accepted and revoked, bulk import.
Platform admin login, support session requested, approved, rejected, opened and ended, every request inside a support session, platform approvals requested, approved, rejected and executed.
Organization provisioning, suspend, reactivate, plan change, slug change, export and delete requests, single-user data requests.
Role create, permission change and delete. User role change, lock, unlock, delete and MFA reset. MFA enroll and disable. Email change and revert. Identity link and unlink.

Only security events. The audit log and its hash chain are for the events above. High-volume product events (progress, xAPI statements, AI usage, page views) go to their own partitioned tables and never through the chain.

Written in the same transaction. Every audit row for a state change is inserted inside the same unit of work as the change. If the audit insert fails, the change rolls back. A security change can never happen without its audit row.

Events with no tenant or user. A failed login has no authenticated subject. The row uses actor_type: anonymous, outcome: failure, the organization id if the slug resolved (otherwise null), and an HMAC of the lowercased email, never the email or password. These rows are written by AuditService.recordAnonymous() through the lms_audit_writer role, on the org:{id}:anonymous chain when the slug resolved. Failures whose slug did not resolve are counted per minute and IP range and written as one summary row per minute on the anonymous chain, so a flood with random slugs cannot serialize every request on one chain.

Tamper resistance.

Database privileges. lms_app has INSERT and SELECT on tenancy.audit_logs; lms_audit_writer has INSERT only. A trigger rejects every UPDATE and DELETE, even from the table owner, except the lms_retention role.
RLS policies on the audit tables. These are the only tenant tables where organization_id may be null, so they have explicit policies per role instead of the standard one:
lms_app, organization chain: INSERT and SELECT where organization_id = app.organization_id and chain_id = 'org:' || organization_id.
lms_app, platform chain: INSERT and SELECT where organization_id IS NULL AND chain_id = 'platform' and current_setting('app.platform', true) = 'on'. Only PlatformUnitOfWork sets app.platform.
lms_audit_writer: INSERT only, where actor_type = 'anonymous' and chain_id is org:{organization_id}:anonymous or anonymous.
audit_chain_heads: matching SELECT, INSERT and UPDATE policies for each of these role and chain combinations, because append() takes SELECT … FOR UPDATE on the head and updates it.
lms_retention: DELETE only, plus the trigger exemption.
Serialized hash chain. Each audit row belongs to one chain: org:{id}, org:{id}:anonymous, anonymous, or platform. AuditChainService.append() runs inside the caller's unit of work as its last statement, immediately before commit: lock the chain's audit_chain_heads row with SELECT … FOR UPDATE (creating it on first use), set seq = last_seq + 1 and prev_hash = last_hash, compute row_hash = SHA-256 of the row's canonical content plus prev_hash, insert the row, update the head. Two concurrent inserts on the same chain wait for each other for microseconds and can never share a prev_hash. Because the table is partitioned, its unique key (chain_id, seq, created_at) cannot stop a duplicate seq; the head-row lock prevents duplicates and the daily verification detects any. If chain contention ever shows in metrics, the fallback is to insert unchained rows in the transaction and let one worker per chain assign seq and hashes within seconds; the schema already supports it.
Verification. A daily job walks every chain in seq order, recomputes every hash, and alerts on any gap, duplicate or mismatch.
External copy. An hourly job exports new rows to object storage with object lock (write-once, compliance mode) for the retention period, in a separate account the app cannot delete from. Each export also records the chain heads, so a later deletion of the live rows can still be checked against it.

Retention. Security audit rows are kept at least 1 year (longer if a contract requires). Org Admins cannot shorten it below that. Expired months are removed by the ops pipeline dropping whole partitions, only after that month's write-once export is confirmed. lms_retention row deletes are used only for offboarding and for per-organization policies shorter than a month.

Monitoring. A failed audit insert is a metric and an alert. If the anonymous audit path fails, login still works, but the alert fires.

Never store passwords, tokens, prompts or request bodies in an audit row.

AI invariants (binding for Phase 4, nothing built now)

These decisions are cheap now and expensive after Phase 4. The rule behind all of them: the model is never trusted. It can be fooled by anything it reads, so everything it does passes through the same checks as an anonymous request.

One LLM gateway. All calls to AI providers go through one module (src/ai/gateway, later its own service verifying tokens through JWKS). It owns provider abstraction, the per-plan model allowlist, prompt and model versioning, response caching, redaction of personal and health data before sending, and usage metering per organization and per user. No other module imports a provider SDK or holds a provider key; a lint rule enforces it.
AI budget works like storage quota. Before a call, reserve the estimated cost against ai_monthly_budget_usd under FOR UPDATE on an organization_ai_usage row; after the call, commit the actual cost and release the rest. A platform-wide daily cap and provider-side hard limits sit behind it (system-design C11). Per-user rate limits sit in redis-cache. Every call has maximum input and output tokens and a timeout.
Embeddings in Postgres. Use pgvector in the ai schema, with RLS and composite foreign keys like every tenant table. Each chunk carries organization_id, its source (course, document) and the embedding model version. Retrieval filters by organization (RLS) and by the caller's access to the source (enrollment or authoring rights, through AuthorizationService.assert()), so a learner never retrieves from a course they cannot open. Embeddings are treated as sensitive (they can leak the text they came from): never returned by the API, deleted with their source.
Prompt injection. Uploaded and retrieved content is data, never instructions; it is placed in clearly delimited, labelled sections of the prompt. Organization-defined prompt settings can add guidance but can never replace the platform's safety instructions. Prompts never contain secrets, tokens, internal ids of other organizations, or data the user could not read themselves.
Tools and actions. The model can only call tools from a fixed allowlist. Every tool call runs through the normal authorization (@Permissions, assert()) as the requesting user, inside their organization; nothing the model says about identity, role or organization is trusted. Tools that change data or send messages require the user's explicit confirmation in the client. No tool fetches arbitrary URLs except through safe-fetch.ts with an allowlist.
Model output is untrusted. Chat markdown is rendered through a sanitizer on every client; links are shown with their real destination, and images from model output are only loaded from our own media domain. This blocks stored XSS and data exfiltration through image URLs.
Streaming revocation. SSE and WebSocket chat follow the long-lived connection rule in Part 3.
AI grading. A human reviews or can override every AI grade. The model, prompt version, inputs and output are stored so a grade can be reproduced and appealed.
Sensitive by default. Prompts, transcripts and model logs are encrypted with the organization's data key, have their own retention policy (ai_prompts, model_logs), and are sent only to providers under zero-data-retention terms in approved regions.
Moderation. Learner input and model output pass a content-safety check configurable per organization.
Tested like an attacker. The gateway has a prompt-injection and data-exfiltration test corpus (instructions hidden in documents, attempts to reveal other organizations' data or system prompts, tool-abuse attempts) that runs in CI and must pass before any AI feature ships.
LMS content invariants (binding for Phases 2–3, nothing built now)
Course access. Every course, lesson, file and media request is checked with AuthorizationService.assert() against enrollment or authoring rights (role_assignments), never only by knowing an id. A learner's own rows use owner scope.
Media. Video is reachable only through 15-minute signed CloudFront cookies scoped to one course's media path (/org/{organizationId}/course/{courseId}/…, cookie Path set to the same prefix), issued only after AuthorizationService.assert() on that course (system-design C6). A cookie for one course never opens another course's media, even in the same organization. Media URLs and keys contain random ids, never course or user names.
Certificates cannot be forged.
Each issued certificate is a signed record: the issuing service signs {certificateId, organizationId, learner name, course title, issuedAt, expiresAt} with a dedicated Ed25519 certificate key (separate from the JWT keys, with a kid, rotated yearly; old public keys stay published).
The PDF carries the signature and a QR code with a random verification_code.
GET /api/v1/certificates/verify/{code} (@Public, rate-limited) returns only what the certificate states, its validity, and whether it was revoked or expired; never other learner data. Codes are random (128 bits), so they cannot be guessed or enumerated.
Revocation and expiry are checked on every verification. Editing a certificate is impossible; a correction revokes it and issues a new one.
Completion and grades are computed on the server from recorded progress and graded attempts; a client can never set "completed", "passed" or a score.
Payment invariants (binding whenever paid courses or billing are built)
Prices on the server only. The amount, currency, discount and tax for an order are computed by the server from its own price list and coupon rules. Any price, total or discount sent by a client is ignored.
One payment provider integration (for example Razorpay for India), in one module, the only code that holds provider keys. Provider keys live in the secret manager.
Access only after verified payment. An enrollment or plan upgrade is granted only when the server has confirmed the payment with the provider: a signature-verified webhook, or a server-to-server status call. A client redirect or "payment success" message never grants anything.
Webhooks are verified and replay-safe. Signature verified with a constant-time comparison; the provider's event id is stored with a unique constraint, so a replayed or duplicated event is processed once; events older than the provider's tolerance window are rejected; processing is idempotent.
Orders follow a fixed state machine: created → pending → paid → fulfilled, with refunded and failed as the only other states. Each transition is a conditional update from the expected previous state, so two concurrent webhooks or retries cannot fulfil twice.
Coupons and discounts have usage limits enforced under a row lock, expiry, per-user limits and a maximum discount; a coupon cannot make an order free unless it is explicitly a 100% coupon.
Refunds are requested by an authorized role, recorded with a reason, executed through the provider, and revoke access in the same state machine. Every payment, refund and coupon change is audited.
Money is stored as integer minor units with a currency code (foundation §7.6); amounts are reconciled daily against the provider's settlement report, with alerts on any difference.
Card data never touches our servers. Payment pages and SDKs are the provider's (PCI DSS scope stays with the provider).
Target architecture and scale path
Clients (web / Android / iOS / desktop)
        │
   CloudFront + WAF ── load balancer (TLS)          admin-api. (IP allowlist / VPN)
        │
  NestJS API replicas (modular monolith, stateless)
        │          │              │              │
        │   redis-security   redis-queue    redis-cache
        │   (authz state,    (BullMQ,       (rate limits,
        │    MemoryDB)        noeviction)    cache, pub/sub)
        │                          │
        │                       workers (email, outbox, scan, export,
        │                        bulk import, embeddings, grading)
        │                          │
  PgBouncer (central) ── Postgres primary (+ read replica)    LLM gateway ── AI providers
        │       schemas: identity, tenancy, catalog, storage,
        │       learning, progress, assessment, ai (pgvector)
  S3 (private, org-prefixed, per-org KMS keys, versioned) · write-once audit bucket · KMS (multi-Region)
Modular monolith. One deployable NestJS app; module boundaries follow the Postgres schemas. A module talks to another only through its service interface, never its tables. Split a module into a service only when it needs independent scaling (the LLM gateway and workers are the first candidates); JWKS verification makes that possible without sharing secrets.
Scale path, in order:
Horizontal API replicas and workers (already possible: stateless, no sticky sessions).
Read replica for lists, reporting and lms_platform_reader (RLS applies there too).
Time-partition high-volume tables (audit_logs, progress and xAPI events, AI usage).
Move very large tenants to their own Postgres cluster, routed by organization_id, which every row already carries.
Multi-region only when a customer contract requires it, region pinned per organization.
Build order

Two phases. Phase 1a is the foundation everything else stands on, including the full revocation guarantee; Phase 1b is required before the first hospital contract. Nothing in 1b changes a 1a schema. RLS comes first, so every later table and test runs under it.  Each step uses only what earlier steps built. Finish each step's test before starting the next. Every step also meets the definition of done in foundation §14.

Phase 1a — Foundation
Stage	Step	Build	Done when
1 Database and platform	1	Freeze the MongoDB variant behind the DATABASE_TYPE startup check (Part 1); one DataSource; create the eight schemas; roles lms_migrator, lms_app, lms_worker, lms_security_watcher, lms_audit_writer, lms_retention, lms_platform_reader with their timeouts and grants	App connects as lms_app and jobs as lms_worker; a 6-second API query is cancelled; the app refuses to start with mongodb outside local development; no file under document/ is changed
	2	TenantUnitOfWork (API and worker variants), PlatformUnitOfWork with forOrganization(), tenant repository with scope modes and @OwnerColumn, global-tables.ts, set_config lint rule, migration check (FK, RLS, index prefix, partition-key uniqueness, security_invoker, SECURITY DEFINER)	A query outside a unit of work fails; global scope on a tenant table throws; CI blocks a table without RLS; a tenant unit of work cannot set app.platform
	3	API platform: response interceptor with headers, problem+json filter, error-codes.ts, scrubbed Sentry filter, signed cursor codec, idempotency_keys and interceptor, input-bound check, safe-fetch.ts, sanitize and CSV-escape helpers, trusted-proxy setup, module boundary lint rules, hygen generate:module template with its cross-tenant and authorization specs	A generated sample module passes its specs; an unhandled error returns problem+json 500 without a stack; a repeated Idempotency-Key replays; a DTO without bounds fails CI; safe-fetch refuses metadata addresses
	4	tenancy.organizations (tombstone, data-key and storage-key fields), organization_plans, organization_storage_usage, organization_slug_aliases, DataKeyService with KMS multi-Region keys	Unique slug across organizations and aliases; limits readable; an encrypted value round-trips
2 Infrastructure services	5	Three Redis clients (SECURITY_REDIS cluster-aware with per-node reads, QUEUE_REDIS, CACHE_REDIS) with the noeviction startup checks; BullMQ processor base class (signed payloads, TenantContext, organization re-check, per-organization semaphore, retries and dead-letter queue; the actor re-check is wired in step 7); email_outbox (global, grants), outbox relay and email worker with provider message tags	Startup fails when redis-security or redis-queue is not noeviction; a job written into redis-queue without a valid signature is rejected without running; an email written inside a rolled-back transaction is never sent; a committed one is sent and retried on failure
	6	audit_logs (partitioned) and audit_chain_heads with per-role policies, AuditChainService with transactional writes, AuditService.recordAnonymous() and anonymous summaries, ops-pipeline partition creation and drop	Concurrent inserts keep an unbroken chain; tampering detected; anonymous and platform rows insert under RLS; a forced audit failure rolls back its change
3 Identity and sessions	7	organization_roles, users, account_directory, user_consents, identity.sessions (users only, UUIDv7 ids), Ed25519 signing keys and JWKS (EdDSA only), bounded bcrypt pool, user email login with progressive delays and the consent challenge, safe refresh rotation with the organization in the refresh token, logout; wire the job actor re-check	Same email in two organizations = two accounts; a token signed with an unknown kid or another alg is refused; concurrent refresh does not log out; replay revokes the session; login takes the same time for existing and unknown accounts
4 Authorization and revocation	8	Middleware (token verification, per-node redis-security reads, permission resolution), AuthorizationGuard, decorators (including @NoSupportAccess and @RequireRecentAuth), route scanner, authenticated-allowlist.ts, compare-and-set cache, AuthorizationService.assert(), resolveScope(), stream tickets	CI fails on a handler with no authorization declaration; .own callers only see their own rows; checks pass against a cluster-mode Redis
	9	security_outbox (global, grants), NOTIFY trigger, revocation through redis-security, SecurityStateRebuildJob, sec:epoch and the middleware's sentinel check	Lock, role change and suspend stop the next request; with the sentinel deleted, authenticated requests get 503 until the rebuild restores deny keys
	10	Pending-security watcher (lms_security_watcher, direct LISTEN), freshness lease, 202 pending_enforcement	With Redis refusing writes, every serving replica rejects a locked user within the lease; a replica that can't sync returns 503 to authenticated requests while staying ready
5 Account flows	11	user_identities, identity_link_tickets, email_action_tokens, identity-provider interface and social login, verify email, forgot and reset (with session revocation), password policy and breach check, find-organization job, my sessions	An email match returns link_required; a breached or over-long password is refused; a reused reset link fails; a reset revokes every session; find-organization emails the right organizations
	12	user_recovery_codes, user MFA (TOTP) with challenges and attempt limits, re-authentication, change of email with revert, consent view and withdrawal, security notifications	Org Admin cannot get past enrollment without MFA; five wrong MFA codes end the challenge; an email change without recent re-auth returns 401 reauth_required
	13	platform_admins, platform_admin_sessions, platform_admin_recovery_codes, platform_admin_webauthn_credentials (table now, WebAuthn flow in 1b), admin login and refresh on the admin hostname, TOTP	No admin tokens without a valid MFA code; admin sessions live only in their own table
6 Read path and organization lifecycle	14	Read path and fairness: @Cacheable interceptor with canonical keys and generation counters, @Sensitive and resource-scope static checks, request-scoped BatchLoader, query-budget helper, per-organization rate limits and concurrency caps, job semaphores, load shedding	Same request in org A and org B never shares a cache entry; one organization over its limit gets 429 while another is served; overload returns 503 quickly
	15	Provisioning through forOrganization(), single invitations, bounce webhook	Same idempotency key creates one organization; a hard bounce marks the invitation
	16	Seat limits on every activating path, suspend and reactivate, slug changes, MFA reset by Org Admin	Concurrent accepts and unlocks never exceed the seat limit
7 Operations baseline	17	Logs, traces, metrics, alerts, health endpoints, graceful shutdown	Alerts fire in a test; readiness stays up with Postgres down; a deploy loses no in-flight request
Phase 1b — Before the first hospital contract
Stage	Step	Build	Done when
8 Hardening	18	support_sessions (server-bound Redis state, support permissions, Org Admin approval, notifications), platform_approvals and the two-person rule, last-Org-Admin and platform-admin MFA recovery	A support token is refused the moment its session ends; an admin cannot approve their own request
	19	WebAuthn for platform admins; write-once audit export and daily verification	Every platform admin has a WebAuthn credential; export holds chain heads
	20	Web refresh cookie (httpOnly, SameSite=Strict, Origin check, CSRF double-submit); CAPTCHA thresholds on public auth routes	The web app never holds a refresh token in script-readable storage; a cross-site refresh is refused
9 Data lifecycle	21	storage.files, reservations, signed URLs with per-organization KMS keys, download domain, magic-byte check, isolated scanning and metadata stripping, remove multer path; bulk invitations from an uploaded CSV file (Part 5)	Concurrent uploads never exceed the storage limit; an infected file is never served; a 10,000-row bulk import respects the seat limit
	22	data_requests (organization and single user, with approval), retention_policies, deletion certificate, key destruction	One organization deleted; only its tombstone and write-once audit export remain, and its keys are scheduled for destruction
10 Production operations	23	Backups and DR copies, restore drill, break-glass access, CI/CD security checks and image signing, threat model, external penetration test, load test and failure drills (system-design C9)	Restore drill passes; pentest findings triaged; load test meets the C1 targets

Required tests

Phase 1 is done only when these e2e tests pass against PostgreSQL (real roles and RLS) and all three Redis instances, with redis-security in cluster mode. Each test gates the phase that builds its feature: tests 9, 22, 24, 25, 68, 69, the WebAuthn part of test 8 and the bulk-import part of test 20 gate Phase 1b; test 82 gates Phase 3 and test 83 gates any paid feature; everything else gates Phase 1a.

Cross-tenant matrix: create org A and org B. As each role in A, call every route against B's data: 404 or 403. Every later module copies this test.
Authorization declarations: every handler has exactly one of @Public, @Permissions, @PlatformAdmin, @Authenticated; a handler with none is denied and fails the route scanner.
Every non-public route returns 401 without a token, built from the router.
Same email in org A and org B: two accounts, two passwords; logging into A never returns B's data.
Social linking: a changed email at the provider still finds the same user; an unknown Google identity whose verified email matches an existing account returns link_required and creates no link; the link is created only after the user signs in and uses the ticket; an expired or reused ticket fails; a provider token with the wrong audience is rejected.
Wrong organization slug, wrong email, wrong password and locked account all return the same 401, with response times within a small margin of each other, including after 5 or more failures, when the progressive delay applies equally to existing and non-existing accounts.
A user token is rejected on admin routes and on the admin hostname; an admin token cannot call the user login.
Platform admin: no tokens without MFA; a reused TOTP code is rejected; a recovery code works once; a WebAuthn assertion for another admin, or with a non-increasing sign count, is rejected.
Support sessions: read permissions work without approval; write permissions do nothing until an Org Admin approves; @NoSupportAccess routes are always refused; a token is refused when its Redis entry is ended, revoked, expired, or names a different admin or organization; editing a token's claims cannot add permissions; Org Admins are notified.
A refresh token sent as an access token is rejected, and the reverse. A token with a wrong iss, aud or unknown kid is rejected.
Two simultaneous refreshes keep the user logged in. An old refresh token after the grace window revokes the whole session.
Lock, role change, password reset and delete stop the next request. Suspending an organization stops every user in it. After a password reset, an old refresh token is detected as a replay.
Barrier: with Redis refusing writes, locking a user returns 202 pending_enforcement, and a request with the old token to another replica is refused within the lease.
Freshness lease: block one replica's access to Postgres, or drop its LISTEN connection; within the lease it returns 503 to authenticated requests while public routes and /health/ready keep working, and the lease-expiry alert fires; it serves again only after a full re-read.
With redis-security unreachable, authenticated requests get 503. Deleting every authz:ver:* key does not revive a revoked token. Filling redis-cache to its memory limit does not affect authorization or jobs.
Stale cache after expiry: expire role:perms:{id} while a role change is committing, with a refill read started before the commit; no request is ever authorized with the old permission set.
No replica keeps permissions in memory: a static check fails if any module caches permission or support state outside Redis.
No escalation: a caller cannot grant a permission they don't hold, edit their own role, or assign Org Admin without being one.
Two concurrent demotions of the last two Org Admins: exactly one succeeds.
Provisioning: the same idempotency key twice creates one organization, one data key and one storage key; a failed email is retried from the outbox; all tenant rows are written under RLS.
Seats: many concurrent invitation accepts, unlocks, bulk-import rows and self-registration confirms never exceed the seat limit.
Storage: many concurrent upload reservations never exceed the storage limit; an upload larger than declared is rejected and released; a file whose magic bytes don't match its type is rejected; an abandoned reservation is released when it expires; reconciliation matches the provider's totals.
Audit: forcing the audit insert to fail rolls back the change; UPDATE and DELETE fail for lms_app; 100 concurrent inserts on one chain produce consecutive seq values and a valid chain; a hand-edited row and a duplicate seq are caught by verification; a failed login writes an anonymous row on the anonymous chain.
Files: an org B user cannot get a signed URL for an org A file; an unscanned or infected file is never served; downloads are served from the download domain as attachments; a stored image has no EXIF or GPS data.
Offboarding: deletion is refused without an approved platform approval and until the audit export is complete; afterwards no row or file with that organization_id remains except the tombstone; both keys are scheduled for destruction and an encrypted value from a backup can no longer be decrypted; the slug cannot be reused for 90 days; the deletion certificate lists what was retained.
Database layer: with the app's checks bypassed, a tenant-aware FK rejects a cross-organization link, and RLS returns no rows when app.organization_id is missing or different. A view without security_invoker is rejected by the migration check.
No API unit of work stays open longer than 5 seconds, and no job unit of work longer than 60 seconds, during the whole test run; the database cancels one that tries.
Rate limits trigger across two API processes sharing redis-cache. /health/ready stays 200 when Postgres or any Redis is down, the lease has expired or the sentinel is missing (authenticated requests get 503 instead), and is not ready only while startup is incomplete.
User MFA: an Org Admin without MFA can only reach enrollment; a reused TOTP step is rejected; a recovery code works once; requireMfaForAll applies to every user at next login.
Re-authentication: a role change, invitation, user lock, identity unlink, email change or export without recent re-authentication returns 401 reauth_required; it succeeds after re-auth within the window.
Enumeration: register with an unknown slug, a closed organization, an existing email and a new email all return the same response in similar time; find-organization and forgot return the same response for any email.
Passwords: a password under 12 characters, over 72 UTF-8 bytes, or found in the breach list is refused; the breach API being down does not block registration.
My sessions: a user can list and revoke their own sessions, and a revoked session stops the next request; a session id from another user or organization returns 404; the 21st login revokes the least recently used session; support sessions are refused on these routes.
Worker re-check: a job queued for an organization that is then suspended does not run, and a job for a locked actor does not run.
Single-user deletion: a user_delete request removes or anonymizes that user's rows and directory entry, leaves other users untouched, and keeps audit rows with an actor id that no longer resolves.
Self-registration: a newly registered user cannot log in until the email is confirmed.
Response format: every error response is application/problem+json with the real HTTP status and a code from error-codes.ts; no error is returned with status 200; production mode never includes a stack trace or SQL.
Cache isolation: the same cached handler with the same parameters returns each organization's own data; a role permission change or module write makes old entries unreachable; an owner-scoped entry is never served to another user; @Cacheable on a @Sensitive response fails the static check.
Batching: concurrent requests from org A and org B never share a batch query; list endpoints stay within their query budget.
Scope modes: a caller with x.read.own sees only their own rows (another user's row returns 404); x.read.any sees the organization's rows; global scope on a tenant table throws.
Cursors: a tampered or expired cursor returns 400 invalid_cursor; a cursor issued in org A returns 400 in org B; a cursor signed with the previous secret still works after rotation.
Idempotency: the same key and body replays the stored response; the same key with a different body returns 422 idempotency_key_reused; keys are isolated per organization and actor.
Module boundaries: lint fails when a module imports another module's entity or repository, when code outside the LLM gateway imports a provider SDK, or when code outside safe-fetch.ts makes an outbound HTTP call.
Partitioned audit: rows in two different months keep one valid chain; the ops pipeline can drop an exported month's partition; lms_app, lms_worker and lms_retention cannot create, drop or detach partitions.
Audit under RLS: a failed login with an unknown slug is counted into a summary row on the anonymous chain through lms_audit_writer; lms_audit_writer cannot write a non-anonymous row, read audit rows, or touch an organization chain; a platform-admin action writes to the platform chain only inside PlatformUnitOfWork; a tenant unit of work cannot write to the platform chain.
Emailed tokens: reusing a verify, reset, change-email or revert link fails; two concurrent clicks consume it once; a newer reset link revokes the older one; expired tokens fail with the same generic 400.
Change of email: without recent re-auth it returns 401 reauth_required; the address changes only after the new address confirms; the old address is notified and its revert link restores the old address and revokes all sessions; a deactivated user stays deactivated after confirming; confirming an address another account took meanwhile returns 409 email_taken.
Long jobs: an export of a large organization finishes in batches with no unit of work over 60 seconds; a cross-tenant job opens one unit of work per organization and never reads two organizations in one.
Fairness: an organization over its API limit or concurrency cap gets 429 org_rate_limited while another organization's requests succeed; one organization's job backlog does not delay another organization's jobs beyond its semaphore limit.
Request-order keys: cache and idempotency keys ignore undeclared query parameters and key order; 4xx responses are never cached or stored; a @Sensitive response is never stored in idempotency_keys; @Cacheable({ scope: 'organization' }) on a handler that calls assert() with a resource fails the static check.
Slug aliases: after a rename, login with the old slug works for 90 days and returns the current slug, then fails like an unknown slug; no other organization can claim the old slug during that window.
Email delivery: a hard-bounce webhook marks the outbox row and the invitation and notifies the inviter; a webhook with an invalid signature is rejected.
Outbox access: the watcher, connected as lms_security_watcher, sees unprocessed keys from every organization; lms_app cannot read another organization's outbox payloads or email addresses; the watcher role cannot read any other table.
Admin sessions: a platform admin login creates a row in platform_admin_sessions and none in identity.sessions; admin refresh and replay detection work; identity.sessions rejects a row without organization_id.
Refresh routing: refresh opens the unit of work for the organization in the refresh token; a refresh token with a forged organization claim fails signature verification.
Platform units of work: provisioning, suspend and support-session writes succeed through forOrganization() under RLS; the same writes through plain PlatformUnitOfWork.run() are rejected by RLS.
Directory: account_directory holds only HMACs; it is updated on user create, email change, lock and delete; find-organization never reveals results in the response.
Redis cluster: the middleware's checks work with keys on different nodes; one node timing out returns 503, never a partial pass.
Sentinel: flushing redis-security makes every authenticated request 503 until the rebuild job finishes; afterwards a suspended organization and a logged-out session are still refused.
Queue safety: startup fails when redis-queue is not noeviction; filling redis-cache loses no job; jobs lost in a simulated redis-queue failover are re-enqueued from the outboxes.
Per-organization keys: an object written for org A is encrypted with org A's storage key; scheduling that key's deletion makes the object and a backup copy unreadable; a disaster-recovery restore can decrypt data while the key exists.
Trusted proxy: a request with a forged X-Forwarded-For is rate-limited and audited under its real client IP.
Headers: every authenticated response has Cache-Control: no-store, HSTS, nosniff and Referrer-Policy: no-referrer; CloudFront never caches an API response.
No tokens in URLs: email links carry the token only in the fragment; a stream ticket works once, within 30 seconds, only for its session and path; access logs never contain a token.
MFA attempts: five wrong codes delete the challenge; ten failures in an hour add the account delay and alert; a challenge cannot be used after success.
Brute force: after 5 failed passwords the account's attempts are delayed and never hard-locked; the owner is emailed after 10; CAPTCHA is required after the threshold.
SSRF: safe-fetch.ts refuses private, loopback, link-local, metadata and IPv6 equivalents, redirects to them, DNS answers that change between resolve and connect, plain HTTP, and oversized responses.
Two-person rule: organization export and delete, last-Org-Admin MFA reset and platform-admin MFA reset do not run without a second admin's approval; the requester cannot approve; an approval expires after 24 hours.
MFA recovery: an Org Admin can reset another user's MFA but not their own; the user is notified and all their sessions are revoked; the last-Org-Admin recovery waits 24 hours and notifies the organization.
Input bounds: oversized strings, arrays, nesting depth and bodies return 422 or 413; a DTO field without a bound fails CI.
Content safety: stored rich text loses script, event handlers and javascript: URLs; CSV exports and bulk-import reports neutralize formula cells; download file names are safely encoded.
Consent: tokens are not issued until the current notice is accepted; a new notice version requires acceptance at next login; withdrawal is recorded and audited.
Security notifications: new-device login, password change, MFA disable, identity link, support session start and Org Admin changes send the right emails, without secrets or one-click actions.
Token algorithms: a token with alg: none, HS256, or RS256 signed with the public key is rejected; a token with nbf in the future beyond 30 seconds is rejected.
Graceful shutdown: during a rolling deploy under load, no request fails and SSE clients reconnect.
Load shedding and hashing: a burst of logins does not raise p95 latency for authenticated reads beyond target; over-capacity requests get 503 overloaded quickly.
Database switch: the app refuses to start with DATABASE_TYPE=mongodb in staging or production, or in development without ALLOW_DOCUMENT_DB=true; with postgres it starts normally.
Response allowlists: every response matches its response DTO exactly; a test fails if any route returns a field not declared on its DTO (for example a password hash, authz_version or another organization's id).
Injection: a JSON body with __proto__, constructor or prototype keys is rejected with 422; a lint rule fails on child_process.exec or spawn with shell: true anywhere in the code.
Signed jobs: a job inserted directly into redis-queue without a valid signature, or with a payload changed after signing, is rejected and alerted without running.
Webhook replay: the same signed bounce or payment event delivered twice is processed once; an event with a timestamp outside the tolerance window is rejected.
Certificates (Phase 3 gate): a certificate with a changed field fails signature verification; a revoked or expired certificate verifies as such; random codes cannot be enumerated (rate limit and 128-bit codes).
Payments (gate for any paid feature): a client-supplied price is ignored; access is never granted from a client redirect; two concurrent webhooks for one order fulfil it once; a coupon over its usage limit is refused.
After Phase 1

Not in this plan. The seams above keep each from becoming a rewrite:

Phase 2: courses, sections, lessons, batches, instructors, enrollments, progress, completion rules, publishing; video (system-design C6); the MedTrain catalog with licenses and fork-on-edit; course- and batch-scoped roles through role_assignments; push notification tokens.
Phase 3: versioned assessments and questions, attempts, rubrics, grading reviews, certificates.
Phase 4: AI documents, versioned embeddings, secure retrieval, tutor chat, jobs, prompt versions, usage and quotas, built on the AI invariants above.
Phase 5: live sessions, attendance, SCORM/xAPI (packages served from a separate sandboxed origin), SSO/SAML and SCIM, HR integrations, signed outbound webhooks, analytics, billing, WebAuthn for organization users.