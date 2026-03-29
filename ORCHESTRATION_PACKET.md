# Executive Summary

This execution packet converts `VILLAGE_COMMUNITY_DIGITAL_PLATFORM.md` into a deterministic, contract-first orchestration plan for Codex desktop subagent execution. It preserves the blueprint architecture (modular monolith + event orchestration), enforces module isolation, and expands the original task graph into a strict DAG with phase gates, parallel groups, synchronization barriers, ownership, and task-level prompts.

Key outcomes:
- Strict dependency-safe task DAG with explicit blockers and unblocking criteria.
- Minimal correction layer only where the blueprint has ambiguity/overlap.
- Subagent ownership map across `CoreAgent`, `InfraAgent`, `ModuleAgent`, `IntegrationAgent`, `ReviewAgent`, `QAAgent`.
- Desktop-ready Build/Review/QA prompt triplets per task.
- Integration checkpoints and release gate criteria.

---

# Blueprint Validation

## Parsed and Normalized Blueprint Elements

- **Architecture layers:** Presentation → Application → Domain → Infrastructure.
- **Core services:** Kernel/bootstrap, config, DBAL, JWT auth, event bus, queue, cache, logging, DI.
- **Modules:** Auth, Users, Forum, Projects, News, DigitalID, Forms, Scholarships, Ads, Notifications, Admin.
- **Events:** Shared envelope + module event catalog + cross-module consumers.
- **Contracts:** API envelope, DTO immutability, DB conventions, service interfaces, auth/policy/audit requirements.
- **Existing task graph:** 20 tasks from `CORE-001` through `QA-CONTRACT-001` with high-level build order and critical path.

## Validation Findings

1. **Duplicate ownership overlap**
   - `INTEGRATION-002` recreates `app/Contracts/Api/Users/get_leaderboard.json`, already defined in `USERS-001`.
   - Risk: nondeterministic contract drift and race condition.

2. **Contract coverage gap for consumed events**
   - Multiple modules consume `users.user_deactivated`, but no explicit event schema task creates `users.user_deactivated.json`.
   - Risk: integration binding cannot be fully schema-verified.

3. **Task granularity mismatch in integration layer**
   - `INTEGRATION-001` includes both routing matrix definition and consumer binding assurance across all modules.
   - Large but still acceptable if constrained by explicit acceptance checks.

4. **Type labeling inconsistency**
   - `QA-CONTRACT-001` is semantically QA but typed as integration in the source graph.
   - Risk: orchestration ownership ambiguity.

5. **No circular dependency detected**
   - DAG is acyclic after resolving overlap and missing contract artifact.

---

# Minimal Correction Layer

> Scope-limited corrections only; no architectural redesign.

1. **MCL-01 (Ownership correction):**
   - Keep `get_leaderboard.json` contract creation in `USERS-001` only.
   - Change `INTEGRATION-002` to *consume and reference* leaderboard API contract rather than recreate.

2. **MCL-02 (Missing schema completion):**
   - Add `users.user_deactivated.json` to `USERS-001` output contracts.
   - Mark as required input contract for all module tasks consuming this event.

3. **MCL-03 (Task type normalization):**
   - Reclassify `QA-CONTRACT-001` task TYPE to `qa` for orchestration ownership.

4. **MCL-04 (Integration task constraint):**
   - Keep `INTEGRATION-001` single task, but enforce deterministic checklist:
     - Subscription matrix completeness
     - Schema-only payload binding
     - Consumer declaration parity test

---

# Normalized Architecture Map

## Layer Map

1. **Presentation Layer**
   - HTTP API routes and admin endpoints (`/api/v1/*`).
2. **Application Layer**
   - Use-case orchestration, command/query handlers, runtime coordination.
3. **Domain Layer**
   - Entities, invariants, state transitions.
4. **Infrastructure Layer**
   - DBAL, queue, cache, logging, JWT verification, worker/scheduler transports.

## Runtime Map

- **Web Runtime:** request validation/auth/context/correlation + sync event dispatch.
- **Worker Runtime:** queue consume/retry/DLQ handling + async effects.
- **Scheduler Runtime:** periodic jobs (digests, snapshots, cleanup).

## Isolation Rules

- No module-to-module database access.
- Cross-module interaction only via:
  - `/app/Contracts/Services/*` interfaces
  - public API contracts
  - event contracts + subscriptions.

---

# Normalized Contract Map

## Global Contracts

- **Event envelope contract:** `event_id`, `event_name`, `occurred_at`, `actor_id`, `correlation_id`, `causation_id`, `payload`.
- **API error envelope contract:** `error{code,message,details}`, `correlation_id`.
- **DTO contract:** immutable boundary objects; no domain leakage.
- **DB contract:** UUID PK, timestamp conventions, controlled soft-delete use.
- **Security contract:** Google JWT (`iss/aud/exp/sub`), per-action policy checks, mutation audit.

## Service Contract Families

- Runtime/boot: `ModuleInterface`, `ServiceProviderInterface`, `ConfigInterface`.
- Infra: `DatabaseInterface`, `EventPublisherInterface`, `QueueDispatcherInterface`, `CacheStoreInterface`, `LoggerInterface`.
- Module APIs: per-module JSON contracts under `/app/Contracts/Api/*`.
- Module events: per-event JSON schemas under `/app/Contracts/Events/*`.

## Event Dependency Highlights

- `USERS` is a central producer for identity lifecycle (`user_registered`, `user_deactivated`, points).
- `NOTIFICATIONS` is central consumer for user/news/scholarship/digital-id events.
- `INTEGRATION-001` is the subscription-binding consolidation barrier.

---

# Task DAG

## DAG Nodes (Normalized)

TASK_ID: CORE-001
TITLE: Initialize kernel, modular loader, and runtime entry points
DOMAIN: Core
TYPE: core
OWNER_AGENT: CoreAgent
EXECUTION_MODE: sequential
PARALLEL_GROUP: none
DEPENDS_ON: []
BLOCKS: [CORE-002, CORE-003]
INPUT_CONTRACTS: [Runtime mode definitions]
OUTPUT_CONTRACTS: [Boot contract, module registration contract]
FILES_EXPECTED_TO_CHANGE: [app/Core/Application/Kernel.php, app/Bootstrap/WebRuntime.php, app/Bootstrap/WorkerRuntime.php, app/Bootstrap/SchedulerRuntime.php, app/Contracts/Services/ModuleInterface.php, public/index.php]
RATIONALE: Establishes deterministic startup for all runtimes.
IMPLEMENTATION_BOUNDARY: Core bootstrapping only; no module logic.
ACCEPTANCE_CRITERIA: All runtimes boot with no module implementation.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: CORE-002
TITLE: Implement environment-aware config subsystem
DOMAIN: Core
TYPE: core
OWNER_AGENT: CoreAgent
EXECUTION_MODE: sequential
PARALLEL_GROUP: none
DEPENDS_ON: [CORE-001]
BLOCKS: [CORE-003, INFRA-DB-001, INFRA-LOG-001]
INPUT_CONTRACTS: [Config precedence contract]
OUTPUT_CONTRACTS: [Deterministic environment resolution]
FILES_EXPECTED_TO_CHANGE: [app/Core/Infrastructure/Config/ConfigLoader.php, app/Contracts/Services/ConfigInterface.php, config/dev.php, config/staging.php, config/prod.php, app/Bootstrap/WebRuntime.php]
RATIONALE: Infra and runtime configuration source of truth.
IMPLEMENTATION_BOUNDARY: Config loading and typed accessors only.
ACCEPTANCE_CRITERIA: Key resolution deterministic across envs.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: CORE-003
TITLE: Define DI container strategy and service registration contracts
DOMAIN: Core
TYPE: core
OWNER_AGENT: CoreAgent
EXECUTION_MODE: sequential
PARALLEL_GROUP: none
DEPENDS_ON: [CORE-001, CORE-002]
BLOCKS: [INFRA-DB-001, INFRA-LOG-001, INFRA-EVENT-001, INFRA-CACHE-001, INFRA-QUEUE-001, AUTH-001]
INPUT_CONTRACTS: [Service interface conventions]
OUTPUT_CONTRACTS: [Container binding/resolution contract]
FILES_EXPECTED_TO_CHANGE: [app/Core/Infrastructure/DI/Container.php, app/Contracts/Services/ServiceProviderInterface.php, app/Core/Application/Kernel.php]
RATIONALE: Enables interface-first runtime composition.
IMPLEMENTATION_BOUNDARY: Registration/resolution only.
ACCEPTANCE_CRITERIA: Interface bindings resolvable at boot.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: INFRA-DB-001
TITLE: Establish DBAL connection factory and transaction boundary service
DOMAIN: Core Infrastructure
TYPE: infra
OWNER_AGENT: InfraAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-INFRA-A
DEPENDS_ON: [CORE-002, CORE-003]
BLOCKS: [USERS-001, FORUM-001, PROJECTS-001, NEWS-001, DIGITALID-001, FORMS-001, SCHOLARSHIPS-001, ADS-001, ADMIN-001]
INPUT_CONTRACTS: [DB config contract]
OUTPUT_CONTRACTS: [DatabaseInterface transaction contract]
FILES_EXPECTED_TO_CHANGE: [app/Core/Infrastructure/Database/ConnectionFactory.php, app/Core/Infrastructure/Database/TransactionManager.php, app/Contracts/Services/DatabaseInterface.php, app/Core/Infrastructure/DI/Container.php]
RATIONALE: Shared persistence boundary for all modules.
IMPLEMENTATION_BOUNDARY: No module repository logic.
ACCEPTANCE_CRITERIA: Deterministic transaction boundary callable.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: INFRA-LOG-001
TITLE: Configure structured logging channels with Monolog
DOMAIN: Core Infrastructure
TYPE: infra
OWNER_AGENT: InfraAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-INFRA-A
DEPENDS_ON: [CORE-002, CORE-003]
BLOCKS: []
INPUT_CONTRACTS: [Logging envelope]
OUTPUT_CONTRACTS: [LoggerInterface + channel map]
FILES_EXPECTED_TO_CHANGE: [app/Core/Infrastructure/Logging/LoggerFactory.php, app/Contracts/Services/LoggerInterface.php, app/Bootstrap/WebRuntime.php, app/Bootstrap/WorkerRuntime.php]
RATIONALE: Observability baseline.
IMPLEMENTATION_BOUNDARY: Channel and formatter setup only.
ACCEPTANCE_CRITERIA: correlation_id and actor_id carried.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: INFRA-EVENT-001
TITLE: Define event envelope, sync dispatcher, and publisher interfaces
DOMAIN: Core Infrastructure
TYPE: infra
OWNER_AGENT: InfraAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-INFRA-A
DEPENDS_ON: [CORE-003]
BLOCKS: [INFRA-QUEUE-001, AUTH-001, USERS-001, FORUM-001, PROJECTS-001, NEWS-001, DIGITALID-001, FORMS-001, SCHOLARSHIPS-001, ADS-001, NOTIFICATIONS-001, ADMIN-001]
INPUT_CONTRACTS: [Event envelope schema]
OUTPUT_CONTRACTS: [EventPublisherInterface + dispatcher contract]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Events/EventEnvelope.php, app/Core/Infrastructure/Event/EventDispatcher.php, app/Contracts/Services/EventPublisherInterface.php, app/Core/Infrastructure/DI/Container.php]
RATIONALE: Event-driven integration backbone.
IMPLEMENTATION_BOUNDARY: Envelope + sync dispatch only.
ACCEPTANCE_CRITERIA: Listeners receive full required metadata.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: INFRA-CACHE-001
TITLE: Define cache store and lock-assisted invalidation contracts
DOMAIN: Core Infrastructure
TYPE: infra
OWNER_AGENT: InfraAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-INFRA-A
DEPENDS_ON: [CORE-003]
BLOCKS: [INTEGRATION-002]
INPUT_CONTRACTS: [Cache read-model requirements]
OUTPUT_CONTRACTS: [CacheStoreInterface contract]
FILES_EXPECTED_TO_CHANGE: [app/Core/Infrastructure/Cache/CacheStore.php, app/Contracts/Services/CacheStoreInterface.php, app/Core/Infrastructure/DI/Container.php]
RATIONALE: Read-model performance and deterministic invalidation.
IMPLEMENTATION_BOUNDARY: Contract + base adapter only.
ACCEPTANCE_CRITERIA: remember semantics deterministic.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: INFRA-QUEUE-001
TITLE: Define queue dispatcher, retry policy, and DLQ contracts
DOMAIN: Core Infrastructure
TYPE: infra
OWNER_AGENT: InfraAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-INFRA-B
DEPENDS_ON: [CORE-003, INFRA-EVENT-001]
BLOCKS: [NOTIFICATIONS-001]
INPUT_CONTRACTS: [Queue + idempotency requirements]
OUTPUT_CONTRACTS: [QueueDispatcherInterface, retry, DLQ]
FILES_EXPECTED_TO_CHANGE: [app/Core/Infrastructure/Queue/QueueDispatcher.php, app/Core/Infrastructure/Queue/RetryPolicy.php, app/Contracts/Services/QueueDispatcherInterface.php, app/Core/Infrastructure/DI/Container.php]
RATIONALE: Async processing safety.
IMPLEMENTATION_BOUNDARY: Queue abstraction only.
ACCEPTANCE_CRITERIA: Retry then DLQ path contractually defined.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: AUTH-001
TITLE: Define Auth module contracts for Google JWT verification
DOMAIN: Auth Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-B
DEPENDS_ON: [CORE-003, INFRA-EVENT-001]
BLOCKS: [ADMIN-001, INTEGRATION-001]
INPUT_CONTRACTS: [Security/API conventions]
OUTPUT_CONTRACTS: [Auth API contract, auth.user_authenticated schema, AuthIdentityResolverInterface]
FILES_EXPECTED_TO_CHANGE: [app/Modules/Auth/Application/Contracts/VerifyGoogleTokenCommand.php, app/Contracts/Api/Auth/verify_google_token.json, app/Contracts/Services/AuthIdentityResolverInterface.php, app/Contracts/Events/auth.user_authenticated.json, routes/api.php]
RATIONALE: Identity entry point and auth contract anchor.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: malformed JWT claim contract tests fail.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: USERS-001
TITLE: Define Users module contracts, schema, and leaderboard interfaces
DOMAIN: Users Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-B
DEPENDS_ON: [INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [FORUM-001, PROJECTS-001, NEWS-001, DIGITALID-001, SCHOLARSHIPS-001, NOTIFICATIONS-001, ADMIN-001, INTEGRATION-001, INTEGRATION-002]
INPUT_CONTRACTS: [API/DTO/DB conventions]
OUTPUT_CONTRACTS: [Users API contracts, users.user_registered.json, users.points_awarded.json, users.user_deactivated.json (MCL-02)]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Users/get_user.json, app/Contracts/Api/Users/update_user.json, app/Contracts/Api/Users/get_leaderboard.json, app/Contracts/Events/users.user_registered.json, app/Contracts/Events/users.points_awarded.json, app/Contracts/Events/users.user_deactivated.json, database/migrations/users_001.sql, routes/api.php]
RATIONALE: Foundational identity and points contracts used widely.
IMPLEMENTATION_BOUNDARY: Contracts + migration only.
ACCEPTANCE_CRITERIA: migration lint passes; leaderboard contract stable.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: FORUM-001
TITLE: Define Forum module contracts and schemas
DOMAIN: Forum Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-C
DEPENDS_ON: [USERS-001, INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [User identity + users.user_deactivated]
OUTPUT_CONTRACTS: [Forum API + topic/reply event schemas]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Forum/create_topic.json, app/Contracts/Api/Forum/add_reply.json, app/Contracts/Events/forum.topic_created.json, app/Contracts/Events/forum.reply_added.json, database/migrations/forum_001.sql, routes/api.php]
RATIONALE: Community discussion capability + points/moderation events.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: topic/reply event schema tests pass.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: PROJECTS-001
TITLE: Define Projects module contracts and schemas
DOMAIN: Projects Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-C
DEPENDS_ON: [USERS-001, INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [User ownership + users.user_deactivated]
OUTPUT_CONTRACTS: [Projects API + project_published event schema]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Projects/create_project.json, app/Contracts/Api/Projects/publish_project.json, app/Contracts/Events/projects.project_published.json, database/migrations/projects_001.sql, routes/api.php]
RATIONALE: Project showcase and publish lifecycle integration.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: publish state transition contract enforced.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: NEWS-001
TITLE: Define News module contracts, comments, and publish workflows
DOMAIN: News Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-C
DEPENDS_ON: [USERS-001, INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [User identity + users.user_deactivated]
OUTPUT_CONTRACTS: [News API + published/comment event schemas]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/News/create_news.json, app/Contracts/Api/News/publish_news.json, app/Contracts/Api/News/add_comment.json, app/Contracts/Events/news.published.json, app/Contracts/Events/news.comment_added.json, database/migrations/news_001.sql, routes/api.php]
RATIONALE: Publishing workflow + moderation/event fan-out.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: publish/comment payload schemas validate.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: DIGITALID-001
TITLE: Define DigitalID module contracts and generation events
DOMAIN: DigitalID Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-C
DEPENDS_ON: [USERS-001, INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [users.user_registered, users.user_deactivated]
OUTPUT_CONTRACTS: [DigitalID API + generated event schema]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/DigitalID/generate_digital_id.json, app/Contracts/Events/digital_id.generated.json, database/migrations/digital_id_001.sql, routes/api.php]
RATIONALE: Identity artifact lifecycle integration.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: generated schema includes artifact path and checksum.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: FORMS-001
TITLE: Define dynamic form builder contracts and schemas
DOMAIN: Forms Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-B
DEPENDS_ON: [INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [SCHOLARSHIPS-001, INTEGRATION-001]
INPUT_CONTRACTS: [DTO validation conventions]
OUTPUT_CONTRACTS: [Forms API + submission_received event schema]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Forms/create_form.json, app/Contracts/Api/Forms/submit_form.json, app/Contracts/Events/forms.submission_received.json, database/migrations/forms_001.sql, routes/api.php]
RATIONALE: Reusable workflow foundation for scholarships and admin.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: submission schema validation deterministic.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: SCHOLARSHIPS-001
TITLE: Define Scholarships contracts, forms integration, and award events
DOMAIN: Scholarships Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-D
DEPENDS_ON: [FORMS-001, USERS-001, INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [forms.submission_received + users.user_deactivated]
OUTPUT_CONTRACTS: [Scholarships APIs + submitted/awarded event schemas]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Scholarships/apply.json, app/Contracts/Api/Scholarships/award.json, app/Contracts/Events/scholarships.application_submitted.json, app/Contracts/Events/scholarships.application_awarded.json, database/migrations/scholarships_001.sql, routes/api.php]
RATIONALE: Cross-module workflow with strict state transitions.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: award requires existing submitted application state.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: ADS-001
TITLE: Define Ads contracts, targeting metadata, and metrics events
DOMAIN: Ads Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-B
DEPENDS_ON: [INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [Optional viewer identity contract]
OUTPUT_CONTRACTS: [Ads API + impression/click event schemas]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Ads/create_campaign.json, app/Contracts/Api/Ads/get_placement.json, app/Contracts/Events/ads.impression_recorded.json, app/Contracts/Events/ads.click_recorded.json, database/migrations/ads_001.sql, routes/api.php]
RATIONALE: Ad delivery + analytics contracts.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: payloads support anonymous/authenticated viewers.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: NOTIFICATIONS-001
TITLE: Define Notifications contracts and queue integration surfaces
DOMAIN: Notifications Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-C
DEPENDS_ON: [INFRA-QUEUE-001, INFRA-EVENT-001, USERS-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [Queue + event + user contracts]
OUTPUT_CONTRACTS: [Notifications APIs + queued/delivered events]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Notifications/get_preferences.json, app/Contracts/Api/Notifications/update_preferences.json, app/Contracts/Events/notifications.message_queued.json, app/Contracts/Events/notifications.message_delivered.json, database/migrations/notifications_001.sql, routes/api.php]
RATIONALE: Cross-domain communication endpoint.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: queue payload includes idempotency key.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: ADMIN-001
TITLE: Define Admin moderation and audit contracts
DOMAIN: Admin Module
TYPE: module
OWNER_AGENT: ModuleAgent
EXECUTION_MODE: parallel
PARALLEL_GROUP: P-MODULE-C
DEPENDS_ON: [USERS-001, AUTH-001, INFRA-DB-001, INFRA-EVENT-001]
BLOCKS: [INTEGRATION-001]
INPUT_CONTRACTS: [Role/policy/audit contracts]
OUTPUT_CONTRACTS: [Admin dashboard/moderation APIs + moderation event schema]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Api/Admin/get_dashboard.json, app/Contracts/Api/Admin/resolve_moderation_case.json, app/Contracts/Events/admin.moderation_case_resolved.json, database/migrations/admin_001.sql, routes/api.php]
RATIONALE: Operational governance and audit surface.
IMPLEMENTATION_BOUNDARY: Contract definitions only.
ACCEPTANCE_CRITERIA: unauthorized actors denied by contract tests.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: INTEGRATION-001
TITLE: Implement cross-module event routing matrix and consumer bindings
DOMAIN: Integration
TYPE: integration
OWNER_AGENT: IntegrationAgent
EXECUTION_MODE: sequential
PARALLEL_GROUP: none
DEPENDS_ON: [AUTH-001, USERS-001, FORUM-001, PROJECTS-001, NEWS-001, DIGITALID-001, FORMS-001, SCHOLARSHIPS-001, ADS-001, NOTIFICATIONS-001, ADMIN-001]
BLOCKS: [INTEGRATION-002, QA-CONTRACT-001]
INPUT_CONTRACTS: [All module event schemas]
OUTPUT_CONTRACTS: [subscription_matrix.md + deterministic consumer map]
FILES_EXPECTED_TO_CHANGE: [app/Bootstrap/EventSubscriptions.php, app/Contracts/Events/subscription_matrix.md, app/Core/Application/Kernel.php]
RATIONALE: System-wide integration convergence barrier.
IMPLEMENTATION_BOUNDARY: subscription and binding declarations only.
ACCEPTANCE_CRITERIA: all consumers validate against schema.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: INTEGRATION-002
TITLE: Define leaderboard aggregation read model contract
DOMAIN: Integration
TYPE: integration
OWNER_AGENT: IntegrationAgent
EXECUTION_MODE: sequential
PARALLEL_GROUP: none
DEPENDS_ON: [USERS-001, INTEGRATION-001, INFRA-CACHE-001]
BLOCKS: [QA-CONTRACT-001]
INPUT_CONTRACTS: [users.points_awarded + contributor events + existing Users leaderboard API contract]
OUTPUT_CONTRACTS: [LeaderboardQueryInterface, leaderboard.rebuilt event schema, subscription matrix update]
FILES_EXPECTED_TO_CHANGE: [app/Contracts/Services/LeaderboardQueryInterface.php, app/Contracts/Events/leaderboard.rebuilt.json, app/Contracts/Events/subscription_matrix.md]
RATIONALE: Deterministic ranking read model integration.
IMPLEMENTATION_BOUNDARY: contract-level aggregation semantics only.
ACCEPTANCE_CRITERIA: tie-break policy deterministic and documented.
REVIEW_REQUIRED: yes
QA_REQUIRED: yes

TASK_ID: QA-CONTRACT-001
TITLE: Add platform-wide contract test harness
DOMAIN: Platform QA
TYPE: qa
OWNER_AGENT: QAAgent
EXECUTION_MODE: sequential
PARALLEL_GROUP: none
DEPENDS_ON: [INTEGRATION-001, INTEGRATION-002]
BLOCKS: []
INPUT_CONTRACTS: [All API/event/schema contracts]
OUTPUT_CONTRACTS: [Executable deterministic contract verification baseline]
FILES_EXPECTED_TO_CHANGE: [tests/Contract/ApiContractTest.php, tests/Contract/EventContractTest.php, tests/Contract/MigrationContractTest.php, composer.json]
RATIONALE: Final quality gate for contract-first delivery.
IMPLEMENTATION_BOUNDARY: Harness and checks only.
ACCEPTANCE_CRITERIA: single command validates all contracts deterministically.
REVIEW_REQUIRED: yes
QA_REQUIRED: no

---

# Phase Execution Plan

## Phase 1: Core Foundation

**Entry Criteria**
- Repository initialized.
- Blueprint and MCL accepted.

**Tasks (ordered)**
1. CORE-001
2. CORE-002
3. CORE-003

**Parallel Groups**
- none

**Completion Criteria**
- Runtime and DI boot contracts complete and review-passed.

**Output Artifacts**
- Kernel + runtime contracts + config + DI contracts.

## Phase 2: Infrastructure Backbone

**Entry Criteria**
- Phase 1 complete.

**Tasks**
- Parallel group `P-INFRA-A`: INFRA-DB-001, INFRA-LOG-001, INFRA-EVENT-001, INFRA-CACHE-001
- Then `P-INFRA-B`: INFRA-QUEUE-001

**Synchronization Barrier**
- Barrier B1: `INFRA-EVENT-001` must complete before `INFRA-QUEUE-001` and event-dependent modules.

**Completion Criteria**
- DB, logging, event, cache, queue contracts are available and validated.

**Output Artifacts**
- Shared infra interfaces and runtime bindings.

## Phase 3: Parallel Module Contract Build

**Entry Criteria**
- Phase 2 complete.

**Tasks / Groups**
- `P-MODULE-B`: AUTH-001, USERS-001, FORMS-001, ADS-001
- `P-MODULE-C`: FORUM-001, PROJECTS-001, NEWS-001, DIGITALID-001, NOTIFICATIONS-001, ADMIN-001 (respect per-task deps)
- `P-MODULE-D`: SCHOLARSHIPS-001

**Synchronization Barrier**
- Barrier B2: All module tasks complete and review-passed before integration phase.

**Completion Criteria**
- All module APIs/events/migrations/contracts are declared and internally consistent.

**Output Artifacts**
- Contract surfaces per module and migration specs.

## Phase 4: Integration

**Entry Criteria**
- B2 achieved.

**Tasks (ordered)**
1. INTEGRATION-001
2. INTEGRATION-002

**Synchronization Barrier**
- Barrier B3: Integration contracts complete before global QA harness.

**Completion Criteria**
- Event subscription matrix complete and leaderboard integration contract fixed.

**Output Artifacts**
- System subscription matrix and leaderboard integration contracts.

## Phase 5: Validation and Hardening

**Entry Criteria**
- B3 achieved.

**Tasks (ordered)**
1. QA-CONTRACT-001

**Completion Criteria**
- Deterministic single-command contract verification established.

**Output Artifacts**
- Platform contract test harness and CI-ready verification command.

---

# Parallelization Matrix

| Group | Tasks | Can Run Concurrently | Barrier Exit |
|---|---|---|---|
| P-INFRA-A | INFRA-DB-001, INFRA-LOG-001, INFRA-EVENT-001, INFRA-CACHE-001 | Yes | All complete |
| P-INFRA-B | INFRA-QUEUE-001 | Single (depends on INFRA-EVENT-001) | Complete |
| P-MODULE-B | AUTH-001, USERS-001, FORMS-001, ADS-001 | Yes (per deps) | All complete |
| P-MODULE-C | FORUM-001, PROJECTS-001, NEWS-001, DIGITALID-001, NOTIFICATIONS-001, ADMIN-001 | Yes (per deps) | All complete |
| P-MODULE-D | SCHOLARSHIPS-001 | Single (depends on FORMS-001 + USERS-001) | Complete |

---

# Subagent Role Map

- **CoreAgent**: CORE-001/002/003
- **InfraAgent**: INFRA-DB-001, INFRA-LOG-001, INFRA-EVENT-001, INFRA-CACHE-001, INFRA-QUEUE-001
- **ModuleAgent**: AUTH-001, USERS-001, FORUM-001, PROJECTS-001, NEWS-001, DIGITALID-001, FORMS-001, SCHOLARSHIPS-001, ADS-001, NOTIFICATIONS-001, ADMIN-001
- **IntegrationAgent**: INTEGRATION-001, INTEGRATION-002
- **ReviewAgent**: mandatory review for every task (`REVIEW_REQUIRED: yes`)
- **QAAgent**: QA validation per task with final authority on QA-CONTRACT-001

Ownership policy:
- Build output accepted only if matching task contracts and isolation rules.
- ReviewAgent cannot broaden scope; only enforce boundary and contract adherence.
- QAAgent validates deterministic behavior and integration safety.

---

# Desktop Prompt Pack

> Use each prompt exactly for the mapped task. Replace `{TASK_ID}` with the concrete task.

## Prompt Template Contract (applies to every task)

- Do not implement tasks outside declared scope.
- Do not access another module’s persistence directly.
- Use only declared contracts/events/interfaces.
- If dependency artifacts are missing, stop and report blocker.

## Task Prompt Triplets

### CORE-001
BUILD PROMPT
"You are CoreAgent executing CORE-001. Purpose: establish kernel/module loader/runtime entry contracts. Change only: app/Core/Application/Kernel.php, app/Bootstrap/WebRuntime.php, app/Bootstrap/WorkerRuntime.php, app/Bootstrap/SchedulerRuntime.php, app/Contracts/Services/ModuleInterface.php, public/index.php. Assume no dependencies. Enforce runtime correlation ID emission and no module-specific logic. Acceptance: web/worker/scheduler boot without module implementation. Forbidden: module business logic, cross-module coupling, undocumented contracts. Provide completion notes mapped to acceptance criteria."

REVIEW PROMPT
"You are ReviewAgent reviewing CORE-001. Verify architecture boundary (core-only), contract completeness for module registration/runtime startup, and correlation ID requirement. Confirm no module-specific logic leaked into kernel/runtime files. Reject if acceptance criteria are not explicitly evidenced."

QA PROMPT
"You are QAAgent validating CORE-001. Validate deterministic boot behavior for web/worker/scheduler with empty modules. Verify correlation_id presence in runtime startup context. Report pass/fail by acceptance criterion with blocker details."

### CORE-002
BUILD PROMPT
"You are CoreAgent executing CORE-002. Purpose: config subsystem with precedence defaults < env file < process env. Change only declared config and runtime files. Dependency assumed complete: CORE-001. Acceptance: deterministic same-key resolution across dev/staging/prod. Forbidden: hard-coded environment branching outside config layer."

REVIEW PROMPT
"Review CORE-002 for precedence correctness, typed accessor contract clarity, and runtime integration correctness. Ensure no config retrieval bypasses contract interface."

QA PROMPT
"Validate CORE-002 with matrix checks across dev/staging/prod; assert deterministic precedence and stable key resolution behavior."

### CORE-003
BUILD PROMPT
"Execute CORE-003 as CoreAgent. Define DI container and ServiceProviderInterface, update kernel wiring only. Dependencies complete: CORE-001, CORE-002. Acceptance: interface bindings resolve at runtime boot. Forbidden: domain-layer service location usage."

REVIEW PROMPT
"Review CORE-003 for interface-first registration, explicit module entry service registration, and prohibition of domain container access."

QA PROMPT
"QA CORE-003 by validating runtime boot resolves declared interfaces and fails predictably on missing bindings."

### INFRA-DB-001
BUILD PROMPT
"Execute INFRA-DB-001 as InfraAgent. Establish DB connection factory, transaction manager, DatabaseInterface contract, and DI registration. Dependencies: CORE-002, CORE-003. Acceptance: transaction boundary callable from dummy command service. Forbidden: module repository code."

REVIEW PROMPT
"Review INFRA-DB-001 for DB abstraction purity, transaction boundary determinism, and absence of module persistence coupling."

QA PROMPT
"Validate INFRA-DB-001 transaction begin/commit/rollback contract behavior deterministically and confirm DI registration."

### INFRA-LOG-001
BUILD PROMPT
"Execute INFRA-LOG-001 as InfraAgent. Configure logger factory and LoggerInterface with required channels app/audit/security/queue/integration; wire into web and worker runtime only. Dependencies: CORE-002, CORE-003. Acceptance: sample logs carry correlation_id and actor_id when available."

REVIEW PROMPT
"Review INFRA-LOG-001 for channel completeness, structured envelope compliance, and runtime hookup boundaries."

QA PROMPT
"Validate INFRA-LOG-001 by checking channel outputs and required metadata fields under deterministic test emissions."

### INFRA-EVENT-001
BUILD PROMPT
"Execute INFRA-EVENT-001 as InfraAgent. Define event envelope contract, sync event dispatcher, publisher interface, and DI wiring. Dependency: CORE-003. Acceptance: listeners receive required envelope metadata exactly. Forbidden: async queue behavior in this task."

REVIEW PROMPT
"Review INFRA-EVENT-001 for strict required-field enforcement and schema-consistent envelope structure."

QA PROMPT
"Validate INFRA-EVENT-001 with listener assertions for event_id, event_name, occurred_at, actor_id, correlation_id, causation_id, payload."

### INFRA-CACHE-001
BUILD PROMPT
"Execute INFRA-CACHE-001 as InfraAgent. Define CacheStore contract and remember semantics with lock-aware stampede controls. Dependency: CORE-003. Acceptance: deterministic hit/miss behavior."

REVIEW PROMPT
"Review INFRA-CACHE-001 for contract clarity, lock semantics, and no module-specific cache policy leakage."

QA PROMPT
"Validate INFRA-CACHE-001 remember behavior, TTL handling, and deterministic miss→fill→hit flow."

### INFRA-QUEUE-001
BUILD PROMPT
"Execute INFRA-QUEUE-001 as InfraAgent. Implement queue dispatcher, retry policy, DLQ contracts, and DI registration. Dependencies: CORE-003, INFRA-EVENT-001. Acceptance: failed job transitions retry then DLQ; idempotency key required."

REVIEW PROMPT
"Review INFRA-QUEUE-001 for idempotency enforcement, retry determinism, and DLQ contract completeness."

QA PROMPT
"Validate INFRA-QUEUE-001 with controlled failing jobs proving retry count and DLQ transition contracts."

### AUTH-001
BUILD PROMPT
"Execute AUTH-001 as ModuleAgent. Define Google JWT auth contracts only (API schema, command DTO, interface, auth event schema, route contract). Dependencies: CORE-003, INFRA-EVENT-001. Acceptance: malformed iss/aud/exp/sub claims rejected by contract tests. Forbidden: provider SDK integration beyond contract definitions."

REVIEW PROMPT
"Review AUTH-001 for claim contract completeness, route scope accuracy, and event payload correctness."

QA PROMPT
"Validate AUTH-001 contract test cases for malformed claim combinations and accepted schema shape."

### USERS-001
BUILD PROMPT
"Execute USERS-001 as ModuleAgent. Define users/profile/roles/points contracts, migration, routes, and event schemas including users.user_deactivated (MCL-02). Dependencies: INFRA-DB-001, INFRA-EVENT-001. Acceptance: migration lints; leaderboard contract deterministic."

REVIEW PROMPT
"Review USERS-001 for schema conventions, event coverage completeness, and ownership of leaderboard API contract (no duplication elsewhere)."

QA PROMPT
"Validate USERS-001 migration naming/schema checks and event schema validity including user_deactivated."

### FORUM-001
BUILD PROMPT
"Execute FORUM-001 as ModuleAgent. Define forum API/event contracts and migration, with dependence on users identity contract and user_deactivated consumption contract. Dependencies: USERS-001, INFRA-DB-001, INFRA-EVENT-001. Acceptance: topic/reply payload schemas pass contract tests."

REVIEW PROMPT
"Review FORUM-001 for no direct users table write assumptions and proper event payload declarations."

QA PROMPT
"Validate FORUM-001 create topic/add reply schema validations and emitted event payload contract compliance."

### PROJECTS-001
BUILD PROMPT
"Execute PROJECTS-001 as ModuleAgent. Define project lifecycle API/event contracts and migration; enforce explicit publish state transition contract. Dependencies: USERS-001, INFRA-DB-001, INFRA-EVENT-001."

REVIEW PROMPT
"Review PROJECTS-001 for state-machine contract completeness and ownership boundary adherence."

QA PROMPT
"Validate PROJECTS-001 contract tests for create/publish transitions and event payload schema."

### NEWS-001
BUILD PROMPT
"Execute NEWS-001 as ModuleAgent. Define news publish/comment contracts and migration with user_deactivated consumption compatibility. Dependencies: USERS-001, INFRA-DB-001, INFRA-EVENT-001."

REVIEW PROMPT
"Review NEWS-001 for draft→published workflow contract and comment payload/event schema completeness."

QA PROMPT
"Validate NEWS-001 contract schemas for publish and comment flows and event conformity."

### DIGITALID-001
BUILD PROMPT
"Execute DIGITALID-001 as ModuleAgent. Define digital ID generation API/event contracts and migration; include artifact checksum requirement. Dependencies: USERS-001, INFRA-DB-001, INFRA-EVENT-001."

REVIEW PROMPT
"Review DIGITALID-001 for required artifact_path/checksum fields and lifecycle contract alignment."

QA PROMPT
"Validate DIGITALID-001 generated event schema includes required fields and API schema consistency."

### FORMS-001
BUILD PROMPT
"Execute FORMS-001 as ModuleAgent. Define form definition/submission contracts/events and migration with schema-validated JSON payload boundary. Dependencies: INFRA-DB-001, INFRA-EVENT-001."

REVIEW PROMPT
"Review FORMS-001 for versioned contract design and submission validation constraints."

QA PROMPT
"Validate FORMS-001 required fields/type constraints and forms.submission_received event contract."

### SCHOLARSHIPS-001
BUILD PROMPT
"Execute SCHOLARSHIPS-001 as ModuleAgent. Define scholarship apply/award contracts/events and migration referencing form_submission_id contract only. Dependencies: FORMS-001, USERS-001, INFRA-DB-001, INFRA-EVENT-001."

REVIEW PROMPT
"Review SCHOLARSHIPS-001 for explicit state preconditions (submitted before awarded) and event schema completeness."

QA PROMPT
"Validate SCHOLARSHIPS-001 apply/award contract flow and failure on invalid state transitions."

### ADS-001
BUILD PROMPT
"Execute ADS-001 as ModuleAgent. Define ads campaign/placement/click contracts/events and migration supporting nullable viewer_id for anonymous traffic. Dependencies: INFRA-DB-001, INFRA-EVENT-001."

REVIEW PROMPT
"Review ADS-001 for authenticated/anonymous parity and analytics event schema precision."

QA PROMPT
"Validate ADS-001 schema permutations for anonymous/authenticated impression and click events."

### NOTIFICATIONS-001
BUILD PROMPT
"Execute NOTIFICATIONS-001 as ModuleAgent. Define preferences/message delivery contracts and migration integrated with queue idempotency contract. Dependencies: INFRA-QUEUE-001, INFRA-EVENT-001, USERS-001."

REVIEW PROMPT
"Review NOTIFICATIONS-001 for queue contract adherence, provider_ref requirement, and event schema consistency."

QA PROMPT
"Validate NOTIFICATIONS-001 queue payload includes idempotency key and delivery event schemas are valid."

### ADMIN-001
BUILD PROMPT
"Execute ADMIN-001 as ModuleAgent. Define admin dashboard/moderation/audit contracts and migration with policy contract alignment. Dependencies: USERS-001, AUTH-001, INFRA-DB-001, INFRA-EVENT-001."

REVIEW PROMPT
"Review ADMIN-001 for authorization contract enforcement and mandatory audit event on mutations."

QA PROMPT
"Validate ADMIN-001 denial behavior for unauthorized actors and moderation resolution event schema."

### INTEGRATION-001
BUILD PROMPT
"Execute INTEGRATION-001 as IntegrationAgent. Build event subscription matrix and consumer bindings only, using declared schemas from all modules. Dependencies: all module tasks complete. Acceptance: every consumer binds only to declared payload fields and matrix is complete. Forbidden: introducing new event fields."

REVIEW PROMPT
"Review INTEGRATION-001 for matrix completeness, declared consumer parity, and schema-only payload usage."

QA PROMPT
"Validate INTEGRATION-001 with contract checks that each subscription maps to an existing event schema and declared consumer."

### INTEGRATION-002
BUILD PROMPT
"Execute INTEGRATION-002 as IntegrationAgent. Define leaderboard read-model integration contracts using existing Users leaderboard API contract (MCL-01), add LeaderboardQueryInterface, leaderboard.rebuilt event schema, and subscription matrix updates. Dependencies: USERS-001, INTEGRATION-001, INFRA-CACHE-001."

REVIEW PROMPT
"Review INTEGRATION-002 for no duplicate leaderboard API contract creation, tie-break policy determinism, and cache invalidation trigger boundaries."

QA PROMPT
"Validate INTEGRATION-002 deterministic ranking contract and correct dependency on points-affecting events only."

### QA-CONTRACT-001
BUILD PROMPT
"Execute QA-CONTRACT-001 as QAAgent. Build platform-wide contract harness for API/event/migration checks and single deterministic command execution. Dependencies: INTEGRATION-001, INTEGRATION-002."

REVIEW PROMPT
"Review QA-CONTRACT-001 for coverage completeness across all declared contracts and failure behavior on undeclared/missing fields."

QA PROMPT
"Run QA-CONTRACT-001 validation suite and report deterministic pass/fail outcomes by contract family (API, event, migration)."

---

# Integration Checkpoints

1. **Checkpoint IC-1 (Post-Infra):**
   - Event, queue, DB, cache interfaces available.
   - Correlation envelope policy enforced.

2. **Checkpoint IC-2 (Post-Module):**
   - Every module exports API/event contracts and migration artifact.
   - All consumed cross-module events have schema files.

3. **Checkpoint IC-3 (Post-Integration-001):**
   - Subscription matrix complete and bidirectionally traceable:
     - producer event → consumer list
     - consumer binding → source schema.

4. **Checkpoint IC-4 (Post-Integration-002):**
   - Leaderboard aggregation contract deterministic.
   - No duplicate contract ownership.

5. **Checkpoint IC-5 (Final QA):**
   - Single command validates all contracts.
   - No undeclared fields or missing required fields.

---

# Final Execution Instructions

1. Apply phases strictly in order; do not skip barriers.
2. Within parallel groups, tasks may execute concurrently only after dependencies are marked complete.
3. Every task must run Build → Review → QA sequence before marked done.
4. If a task fails review/QA, re-run only that task and downstream dependents if output contracts changed.
5. Enforce MCL corrections as mandatory orchestration policy.
6. Do not add features beyond blueprint + MCL.
7. Treat this packet as sole orchestration authority for desktop subagent execution.

