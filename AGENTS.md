# AGENTS.md

## 1) System Overview
- Build and maintain a **modular monolith** for the Village Community Digital Platform.
- Enforce **DDD-style module boundaries** and **event-driven orchestration**.
- Deliver changes **contract-first**: define/align contracts before implementation.
- Preserve layered architecture: Presentation → Application → Domain → Infrastructure.

## 2) Execution Model
- Work contract-first: APIs, DTOs, events, and service interfaces are authoritative.
- Execute one declared task per run; do not blend unrelated tasks.
- Respect dependency order: complete blockers before dependents.
- Use subagents by role (core/infra/module/integration/review/qa) and keep ownership explicit.
- Treat integration and QA gates as required phase barriers.

## 3) Hard Constraints (Non-Negotiable)
- Never access another module’s database tables/repositories.
- Use only APIs, service contracts, or events for inter-module communication.
- Never violate published contracts (DTO shape, API schema, event schema, service interface).
- Do not introduce hidden dependencies (implicit imports, runtime service location, undeclared coupling).
- Do not redesign architecture while executing implementation tasks.
- Prevent global state leakage across modules/runtimes.

## 4) Module Boundaries
- Keep each module isolated in domain, application, and infrastructure logic.
- Allowed interactions:
  - Core + Contracts usage by modules.
  - Cross-module via public service interfaces, public APIs, or events.
- Forbidden interactions:
  - Direct module-to-module repository/ORM/table access.
  - Reaching into another module’s internal classes.
  - Sharing mutable module state across boundaries.

## 5) Task Execution Rules
- Enforce one-task-per-run discipline with explicit scope.
- Verify all declared dependencies are satisfied before coding.
- Validate before completion:
  - Contract conformance checks.
  - Dependency/ownership checks.
  - Integration impact checks for touched modules.
- If a required contract is missing or ambiguous, stop and define/clarify contract first.

## 6) Code & Architecture Standards
- Maintain strict layering; no layer skipping.
- Domain layer must remain framework/infrastructure-agnostic.
- Application services orchestrate use-cases; repositories only persist/query.
- Repositories must not contain cross-module orchestration logic.
- Use DTOs for all boundary crossings; do not expose domain entities externally.
- Use events for cross-module side effects and eventual consistency.

## 7) Infrastructure Rules
- All data access goes through DBAL abstractions and transaction boundaries.
- Enforce JWT authentication/authorization rules for protected operations.
- Emit structured logs with correlation and actor context for all critical flows.
- Use cache only for read/performance concerns; invalidation must be event-driven.
- Use queues for async/heavy side effects with retry and dead-letter handling.

## 8) Event System Rules
- Emit events for domain-significant state changes that other modules may consume.
- Event names must follow `module.event_action` (e.g., `users.user_deactivated`).
- Every event must include required envelope metadata:
  - `event_id`, `event_name`, `occurred_at`, `actor_id`, `correlation_id`, `causation_id`, `payload`.
- Payloads must conform to versioned schemas; consumers must bind to schema, not ad hoc fields.

## 9) Review & QA Expectations
- Mandatory review checks:
  - Boundary compliance.
  - Contract compatibility.
  - Layering and dependency integrity.
- Mandatory QA checks:
  - Contract validation tests for changed APIs/events/DTOs.
  - Integration safety checks for emitting/consuming modules.
  - Regression checks for auth, logging, and transaction boundaries.
- No task is complete until review and QA gates pass.

## 10) Forbidden Behaviors (STRICT)
- Do not bypass contracts "temporarily".
- Do not patch another module’s internals to satisfy local needs.
- Do not add direct DB joins/queries across module ownership.
- Do not introduce shared mutable singletons or global caches crossing module boundaries.
- Do not change event names/payloads without contract update + consumer validation.
- Do not merge work that skips dependency order, review, or QA gates.
- Do not make undocumented coupling, hidden side effects, or architecture-level deviations.
