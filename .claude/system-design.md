# MedTrain LMS — System Design and Operations

Version 3 · Oct 24, 2026 · @Bangalore MedTrain LLP

The infrastructure, capacity, client and operations decisions that sit around the code. foundation.md says how modules are built; plan.md says what Phase 1 builds; this document says where it runs, how big it must be, how clients use it, how it is operated, and which risks are accepted.

Each section lists the decision, a recommended default (marked proposed until someone with authority confirms it), and when it is needed. Defaults assume AWS, because the codebase already uses the AWS SDK for S3; change them in one place here if the provider changes.

Changes in v3: load balancer routing algorithm; MongoDB only in local development; no secrets in mobile app binaries; new residual-risk row for the frozen MongoDB variant.

Changes in v2: three Redis instances (MemoryDB for security state in cluster mode, separate noeviction queue Redis), central PgBouncer, per-organization multi-Region KMS keys, CloudFront never caching the API, malware scanning service, short-lived media cookies, stream tickets, access management and break-glass, vulnerability management, cross-platform client rules (carrier NAT, deep links, offline encryption, attestation, localization), and C13 accepted residual risks.

| Needed by | Sections |
|---|---|
| Before Phase 1a step 1 | C1 (first estimate), C2 (provider and region), C3 |
| During Phase 1a | C4, C5, C8, C10 (access management), C12 (token handling) |
| Before Phase 2 design | C6, C7 |
| Before the first hospital contract (Phase 1b step 23) | C1 (confirmed), C9, C10, C11, C13 (signed off) |

## C1. Requirements and targets

Without numbers, pool sizes, replica counts and partitioning are guesses. Fill in the Confirmed column from sales forecasts and the first contracts; until then the proposed values drive sizing.

| Metric | Proposed (year 1) | Proposed (year 3) | Confirmed |
|---|---|---|---|
| Organizations | 50 | 500 |  |
| Users per organization (median / largest) | 500 / 20,000 | 1,000 / 100,000 |  |
| Total users | 100,000 | 1,000,000 |  |
| Peak concurrent users | 5,000 | 50,000 |  |
| Peak API requests per second | 500 | 5,000 |  |
| Share of traffic from mobile apps | 60% | 70% |  |
| Stored files and video | 5 TB | 100 TB |  |
| Video hours streamed per month | 50,000 | 1,000,000 |  |
| AI requests per day (Phase 4) | 20,000 | 500,000 |  |

Service levels (proposed):

- Availability 99.9% per month for the API (about 43 minutes of downtime), measured at the load balancer, excluding announced maintenance.
- p95 latency: authentication 300 ms, single-resource reads 150 ms, lists 300 ms, writes 400 ms, AI first token 2 s (streamed). Measured at the server; mobile clients on slow networks add their own latency.
- Error budget: when a month's budget is spent, feature releases pause until reliability work restores it.
- RPO 15 minutes, RTO 4 hours (plan Part 8), adjusted to contracts.
- Revocation bound: immediate normally, at most SECURITY_LEASE_MS (2 s) in the worst case (plan Part 5).

## C2. Cloud and infrastructure

| Concern | Proposed default |
|---|---|
| Provider and region | AWS ap-south-1 (Mumbai) primary; disaster-recovery copies in ap-south-2 (Hyderabad), so all data stays in India |
| Compute | ECS on Fargate for the API, workers and PgBouncer (switch to EKS only if Kubernetes skills and needs justify it). Task roles with least-privilege IAM per service |
| Postgres | RDS for PostgreSQL 16+, Multi-AZ, with a read replica; pgvector, pg_trgm, btree_gin, pg_stat_statements enabled; storage encrypted with KMS; automated backups copied to ap-south-2; rds.force_ssl = 1 |
| Connection pooling | PgBouncer in transaction mode as one central ECS service with 2 tasks behind an internal load balancer, so pool sizes add up (C5). RDS Proxy is not used, because the watcher needs a direct connection for LISTEN and we need control over pooling mode. The watcher connects to RDS directly |
| redis-security | Amazon MemoryDB (durable transaction log, Multi-AZ, noeviction, TLS, ACL users). MemoryDB always runs in cluster mode, so the middleware reads keys per node (plan Part 3) and every Lua script touches one key |
| redis-queue | ElastiCache (Valkey or Redis OSS), replica in a second AZ, noeviction, TLS. BullMQ queue names use hash tags. Jobs that must not be lost are also in Postgres outboxes, so a failover that drops recent jobs is repaired by re-enqueueing |
| redis-cache | ElastiCache (Valkey or Redis OSS), allkeys-lru, TLS. Rate limits, response cache, real-time pub/sub. Losing it never affects correctness |
| Keys and secrets | KMS: a platform key per environment, plus two multi-Region keys per organization (data key wrapping and S3 storage), each with a replica in ap-south-2. Secrets Manager for all secrets and signing keys, with automatic rotation for database passwords |
| Object storage | S3 (private buckets, Block Public Access on, versioning, organization prefixes, SSE-KMS with the organization's key, bucket keys enabled to reduce KMS calls), replicated to ap-south-2; a separate AWS account for the write-once audit bucket with Object Lock in compliance mode |
| Malware scanning | Amazon GuardDuty Malware Protection for S3 on the upload bucket (verify regional availability; fallback: ClamAV in an isolated worker task with no database access). Results update storage.files through the scan worker |
| Load balancer | Application Load Balancer across 3 availability zones, least outstanding requests routing (not round robin, because SSE streams and AI responses vary widely in length), cross-zone load balancing on, deregistration delay 30 seconds to match graceful shutdown, idle timeout 120 seconds for streams, TLS policy allowing TLS 1.2+ only |
| CDN and edge | CloudFront with AWS WAF (managed common, known-bad-inputs and IP reputation rules, plus rate-based rules) in front of the API, web app, download domain and video. The API behaviour has caching disabled and forwards Authorization; only the JWKS and minimum-client-version documents are cacheable. Origins accept traffic only from CloudFront (origin header secret plus security group restrictions). AWS Shield Standard |
| CAPTCHA | A provider with web, Android and iOS SDKs and an India-friendly privacy posture (proposed: Cloudflare Turnstile or hCaptcha), used only after thresholds (plan Part 5) |
| Email | Amazon SES in ap-south-1 with SNS bounce and complaint notifications; message tags carry organization and outbox ids |
| Network | One VPC per environment; databases, Redis, PgBouncer and workers in private subnets with no public endpoints; only the load balancer and CloudFront are public; VPC endpoints for S3, KMS, Secrets Manager and ECR; outbound internet only through a NAT gateway with egress logging (and egress filtering where practical) |
| DNS and TLS | Route 53 with DNSSEC; ACM certificates; hostnames api., admin-api. (IP allowlisted, separate WAF rules), files. (download domain), media. (video), app.; TLS 1.2+ only; HSTS preload once stable |
| Infrastructure as code | Terraform, one state per environment, applied only from CI with review; no console changes in production (drift detection alerts) |

## C3. Environments

- dev (shared integration), staging (production-like, same Terraform), production. Local development uses docker compose: Postgres with the real roles and RLS, redis-security as a 3-node cluster, redis-queue and redis-cache, MinIO for S3, MailHog. A MongoDB container is optional and only for the frozen document variant (DATABASE_TYPE=mongodb with ALLOW_DOCUMENT_DB=true); dev, staging and production run Postgres only, and the app refuses to start otherwise (plan Part 1).
- Staging never holds real customer data. Seed data comes from a generator script; production data is never copied down, not even anonymized, without a documented exception approved by the security owner.
- Each environment has its own AWS account, its own KMS keys, its own JWT signing keys, CURSOR_SECRET and directory HMAC secret, and its own email sending identity. A token from staging can never verify in production.
- Seed scripts create two organizations with every role so the cross-tenant tests and manual testing always have data.
- AI coding agents and developers' machines only ever hold development credentials (foundation §16).

## C4. Deployment and release

- Pipeline: pull request → CI (plan Part 8 checks) → human review (CODEOWNERS for security paths) → merge → build one container image → scan and sign → deploy to staging → automated smoke and e2e tests → manual approval → production.
- Deploy strategy: rolling deploys on ECS with health checks (local-only readiness, plan Part 8) and graceful shutdown (25-second drain, SSE reconnect); blue-green for releases with risky migrations.
- Migrations: expand → deploy → migrate data (as a job) → contract, each step a separate release. Migrations run as lms_migrator in the pipeline before the new app version starts, use lock_timeout with retry, and must work with the previous version still running.
- Feature flags: a simple flags table or service, evaluated per organization, so features can be enabled for pilot hospitals first. Flags never gate security checks.
- Rollback: redeploy the previous signed image; because migrations are expand-only until contract, rollback never needs a down-migration.
- Release cadence: proposed weekly to production, with hotfixes any time.
- Mobile releases: the API supports every app version at or above the published minimum (C12); a breaking API change ships as /api/v2 and the old version stays at least 6 months.

## C5. Sizing

Derived from C1; revisit after the load test (C9).

- API replicas: start with 3 (one per availability zone), autoscale on CPU and p95 latency; target 300–500 requests per second per replica for typical reads. Each replica caps in-flight requests (proposed 500) and sheds load beyond it (foundation §6.4).
- bcrypt pool: proposed 4 worker threads per replica; login throughput is bounded by design.
- Connections: api_replicas × pool_per_replica + worker_replicas × pool_per_worker ≤ PgBouncer client limit; PgBouncer default_pool_size ≈ 2–4 × Postgres vCPUs, per database user (lms_app, lms_worker get separate pools); Postgres max_connections comfortably above PgBouncer's server connections plus one watcher connection per replica plus admin headroom. Proposed start: pool 10 per replica, PgBouncer server pool 40, max_connections 200.
- Postgres instance: proposed start db.r6g.large (2 vCPU, 16 GB) Multi-AZ, storage autoscaling on; scale vertically first, then read replica, then partitioning (plan scale path).
- Read replica lag: reads that tolerate staleness only; alert above 5 seconds, stop routing to the replica above 30 seconds.
- Workers: separate services per queue class (email and outboxes, files and scan, export and bulk import, AI); concurrency set per queue; AI workers scale on queue depth.
- Redis memory: redis-security holds small keys; proposed MemoryDB db.t4g.small shards with replicas, alerts at 70%. redis-queue proposed cache.t4g.small; redis-cache proposed cache.t4g.medium.
- KMS: request rates (unwraps are cached in memory for 5 minutes; S3 bucket keys reduce SSE-KMS calls) stay well under the regional quota at year-3 volume; alert at 50% of quota.

## C6. Video and media

An LMS lives on video; these choices shape the storage, CDN and security design even though video is built in Phase 2.

- Upload: large files use S3 multipart upload with presigned part URLs (a single presigned POST is limited to 5 GB). The storage quota reservation (plan Part 7) covers the declared total size; the part URLs are issued only for that reservation.
- Transcoding: AWS Elemental MediaConvert jobs triggered by the files worker after the malware scan, producing HLS with an adaptive bitrate ladder (proposed 240p for weak mobile networks, 360p, 540p, 720p, 1080p) and thumbnails. Captions (WebVTT) are stored alongside. Output is written under the organization's prefix with its KMS key.
- Delivery: CloudFront on the media. hostname with signed cookies scoped to one course's path (/org/{organizationId}/course/{courseId}/…, with the cookie Path set to the same prefix so cookies for several courses coexist), valid for 15 minutes, issued and refreshed by the player through an API call that runs AuthorizationService.assert() on that course. A cookie for one course never opens another course's media. A revoked user therefore loses media access within 15 minutes, matching the access token lifetime. Segments are never public. Because cookie names are fixed, one browser streams for one account at a time; apps use one account at a time anyway.
- Quota: renditions count toward the organization's storage quota; originals are kept (proposed) for re-transcoding and also count.
- DRM: not by default. Short-lived signed cookies plus visible watermarking (user id and time, burned in or overlaid) cover most institutional content. If licensed third-party medical content requires DRM, add multi-DRM (Widevine, FairPlay, PlayReady) through a vendor for that content only.
- Offline video on mobile: only where the content license allows; downloads are encrypted at rest in app storage with a key held in the OS keystore, expire with the license, and are deleted on logout or when the app learns the session was revoked.
- Progress tracking: the player reports watch progress in batches (every 30 seconds, and on pause or exit) into the partitioned progress events table, with Idempotency-Key so offline replays never double-count.

## C7. Real-time

- Transport: SSE for AI tutor streaming and notifications (simple, works over HTTP/2 and proxies); WebSocket only for live sessions that need two-way traffic.
- Authentication: native apps and modern browsers send Authorization on a fetch-based stream; browser EventSource and WebSocket use single-use stream tickets (foundation §7.7). Tokens never appear in stream URLs.
- Fan-out across replicas: Redis pub/sub on redis-cache (a separate redis-realtime if load grows), with channels per organization and per user. Messages carry ids only where possible and are re-authorized against the receiving connection's context before sending. Pub/sub is best-effort; clients resume from the last event id and fetch missed state over the API.
- Revocation: long-lived connections re-check every 30 seconds and on matching security notifications (plan Part 3).
- Limits: a per-replica connection cap (proposed 5,000), a per-user connection cap (proposed 5), per-organization caps (foundation §6.4), and a per-connection message rate limit for WebSocket.
- Load balancer: ALB idle timeout raised to 120 seconds; server heartbeats every 30 seconds; clients reconnect with backoff and jitter and resume from the last event id. Mobile apps close streams when backgrounded and reconnect on resume.

## C8. Observability backend

- Collection: an OpenTelemetry Collector in the same region receives traces, metrics and logs from every service. Scrubbing happens in the application and again in the collector (drop authorization, cookie, token-like values, email addresses).
- Storage, keeping data in India: CloudWatch Logs and Metrics plus AWS X-Ray in ap-south-1 (proposed), or a self-hosted Grafana stack (Loki, Tempo, Mimir) in the same VPC. Any SaaS vendor must store data in India or receive only scrubbed, non-personal data.
- Retention (proposed): logs 30 days searchable plus 1 year archived in S3 (encrypted, access-logged); traces 14 days; metrics 13 months; security audit logs per plan.
- Errors: Sentry self-hosted or a region-approved instance, scrubbed (foundation §9).
- Dashboards: golden signals per service, security metrics (plan Part 8), per-organization usage, queue depths, outbox lag, lease and sentinel state, cost.
- Infrastructure audit: AWS CloudTrail in every account (organization trail, log file validation, delivered to the security account), AWS Config rules for public buckets, unencrypted storage and open security groups, GuardDuty for threat detection.

## C9. Testing beyond correctness

- Load test (k6 or Artillery) against the C1 targets in staging before the first hospital contract and before each major release: login bursts (including a credential-stuffing pattern), list-heavy browsing, video start, concurrent quiz submission, bulk invitation, AI chat streaming. Pass criteria are the C1 latency targets at 1.5 × peak, with load shedding returning 503 rather than timeouts beyond that.
- Failure drills in staging, written up each time: Postgres Multi-AZ failover, redis-security node failure and full data loss (sentinel and rebuild), redis-queue failover (outbox re-enqueue), redis-cache loss, a lost LISTEN connection, an expired lease, an AI provider outage, an S3 outage on the download path, a KMS throttling event, a full region outage restored from ap-south-2.
- Security testing: external penetration test yearly and after major changes, covering web, both mobile apps and the API; automated DAST against staging on each release; DTO fuzzing in CI (foundation §13); dependency and container scanning on every build; from Phase 4, the prompt-injection corpus.
- Mobile app security review: OWASP MASVS-based review of each app before first release (secure storage, no tokens in logs, deep link handling, no debug builds in stores).

## C10. Operations

Access management:
- AWS access only through AWS IAM Identity Center (SSO) with MFA; no long-lived IAM user keys. Least-privilege permission sets per role; production write access limited to the deploy pipeline.
- No standing human access to production data. Database access is break-glass only: an engineer requests access with a reason and ticket, a second engineer approves, access is time-limited (1 hour), every session is recorded, and the organization's audit evidence notes it when customer data was viewed.
- Quarterly access review of AWS, GitHub, Sentry, email provider and platform admin accounts; leavers removed the same day.
- Vulnerability management: fix critical findings within 7 days, high within 30, medium within 90; exceptions need security-owner sign-off. A security.txt and a responsible-disclosure address are published; reports are acknowledged within 2 working days.
- Secret and key rotation: database passwords rotate automatically; JWT signing keys every 90 days; cursor and directory HMAC secrets yearly; immediate rotation on suspected exposure (runbook).
- Runbooks (in the repository, reviewed in drills): Redis security instance down or data lost, lease expiries, outbox backlog, refresh replay spike, WebAuthn anomaly, RLS violation alert, Postgres failover, email bounce spike, AI provider outage, suspected credential compromise, suspected breach, emergency signing-key rotation, organization suspension on request, lost MFA of the last Org Admin, break-glass access.
- On-call: a weekly rotation with a primary and a secondary; paging only for alerts with a runbook.
- Severity levels: SEV1 (data exposure, full outage), SEV2 (major feature down or a customer degraded), SEV3 (partial degradation), SEV4 (minor). Each has a response time and an update cadence.
- Breach notification (confirm every timeline with legal counsel):
- CERT-In: report cyber incidents within 6 hours of noticing them (CERT-In directions of April 2022), and keep ICT system logs for 180 days in India as those directions require.
- DPDP Act and Rules: MedTrain, as processor, informs the affected customer (the data fiduciary) without delay as the DPA requires; the fiduciary notifies the Data Protection Board and affected people within the timelines in the Rules.
- A named incident commander decides; legal and the DPO approve outbound notices; every step is logged.
- Post-incident: blameless review within 5 working days for SEV1 and SEV2, with tracked actions.

## C11. Cost

- Model: a spreadsheet per environment with compute, RDS, MemoryDB and ElastiCache, S3, CloudFront egress, MediaConvert minutes, KMS keys and requests, malware scanning, SES, observability and AI tokens, expressed as fixed cost plus cost per 100 organizations and per 1,000 active users. Revisit monthly against the bill.
- Per-organization KMS keys: two keys per organization at about USD 1 per key per month each (plus replicas in the DR region), so about USD 4 per organization per month before request charges; plan pricing covers it.
- Tagging: every resource tagged with environment and component; per-organization cost for storage, video and AI comes from the application's own metering.
- AI guardrails beyond per-organization budgets: a platform-wide daily spend cap enforced in the LLM gateway, provider-side hard limits on each API key, and an anomaly alert when hourly spend exceeds twice the trailing average.
- Video: CDN egress and transcoding minutes are usually the largest variable cost; plan pricing should cover them.

## C12. Clients (cross-platform)

One API, four kinds of client. These rules keep them secure and usable on real networks.

- Web: single-page app on app.; tokens in memory until the httpOnly refresh cookie ships in Phase 1b step 20; strict CSP (no inline script, allowlisted origins, frame-ancestors 'none'); single-flight refresh; rich text and AI markdown rendered through a sanitizer, never as raw HTML.
- Mobile (Android, iOS) and desktop: refresh token in the OS secure store (Keychain, Android Keystore with hardware backing where available); never in logs, crash reports, screenshots or backups (excluded from cloud backup). Single-flight refresh with the 409 retry handling from plan Part 5. The organization's current slug is updated from login responses (slug aliases, plan Part 5).
- Shared networks: many users arrive from one IP (carrier-grade NAT on Indian mobile networks, hospital gateways). Clients must not treat 429 as fatal: they retry after Retry-After with jitter. Server limits key on account and organization, not only IP (foundation §6.4).
- Deep links: email links (verify, reset, invitation, change email) use verified App Links (Android) and Universal Links (iOS) on app., with the token in the URL fragment. The app reads it locally and posts it; if the app is not installed, the web page does the same.
- Low bandwidth: responses are compressed (Brotli or gzip at CloudFront); lists are paginated (default 20, max 100); images and video have mobile-sized renditions; clients request only the fields they show where an endpoint offers it.
- Offline: mobile caches course structure and licensed downloads (encrypted, C6); progress and quiz answers made offline are queued and sent later with Idempotency-Key, so retries never double-count. When server and offline data conflict, the server applies a fixed rule per type (progress: furthest point wins; quiz attempts: only within the attempt's time window).
- Device integrity (optional signal): apps may send Play Integrity or App Attest tokens on login and refresh; the server uses them only as a rate-limit and risk signal, never to grant access. Certificate pinning is not used by default (it risks outages on certificate rotation); if added, pin to the CA's intermediate keys with a backup pin.
- Push notifications: device tokens are a tenant table (identity.device_push_tokens: organization_id, user_id, session_id, platform, token (encrypted), last_seen_at), removed on logout and on session revocation. Notification content never includes personal health data or links that act without a login.
- No secrets in app binaries: mobile and desktop apps contain only public values (API base URL, OAuth client ids, CAPTCHA site keys, push sender ids). Never API keys, signing secrets, provider keys or service credentials; anything secret stays on the server. Anything shipped in an app is assumed to be extracted by attackers. Release builds are checked for secrets in CI, debug logging is off, and builds are obfuscated as a hurdle, not a protection.
- Versioning: clients send X-Client-Version. The API publishes a minimum supported version per platform; below it, the API returns 426 client_upgrade_required so the app can force an update. API deprecations follow foundation §7.6.
- Localization and time: clients map error code values to their own localized messages (English and Hindi first, proposed); dates arrive in UTC and are shown in the organization's time zone or the device's, as the screen requires.
- Accessibility: web and apps target WCAG 2.1 AA; captions on all video (C6).

## C13. Accepted residual risks

No system is unbreakable. These risks remain after every control in the three documents, and are accepted on purpose. Each needs sign-off before the first hospital contract and a review every year.

| Risk | Why it remains | Mitigation in place |
|---|---|---|
| A stolen access token works until it expires or is revoked | Bearer tokens are not bound to a device (DPoP or mTLS would be, at the cost of client complexity) | 15-minute lifetime, immediate revocation, refresh replay detection, new-device notifications. Revisit device-bound tokens if the threat model requires it |
| Revocation can take up to 2 seconds in the worst case | Postgres and Redis are separate systems | The freshness lease bounds it; normally it is immediate |
| A Postgres failover makes authenticated requests fail for 30–120 seconds | The lease depends on Postgres by design | Public routes keep working; Multi-AZ; alerts |
| Media stays reachable up to 15 minutes after revocation | CloudFront cookies cannot be revoked individually | 15-minute cookies, watermarking |
| Plaintext personal columns (names, emails) remain in backups until backup retention ends after an organization is deleted | Encrypting every column would block search and indexing | Backup retention stated in the DPA; encrypted columns are crypto-shredded at once |
| Data keys exist in API memory for up to 5 minutes | Needed to encrypt and decrypt without calling KMS on every request | Memory is never written to disk or logs; short TTL |
| UUIDv7 ids reveal their creation time | Time-ordered ids are needed for index performance | Ids are not secrets; authorization never depends on them |
| An insider with both cloud admin and database admin rights could read data | Someone must be able to operate the system | No standing access, break-glass with approval and recording, CloudTrail, two-person rule for destructive platform actions |
| Zero-day vulnerabilities in dependencies or the runtime | Cannot be prevented, only responded to | Scanning, fast patch SLAs, WAF, least privilege, isolated file processing |
| A determined attacker can still phish an organization user | Users choose their own security | MFA (required for admins), breach-checked passwords, notifications, CAPTCHA and delays against guessing |
| AI models can be manipulated by content they read | No model is immune to prompt injection | The model is never trusted: all tools run through normal authorization, confirmation for actions, sanitized output, injection test corpus (plan AI invariants) |
| The frozen MongoDB variant has no tenant isolation | It is kept for possible future use instead of being deleted | Allowed only in local development; the app refuses to start with it anywhere else; never extended; warning logged while it runs |
| A bug in our own code | Every system has bugs | Three independent tenant-isolation layers, deny-by-default authorization, CI enforcement, external penetration tests |
