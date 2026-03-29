# 🏛 Village Community Digital Platform

## Architecture, Core Services, Contracts, and Executable Task Graph

> **Role alignment:** This document defines the system architecture and execution contracts. It intentionally avoids implementation code and focuses on deterministic, integration-safe task orchestration.

---

## 1) Architecture

### 1.1 Architectural Style

- **Modular Monolith (DDD boundaries) + Event-Driven Orchestration**
- **Contract-first delivery**: APIs, DTOs, schemas, and interfaces are defined before implementation.
- **Strict module isolation**: no cross-module DB access; integration via service contracts and events only.

### 1.2 Layered Structure

```text
Presentation (HTTP/API, Admin UI)
  -> Application (Use-cases, orchestration)
    -> Domain (Entities, value objects, invariants)
      -> Infrastructure (DBAL, queue, cache, logging, JWT, storage)
```

### 1.3 Repository / Project Structure

```text
/app
  /Core
    /Application
    /Domain
    /Infrastructure
    /Presentation
  /Modules
    /Auth
    /Users
    /Forum
    /Projects
    /News
    /DigitalID
    /Forms
    /Scholarships
    /Ads
    /Notifications
    /Admin
  /Contracts
    /Api
    /Dto
    /Events
    /Services
  /Bootstrap
/resources
/routes
/config
/storage
/public
```

### 1.4 Runtime Modes

- **Web Runtime**: serves HTTP requests, validates JWT/session, dispatches sync events.
- **Worker Runtime**: consumes queue jobs for async listeners/side effects.
- **Scheduler Runtime**: runs cron-backed jobs (digests, leaderboards snapshots, cleanup).

### 1.5 Dependency & Integration Rules

- Core does not depend on modules.
- Modules may depend on Core + Contracts.
- Modules never directly depend on other modules’ persistence layer.
- Module communication allowed via:
  - Public service interface in `/app/Contracts/Services`
  - Public API endpoint
  - Domain events on Event Bus

---

## 2) Core Services

### 2.1 Core Kernel / Bootstrap

Defines module registration, DI container wiring, env selection, middleware stack, and transport bindings (HTTP/CLI/worker).

### 2.2 Environment-Aware Config

- Sources: `.env`, environment-specific config files, defaults.
- Environments: `dev`, `staging`, `prod`.
- Domains:
  - DB connection profiles
  - Cache backend
  - Queue transport
  - JWT verifier settings (Google)
  - Logging channels and retention

### 2.3 Database Abstraction (DBAL)

- Query builder + transaction boundary service.
- Migration registry with deterministic ordering.
- Connection factory per environment profile.

### 2.4 Authentication (Google JWT)

- Verifies Google-issued JWTs (`iss`, `aud`, `exp`, `sub`, signature).
- Maps identity claims to internal user identity contract.
- Supports session bridging for admin console if required.

### 2.5 Event Bus

- Sync dispatch for domain events within request lifecycle.
- Async publication to queue for heavy side effects.
- Event metadata required on every event:
  - `event_id`, `event_name`, `occurred_at`, `actor_id`, `correlation_id`, `causation_id`.

### 2.6 Queue System

- Queue topics by module + priority.
- Idempotency key support.
- Dead-letter queue and retry policy contracts.

### 2.7 Cache Layer

- Read model caching: homepage, leaderboards, trending topics.
- Cache stampede controls (locking + TTL jitter).
- Invalidation by event handlers.

### 2.8 Structured Logging (Monolog)

Channels:

- `app` (app lifecycle)
- `audit` (business mutations)
- `security` (authn/authz events)
- `queue` (job lifecycle)
- `integration` (external API interactions)

Common log envelope:

- `timestamp`, `level`, `channel`, `message`, `context`, `correlation_id`, `actor_id`.

### 2.9 Dependency Injection Strategy

- Interfaces in `/app/Contracts/Services`.
- Adapters in module infrastructure.
- No runtime service location in domain layer.
- All module entry services explicitly registered.

---

## 3) Modules

For each module: responsibility, entities, DB schema, public API/services, emitted events, consumed events.

---

### 3.1 Auth Module

**Responsibility**
- Google JWT login, token/session lifecycle, auth state issuance.

**Domain Entities**
- `AuthSession`, `ExternalIdentity`.

**DB Schema (DBAL-ready)**
- `auth_sessions(id, user_id, provider, provider_subject, issued_at, expires_at, revoked_at, created_at)`
- `external_identities(id, user_id, provider, provider_subject, email, created_at, updated_at)`

**Public API / Services**
- `POST /api/v1/auth/google/verify`
- `POST /api/v1/auth/logout`
- `AuthIdentityResolverInterface`

**Events Emitted**
- `auth.user_authenticated`
- `auth.user_logged_out`

**Events Consumed**
- `users.user_deactivated` (invalidate sessions)

---

### 3.2 Users Module

**Responsibility**
- User profile lifecycle, roles, moderation flags, leaderboard points ledger ownership.

**Domain Entities**
- `User`, `UserProfile`, `RoleAssignment`, `PointsLedgerEntry`.

**DB Schema**
- `users(id, email, display_name, status, created_at, updated_at)`
- `user_profiles(user_id, bio, avatar_path, location, updated_at)`
- `user_roles(id, user_id, role, assigned_at)`
- `user_points_ledger(id, user_id, source, points_delta, reference_id, created_at)`

**Public API / Services**
- `GET /api/v1/users/{id}`
- `PATCH /api/v1/users/{id}`
- `GET /api/v1/leaderboard`
- `UserDirectoryQueryInterface`

**Events Emitted**
- `users.user_registered`
- `users.user_profile_updated`
- `users.user_deactivated`
- `users.points_awarded`

**Events Consumed**
- `forum.topic_created`
- `forum.reply_added`
- `projects.project_published`
- `news.comment_added`
- `scholarships.application_submitted`

---

### 3.3 Forum Module

**Responsibility**
- Discussion categories, topics, replies, moderation state.

**Domain Entities**
- `ForumCategory`, `ForumTopic`, `ForumReply`.

**DB Schema**
- `forum_categories(id, name, slug, created_at)`
- `forum_topics(id, category_id, author_id, title, body, status, created_at, updated_at)`
- `forum_replies(id, topic_id, author_id, body, created_at, updated_at)`

**Public API / Services**
- `POST /api/v1/forum/topics`
- `POST /api/v1/forum/topics/{id}/replies`
- `GET /api/v1/forum/topics/{id}`

**Events Emitted**
- `forum.topic_created`
- `forum.reply_added`
- `forum.topic_locked`

**Events Consumed**
- `users.user_deactivated` (freeze authoring)

---

### 3.4 Projects Module

**Responsibility**
- Community project showcase with status and highlights.

**Domain Entities**
- `Project`, `ProjectMember`, `ProjectMedia`.

**DB Schema**
- `projects(id, owner_id, title, summary, status, published_at, created_at, updated_at)`
- `project_members(id, project_id, user_id, role, joined_at)`
- `project_media(id, project_id, media_type, path, created_at)`

**Public API / Services**
- `POST /api/v1/projects`
- `POST /api/v1/projects/{id}/publish`
- `GET /api/v1/projects/{id}`

**Events Emitted**
- `projects.project_created`
- `projects.project_published`

**Events Consumed**
- `users.user_deactivated` (enforce ownership transfer policy)

---

### 3.5 News Module

**Responsibility**
- News posts, publishing workflow, comments.

**Domain Entities**
- `NewsPost`, `NewsComment`.

**DB Schema**
- `news_posts(id, author_id, title, slug, body, status, published_at, created_at, updated_at)`
- `news_comments(id, post_id, author_id, body, status, created_at)`

**Public API / Services**
- `POST /api/v1/news`
- `POST /api/v1/news/{id}/publish`
- `POST /api/v1/news/{id}/comments`

**Events Emitted**
- `news.published`
- `news.comment_added`

**Events Consumed**
- `users.user_deactivated` (disable new comments)

---

### 3.6 DigitalID Module

**Responsibility**
- Generate, store, and revoke digital identity artifacts.

**Domain Entities**
- `DigitalIdCard`, `DigitalIdIssueRecord`.

**DB Schema**
- `digital_ids(id, user_id, id_number, status, issued_at, revoked_at)`
- `digital_id_issues(id, digital_id_id, artifact_path, checksum, created_at)`

**Public API / Services**
- `POST /api/v1/digital-id/generate`
- `GET /api/v1/digital-id/{userId}`

**Events Emitted**
- `digital_id.generated`
- `digital_id.revoked`

**Events Consumed**
- `users.user_registered`
- `users.user_deactivated`

---

### 3.7 Forms Module

**Responsibility**
- Dynamic form definitions + submissions for reusable workflows.

**Domain Entities**
- `FormDefinition`, `FormField`, `FormSubmission`.

**DB Schema**
- `forms(id, key, title, status, version, created_at, updated_at)`
- `form_fields(id, form_id, name, type, config_json, position)`
- `form_submissions(id, form_id, submitter_id, payload_json, submitted_at)`

**Public API / Services**
- `POST /api/v1/forms`
- `POST /api/v1/forms/{id}/submit`
- `GET /api/v1/forms/{id}`

**Events Emitted**
- `forms.form_published`
- `forms.submission_received`

**Events Consumed**
- none mandatory

---

### 3.8 Scholarships Module

**Responsibility**
- Scholarship campaigns and application processing.

**Domain Entities**
- `Scholarship`, `ScholarshipApplication`, `AwardDecision`.

**DB Schema**
- `scholarships(id, title, description, status, opens_at, closes_at, created_at)`
- `scholarship_applications(id, scholarship_id, applicant_id, form_submission_id, status, submitted_at)`
- `scholarship_awards(id, application_id, awarded_by, awarded_at, notes)`

**Public API / Services**
- `POST /api/v1/scholarships`
- `POST /api/v1/scholarships/{id}/apply`
- `POST /api/v1/scholarships/applications/{id}/award`

**Events Emitted**
- `scholarships.application_submitted`
- `scholarships.application_awarded`

**Events Consumed**
- `forms.submission_received`
- `users.user_deactivated` (block pending processing)

---

### 3.9 Ads Module

**Responsibility**
- Ad inventory, targeting metadata, impressions/clicks tracking.

**Domain Entities**
- `AdCampaign`, `AdCreative`, `AdImpression`, `AdClick`.

**DB Schema**
- `ad_campaigns(id, name, status, starts_at, ends_at, budget_cents, created_at)`
- `ad_creatives(id, campaign_id, placement, asset_path, target_json, created_at)`
- `ad_impressions(id, creative_id, viewer_id, context_json, occurred_at)`
- `ad_clicks(id, creative_id, viewer_id, occurred_at)`

**Public API / Services**
- `POST /api/v1/ads/campaigns`
- `GET /api/v1/ads/placement/{slot}`
- `POST /api/v1/ads/click`

**Events Emitted**
- `ads.impression_recorded`
- `ads.click_recorded`

**Events Consumed**
- `users.user_deactivated` (optional anonymization policy)

---

### 3.10 Notifications Module

**Responsibility**
- Email/in-app notification composition and dispatch.

**Domain Entities**
- `NotificationMessage`, `NotificationPreference`, `DeliveryRecord`.

**DB Schema**
- `notification_messages(id, recipient_id, channel, template_key, payload_json, status, created_at)`
- `notification_preferences(id, user_id, channel, enabled, updated_at)`
- `notification_deliveries(id, message_id, provider_ref, delivered_at, failure_reason)`

**Public API / Services**
- `POST /api/v1/notifications/send` (admin/system only)
- `GET /api/v1/notifications/preferences`
- `PATCH /api/v1/notifications/preferences`

**Events Emitted**
- `notifications.message_queued`
- `notifications.message_delivered`
- `notifications.delivery_failed`

**Events Consumed**
- `users.user_registered`
- `news.published`
- `scholarships.application_awarded`
- `digital_id.generated`

---

### 3.11 Admin Module

**Responsibility**
- Moderation dashboards, workflow approvals, audit views, system controls.

**Domain Entities**
- `ModerationCase`, `AdminActionLog`.

**DB Schema**
- `moderation_cases(id, module, reference_id, status, opened_by, opened_at, resolved_at)`
- `admin_action_logs(id, admin_id, action, target_type, target_id, context_json, created_at)`

**Public API / Services**
- `GET /api/v1/admin/dashboard`
- `POST /api/v1/admin/moderation/cases/{id}/resolve`
- `GET /api/v1/admin/audit`

**Events Emitted**
- `admin.moderation_case_opened`
- `admin.moderation_case_resolved`

**Events Consumed**
- `forum.topic_created`
- `news.comment_added`
- `ads.click_recorded` (fraud heuristics)

---

## 4) Event Catalog (Contract-First)

All events share envelope:

```json
{
  "event_id": "uuid",
  "event_name": "string",
  "occurred_at": "datetime",
  "actor_id": "uuid|null",
  "correlation_id": "uuid",
  "causation_id": "uuid|null",
  "payload": {}
}
```

| Event Name | Payload Schema (core fields) | Trigger Source | Consumers | Side Effects |
|---|---|---|---|---|
| `auth.user_authenticated` | `user_id, provider, session_id` | Auth login success | Notifications, Audit | Welcome/login alerts, audit log |
| `users.user_registered` | `user_id, email, registered_at` | User creation | DigitalID, Notifications | ID generation workflow, welcome notification |
| `users.user_profile_updated` | `user_id, changed_fields[]` | Profile update | Admin, Cache | audit, cache invalidation |
| `forum.topic_created` | `topic_id, author_id, category_id` | Topic creation | Users, Admin, Notifications | points award, moderation check, subscriber alerts |
| `forum.reply_added` | `reply_id, topic_id, author_id` | Reply creation | Users, Notifications | points award, thread alerts |
| `projects.project_published` | `project_id, owner_id, published_at` | Publish project | Users, Notifications | points award, feed refresh |
| `news.published` | `post_id, author_id, published_at` | Publish news | Notifications, Cache | member notifications, homepage refresh |
| `news.comment_added` | `comment_id, post_id, author_id` | Comment added | Users, Admin | points, moderation rules |
| `digital_id.generated` | `digital_id_id, user_id, artifact_path` | Digital ID issued | Notifications, Audit | delivery notice, compliance log |
| `forms.submission_received` | `form_id, submission_id, submitter_id` | Form submit | Scholarships, Admin | route application workflows |
| `scholarships.application_submitted` | `application_id, scholarship_id, applicant_id` | Scholarship apply | Users, Notifications, Admin | points, receipt notification |
| `scholarships.application_awarded` | `application_id, applicant_id, awarded_at` | Award decision | Notifications, News | winner notice, announcement draft |
| `ads.impression_recorded` | `impression_id, creative_id, viewer_id?` | Ad served | Admin analytics | performance metrics update |
| `ads.click_recorded` | `click_id, creative_id, viewer_id?` | Ad click | Admin fraud checks | CTR update, anomaly checks |
| `notifications.message_delivered` | `message_id, recipient_id, channel` | Provider ack | Admin/audit | delivery KPI updates |

---

## 5) Contracts

### 5.1 API Schema Conventions

- Base path: `/api/v1`.
- JSON request/response with explicit versioning.
- Error envelope:

```json
{
  "error": {
    "code": "string",
    "message": "string",
    "details": {}
  },
  "correlation_id": "uuid"
}
```

### 5.2 DTO Conventions

- Immutable DTOs in `/app/Contracts/Dto/{Module}`.
- Input DTOs validated at boundary layer.
- Output DTOs represent read models only (no domain entity leakage).

### 5.3 DB Schema Conventions

- UUID primary keys.
- `created_at`, `updated_at` for mutable records.
- Soft-delete fields only where retention policy requires.
- No foreign key to another module table unless shared contract approved in Core migration policy.

### 5.4 Service Interface Contracts (examples)

- `AuthIdentityResolverInterface`
- `UserDirectoryQueryInterface`
- `EventPublisherInterface`
- `QueueDispatcherInterface`
- `CacheStoreInterface`
- `DigitalIdIssuerInterface`
- `NotificationDispatcherInterface`

### 5.5 Security & Authorization Contracts

- JWT validation contract (Google keys, issuer, audience).
- Policy interface per module action: `can{Action}(Actor, Resource)`.
- Audit contract for all mutating commands.

---

## 6) Task Graph (Codex-Executable)

> Every task is self-contained, deterministic, integration-safe, and can be executed independently once dependencies are satisfied.

### TASK_ID: CORE-001
**TITLE:** Initialize kernel, modular loader, and runtime entry points  
**MODULE:** Core  
**TYPE:** core

**DESCRIPTION:** Define application kernel, module registry, boot sequence, and web/worker/scheduler entry contracts.

**FILES_TO_CREATE:**
- `app/Core/Application/Kernel.php`
- `app/Bootstrap/WebRuntime.php`
- `app/Bootstrap/WorkerRuntime.php`
- `app/Bootstrap/SchedulerRuntime.php`
- `app/Contracts/Services/ModuleInterface.php`

**FILES_TO_MODIFY:**
- `public/index.php`

**INPUT_CONTRACTS:**
- Runtime mode definitions from architecture section.

**OUTPUT_CONTRACTS:**
- Boot contract for module registration and runtime startup.

**DEPENDENCIES:**
- none

**IMPLEMENTATION_GUIDELINES:**
- No module-specific logic in Kernel.
- Runtime startup must emit correlation ID.

**ACCEPTANCE_CRITERIA:**
- Web, worker, scheduler can boot without module implementation present.

---

### TASK_ID: CORE-002
**TITLE:** Implement environment-aware config subsystem  
**MODULE:** Core  
**TYPE:** core

**DESCRIPTION:** Define config loader contracts with env override precedence and typed accessors.

**FILES_TO_CREATE:**
- `app/Core/Infrastructure/Config/ConfigLoader.php`
- `app/Contracts/Services/ConfigInterface.php`
- `config/dev.php`
- `config/staging.php`
- `config/prod.php`

**FILES_TO_MODIFY:**
- `app/Bootstrap/WebRuntime.php`

**INPUT_CONTRACTS:**
- Env matrix and config domains.

**OUTPUT_CONTRACTS:**
- Deterministic config resolution contract.

**DEPENDENCIES:**
- `CORE-001`

**IMPLEMENTATION_GUIDELINES:**
- Explicit precedence: defaults < env file < process env.

**ACCEPTANCE_CRITERIA:**
- Same key resolves predictably across all environments.

---

### TASK_ID: CORE-003
**TITLE:** Define DI container strategy and service registration contracts  
**MODULE:** Core  
**TYPE:** core

**DESCRIPTION:** Provide interface-based dependency registration and module-scoped bindings.

**FILES_TO_CREATE:**
- `app/Core/Infrastructure/DI/Container.php`
- `app/Contracts/Services/ServiceProviderInterface.php`

**FILES_TO_MODIFY:**
- `app/Core/Application/Kernel.php`

**INPUT_CONTRACTS:**
- Service interface conventions.

**OUTPUT_CONTRACTS:**
- Container registration and resolution contract.

**DEPENDENCIES:**
- `CORE-001`, `CORE-002`

**IMPLEMENTATION_GUIDELINES:**
- Domain layer cannot pull from container directly.

**ACCEPTANCE_CRITERIA:**
- Interface bindings resolvable in runtime boot.

---

### TASK_ID: INFRA-DB-001
**TITLE:** Establish DBAL connection factory and transaction boundary service  
**MODULE:** Core  
**TYPE:** infra

**DESCRIPTION:** Define DBAL adapters, connection lifecycle, transaction helper contracts, and migration registry stub.

**FILES_TO_CREATE:**
- `app/Core/Infrastructure/Database/ConnectionFactory.php`
- `app/Core/Infrastructure/Database/TransactionManager.php`
- `app/Contracts/Services/DatabaseInterface.php`

**FILES_TO_MODIFY:**
- `app/Core/Infrastructure/DI/Container.php`

**INPUT_CONTRACTS:**
- Config contracts for DB.

**OUTPUT_CONTRACTS:**
- DB access + transaction contracts for modules.

**DEPENDENCIES:**
- `CORE-002`, `CORE-003`

**IMPLEMENTATION_GUIDELINES:**
- No module repositories in this task.

**ACCEPTANCE_CRITERIA:**
- Transaction boundary callable from a dummy command service.

---

### TASK_ID: INFRA-LOG-001
**TITLE:** Configure structured logging channels with Monolog  
**MODULE:** Core  
**TYPE:** infra

**DESCRIPTION:** Set up channel routing, shared context processor, and correlation-aware log formatter.

**FILES_TO_CREATE:**
- `app/Core/Infrastructure/Logging/LoggerFactory.php`
- `app/Contracts/Services/LoggerInterface.php`

**FILES_TO_MODIFY:**
- `app/Bootstrap/WebRuntime.php`
- `app/Bootstrap/WorkerRuntime.php`

**INPUT_CONTRACTS:**
- Logging envelope contract.

**OUTPUT_CONTRACTS:**
- Standardized logging service across runtimes.

**DEPENDENCIES:**
- `CORE-002`, `CORE-003`

**IMPLEMENTATION_GUIDELINES:**
- Required channels: app/audit/security/queue/integration.

**ACCEPTANCE_CRITERIA:**
- Sample log lines include correlation_id and actor_id where available.

---

### TASK_ID: INFRA-EVENT-001
**TITLE:** Define event envelope, sync dispatcher, and publisher interfaces  
**MODULE:** Core  
**TYPE:** infra

**DESCRIPTION:** Implement base event contract and synchronous in-process event dispatching.

**FILES_TO_CREATE:**
- `app/Contracts/Events/EventEnvelope.php`
- `app/Core/Infrastructure/Event/EventDispatcher.php`
- `app/Contracts/Services/EventPublisherInterface.php`

**FILES_TO_MODIFY:**
- `app/Core/Infrastructure/DI/Container.php`

**INPUT_CONTRACTS:**
- Event envelope schema.

**OUTPUT_CONTRACTS:**
- Sync event dispatch contract.

**DEPENDENCIES:**
- `CORE-003`

**IMPLEMENTATION_GUIDELINES:**
- Enforce required metadata fields.

**ACCEPTANCE_CRITERIA:**
- Test event listener receives exact envelope fields.

---

### TASK_ID: INFRA-QUEUE-001
**TITLE:** Define queue dispatcher, retry policy, and DLQ contracts  
**MODULE:** Core  
**TYPE:** infra

**DESCRIPTION:** Provide queue abstraction for async listeners with retry semantics and dead-letter routing.

**FILES_TO_CREATE:**
- `app/Core/Infrastructure/Queue/QueueDispatcher.php`
- `app/Core/Infrastructure/Queue/RetryPolicy.php`
- `app/Contracts/Services/QueueDispatcherInterface.php`

**FILES_TO_MODIFY:**
- `app/Core/Infrastructure/DI/Container.php`

**INPUT_CONTRACTS:**
- Queue + idempotency requirements.

**OUTPUT_CONTRACTS:**
- Async job dispatch contract.

**DEPENDENCIES:**
- `CORE-003`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Every job payload includes idempotency key.

**ACCEPTANCE_CRITERIA:**
- Failed job transitions to retry then DLQ as configured.

---

### TASK_ID: INFRA-CACHE-001
**TITLE:** Define cache store and lock-assisted invalidation contracts  
**MODULE:** Core  
**TYPE:** infra

**DESCRIPTION:** Add cache abstraction with TTL, tags (if supported), and stampede-safe lock usage.

**FILES_TO_CREATE:**
- `app/Core/Infrastructure/Cache/CacheStore.php`
- `app/Contracts/Services/CacheStoreInterface.php`

**FILES_TO_MODIFY:**
- `app/Core/Infrastructure/DI/Container.php`

**INPUT_CONTRACTS:**
- Cache requirements and invalidation model.

**OUTPUT_CONTRACTS:**
- Cache contract for read-model modules.

**DEPENDENCIES:**
- `CORE-003`

**IMPLEMENTATION_GUIDELINES:**
- Provide `remember(key, ttl, closure)` semantics.

**ACCEPTANCE_CRITERIA:**
- Cache hit/miss behavior deterministic in integration test.

---

### TASK_ID: AUTH-001
**TITLE:** Define Auth module contracts for Google JWT verification  
**MODULE:** Auth  
**TYPE:** module

**DESCRIPTION:** Create API schemas, DTOs, and service interfaces for Google JWT verification and session issuance.

**FILES_TO_CREATE:**
- `app/Modules/Auth/Application/Contracts/VerifyGoogleTokenCommand.php`
- `app/Contracts/Api/Auth/verify_google_token.json`
- `app/Contracts/Services/AuthIdentityResolverInterface.php`
- `app/Contracts/Events/auth.user_authenticated.json`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- Security and API conventions.

**OUTPUT_CONTRACTS:**
- Auth API and event contracts.

**DEPENDENCIES:**
- `CORE-003`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Validate `iss/aud/exp/sub` contract before auth acceptance.

**ACCEPTANCE_CRITERIA:**
- Contract tests reject malformed JWT claims.

---

### TASK_ID: USERS-001
**TITLE:** Define Users module contracts, schema, and leaderboard interfaces  
**MODULE:** Users  
**TYPE:** module

**DESCRIPTION:** Establish user/profile/roles/points ledger schemas and public API DTO contracts.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Users/get_user.json`
- `app/Contracts/Api/Users/update_user.json`
- `app/Contracts/Api/Users/get_leaderboard.json`
- `app/Contracts/Events/users.user_registered.json`
- `app/Contracts/Events/users.points_awarded.json`
- `database/migrations/users_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- Base API, DTO, DB conventions.

**OUTPUT_CONTRACTS:**
- Users module public contract surface.

**DEPENDENCIES:**
- `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Leaderboard is read model from points ledger only.

**ACCEPTANCE_CRITERIA:**
- Schema validates with DBAL migration linter.

---

### TASK_ID: FORUM-001
**TITLE:** Define Forum module contracts and schemas  
**MODULE:** Forum  
**TYPE:** module

**DESCRIPTION:** Create topic/reply/category contracts and emitted event schemas.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Forum/create_topic.json`
- `app/Contracts/Api/Forum/add_reply.json`
- `app/Contracts/Events/forum.topic_created.json`
- `app/Contracts/Events/forum.reply_added.json`
- `database/migrations/forum_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- Users identity contract.

**OUTPUT_CONTRACTS:**
- Forum API/event/schema contracts.

**DEPENDENCIES:**
- `USERS-001`, `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- No direct write to users tables.

**ACCEPTANCE_CRITERIA:**
- Event contract tests pass for topic/reply payload shapes.

---

### TASK_ID: PROJECTS-001
**TITLE:** Define Projects module contracts and schemas  
**MODULE:** Projects  
**TYPE:** module

**DESCRIPTION:** Establish project lifecycle contracts with publish event.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Projects/create_project.json`
- `app/Contracts/Api/Projects/publish_project.json`
- `app/Contracts/Events/projects.project_published.json`
- `database/migrations/projects_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- User ownership contract.

**OUTPUT_CONTRACTS:**
- Projects API/event/schema contracts.

**DEPENDENCIES:**
- `USERS-001`, `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Publish transition must be explicit state change.

**ACCEPTANCE_CRITERIA:**
- Contract tests enforce state-machine transitions.

---

### TASK_ID: NEWS-001
**TITLE:** Define News module contracts, comments, and publish workflows  
**MODULE:** News  
**TYPE:** module

**DESCRIPTION:** Define news + comments APIs and event payload contracts.

**FILES_TO_CREATE:**
- `app/Contracts/Api/News/create_news.json`
- `app/Contracts/Api/News/publish_news.json`
- `app/Contracts/Api/News/add_comment.json`
- `app/Contracts/Events/news.published.json`
- `app/Contracts/Events/news.comment_added.json`
- `database/migrations/news_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- User identity contract.

**OUTPUT_CONTRACTS:**
- News module contract surface.

**DEPENDENCIES:**
- `USERS-001`, `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Draft -> Published workflow required.

**ACCEPTANCE_CRITERIA:**
- Publish/comment payload contracts validate.

---

### TASK_ID: DIGITALID-001
**TITLE:** Define DigitalID module contracts and generation events  
**MODULE:** DigitalID  
**TYPE:** module

**DESCRIPTION:** Create generation/revocation contracts and storage metadata schema.

**FILES_TO_CREATE:**
- `app/Contracts/Api/DigitalID/generate_digital_id.json`
- `app/Contracts/Events/digital_id.generated.json`
- `database/migrations/digital_id_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- Users registration event contract.

**OUTPUT_CONTRACTS:**
- DigitalID API/event/schema contracts.

**DEPENDENCIES:**
- `USERS-001`, `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Artifact checksum required in schema.

**ACCEPTANCE_CRITERIA:**
- Generated event must include artifact path + checksum fields.

---

### TASK_ID: FORMS-001
**TITLE:** Define dynamic form builder contracts and schemas  
**MODULE:** Forms  
**TYPE:** module

**DESCRIPTION:** Define form definition/submission APIs with versioned form contracts.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Forms/create_form.json`
- `app/Contracts/Api/Forms/submit_form.json`
- `app/Contracts/Events/forms.submission_received.json`
- `database/migrations/forms_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- DTO and validation conventions.

**OUTPUT_CONTRACTS:**
- Forms API/event/schema contracts.

**DEPENDENCIES:**
- `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Submission payload stored as schema-validated JSON.

**ACCEPTANCE_CRITERIA:**
- Contract tests verify field types and required constraints.

---

### TASK_ID: SCHOLARSHIPS-001
**TITLE:** Define Scholarships contracts, forms integration, and award events  
**MODULE:** Scholarships  
**TYPE:** module

**DESCRIPTION:** Build application + award contracts with explicit dependency on form submissions.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Scholarships/apply.json`
- `app/Contracts/Api/Scholarships/award.json`
- `app/Contracts/Events/scholarships.application_submitted.json`
- `app/Contracts/Events/scholarships.application_awarded.json`
- `database/migrations/scholarships_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- Forms submission event contract.

**OUTPUT_CONTRACTS:**
- Scholarships API/event/schema contracts.

**DEPENDENCIES:**
- `FORMS-001`, `USERS-001`, `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Application references form_submission_id only.

**ACCEPTANCE_CRITERIA:**
- Award command requires existing submitted application state.

---

### TASK_ID: ADS-001
**TITLE:** Define Ads contracts, targeting metadata, and metrics events  
**MODULE:** Ads  
**TYPE:** module

**DESCRIPTION:** Define campaign/creative/impression/click contracts for ad delivery and analytics.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Ads/create_campaign.json`
- `app/Contracts/Api/Ads/get_placement.json`
- `app/Contracts/Events/ads.impression_recorded.json`
- `app/Contracts/Events/ads.click_recorded.json`
- `database/migrations/ads_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- User identity optionality contract for anonymous viewers.

**OUTPUT_CONTRACTS:**
- Ads API/event/schema contracts.

**DEPENDENCIES:**
- `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- Anonymous metrics supported with nullable viewer_id.

**ACCEPTANCE_CRITERIA:**
- Impression/click payload schemas validate anonymous + authenticated flows.

---

### TASK_ID: NOTIFICATIONS-001
**TITLE:** Define Notifications contracts and queue integration surfaces  
**MODULE:** Notifications  
**TYPE:** module

**DESCRIPTION:** Define notification message, preference, and delivery event contracts with queue dispatch hooks.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Notifications/get_preferences.json`
- `app/Contracts/Api/Notifications/update_preferences.json`
- `app/Contracts/Events/notifications.message_queued.json`
- `app/Contracts/Events/notifications.message_delivered.json`
- `database/migrations/notifications_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- Queue and event contracts.

**OUTPUT_CONTRACTS:**
- Notifications API/event/schema contracts.

**DEPENDENCIES:**
- `INFRA-QUEUE-001`, `INFRA-EVENT-001`, `USERS-001`

**IMPLEMENTATION_GUIDELINES:**
- Delivery provider reference field mandatory.

**ACCEPTANCE_CRITERIA:**
- Queue message contract includes idempotency key.

---

### TASK_ID: ADMIN-001
**TITLE:** Define Admin moderation and audit contracts  
**MODULE:** Admin  
**TYPE:** module

**DESCRIPTION:** Define moderation case and audit query contracts, including resolution workflow events.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Admin/get_dashboard.json`
- `app/Contracts/Api/Admin/resolve_moderation_case.json`
- `app/Contracts/Events/admin.moderation_case_resolved.json`
- `database/migrations/admin_001.sql`

**FILES_TO_MODIFY:**
- `routes/api.php`

**INPUT_CONTRACTS:**
- Role and policy contracts from Users/Auth.

**OUTPUT_CONTRACTS:**
- Admin API/event/schema contracts.

**DEPENDENCIES:**
- `USERS-001`, `AUTH-001`, `INFRA-DB-001`, `INFRA-EVENT-001`

**IMPLEMENTATION_GUIDELINES:**
- All admin mutations must produce audit event.

**ACCEPTANCE_CRITERIA:**
- Unauthorized actors denied by policy contract tests.

---

### TASK_ID: INTEGRATION-001
**TITLE:** Implement cross-module event routing matrix and consumer bindings  
**MODULE:** Integration  
**TYPE:** integration

**DESCRIPTION:** Register event subscriptions and ensure each consumer binds to contract-defined payload only.

**FILES_TO_CREATE:**
- `app/Bootstrap/EventSubscriptions.php`
- `app/Contracts/Events/subscription_matrix.md`

**FILES_TO_MODIFY:**
- `app/Core/Application/Kernel.php`

**INPUT_CONTRACTS:**
- All module event schemas.

**OUTPUT_CONTRACTS:**
- Deterministic event routing map.

**DEPENDENCIES:**
- `AUTH-001`, `USERS-001`, `FORUM-001`, `PROJECTS-001`, `NEWS-001`, `DIGITALID-001`, `FORMS-001`, `SCHOLARSHIPS-001`, `ADS-001`, `NOTIFICATIONS-001`, `ADMIN-001`

**IMPLEMENTATION_GUIDELINES:**
- No consumer may rely on undeclared payload fields.

**ACCEPTANCE_CRITERIA:**
- Contract test validates all registered consumers against schema.

---

### TASK_ID: INTEGRATION-002
**TITLE:** Define leaderboard aggregation read model contract  
**MODULE:** Integration  
**TYPE:** integration

**DESCRIPTION:** Build read-model contract that aggregates Users points ledger from emitted domain events.

**FILES_TO_CREATE:**
- `app/Contracts/Api/Users/get_leaderboard.json`
- `app/Contracts/Services/LeaderboardQueryInterface.php`
- `app/Contracts/Events/leaderboard.rebuilt.json`

**FILES_TO_MODIFY:**
- `app/Contracts/Events/subscription_matrix.md`

**INPUT_CONTRACTS:**
- `users.points_awarded` and contributor events.

**OUTPUT_CONTRACTS:**
- Leaderboard query + rebuild event contracts.

**DEPENDENCIES:**
- `USERS-001`, `INTEGRATION-001`, `INFRA-CACHE-001`

**IMPLEMENTATION_GUIDELINES:**
- Read model cache invalidated only by points-affecting events.

**ACCEPTANCE_CRITERIA:**
- Deterministic ranking for equal-score tie-break policy.

---

### TASK_ID: QA-CONTRACT-001
**TITLE:** Add platform-wide contract test harness  
**MODULE:** Core  
**TYPE:** integration

**DESCRIPTION:** Create schema validation harness for API payloads, event payloads, and migration naming consistency.

**FILES_TO_CREATE:**
- `tests/Contract/ApiContractTest.php`
- `tests/Contract/EventContractTest.php`
- `tests/Contract/MigrationContractTest.php`

**FILES_TO_MODIFY:**
- `composer.json`

**INPUT_CONTRACTS:**
- All API/event/schema contracts.

**OUTPUT_CONTRACTS:**
- Executable contract verification baseline.

**DEPENDENCIES:**
- `INTEGRATION-001`, `INTEGRATION-002`

**IMPLEMENTATION_GUIDELINES:**
- Tests must fail on undeclared fields and missing required fields.

**ACCEPTANCE_CRITERIA:**
- One command runs all contract checks deterministically.

---

## 7) Build Order

### 7.1 Dependency Graph (high-level)

1. **Foundation:** `CORE-001 -> CORE-002 -> CORE-003`
2. **Infrastructure Backbone:** parallel after core
   - `INFRA-DB-001`
   - `INFRA-LOG-001`
   - `INFRA-EVENT-001`
   - `INFRA-CACHE-001`
   - `INFRA-QUEUE-001` (depends on event)
3. **Module Contract Definitions:** parallel where possible
   - `AUTH-001`, `USERS-001`, `FORUM-001`, `PROJECTS-001`, `NEWS-001`, `DIGITALID-001`, `FORMS-001`, `SCHOLARSHIPS-001`, `ADS-001`, `NOTIFICATIONS-001`, `ADMIN-001`
4. **Cross-Module Integration:** `INTEGRATION-001 -> INTEGRATION-002`
5. **Verification:** `QA-CONTRACT-001`

### 7.2 Critical Path

`CORE-001 -> CORE-002 -> CORE-003 -> INFRA-EVENT-001 -> INFRA-QUEUE-001 -> NOTIFICATIONS-001 -> INTEGRATION-001 -> INTEGRATION-002 -> QA-CONTRACT-001`

### 7.3 Parallelizable Task Sets

- **Set A (after CORE-003):** `INFRA-DB-001`, `INFRA-LOG-001`, `INFRA-EVENT-001`, `INFRA-CACHE-001`
- **Set B (after infra prerequisites):** `AUTH-001`, `USERS-001`, `ADS-001`, `FORMS-001`
- **Set C (after USERS-001):** `FORUM-001`, `PROJECTS-001`, `NEWS-001`, `DIGITALID-001`, `ADMIN-001`
- **Set D (after FORMS-001 + USERS-001):** `SCHOLARSHIPS-001`

---

## 8) Success Conditions

A developer can take any single task, execute it independently, and integrate with zero rework because:

- Contracts are defined first.
- Dependencies are explicit.
- Module boundaries are enforced.
- Integration points are event/API based and schema-validated.
- Platform-wide contract tests provide deterministic verification.

