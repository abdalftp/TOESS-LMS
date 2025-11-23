<!--
Sync Impact Report

- Version change: 1.0.0 -> 2.0.0
- Modified principles:
	- Testing Approach: explicit removal of TDD requirement -> "No TDD; implementation-first testing with `NUnit`" (redefined)
	- .NET-First Library & API Design: clarified microservices guidance and data ownership
- Added sections:
	- Microservices & Data Ownership (guidance for shared databases and schema governance)
- Removed sections: none
- Templates requiring updates:
	- .specify/templates/plan-template.md : ⚠ pending — adapt "Constitution Check" to include updated testing guidance (no TDD) and .NET tooling (dotnet, analyzers, CI gates)
	- .specify/templates/spec-template.md : ⚠ pending — include mandatory "User Scenarios & Testing" gate and list `NUnit` as preferred framework
	- .specify/templates/tasks-template.md: ⚠ pending — add tasks for `dotnet` project init, analyzers, and coverage enforcement
	- .specify/templates/checklist-template.md: ⚠ pending — include C#/.NET checklist items (nullable, analyzers, tracing, health checks) and DB-sharing governance checks
	- .specify/templates/agent-file-template.md: ⚠ pending — populate "Code Style" with Roslyn/StyleCop rules for C# and .NET 9
- Follow-up TODOs:
	- Update the templates above to reference `dotnet` commands, recommended analyzers (Roslyn analyzers, StyleCop), and CI job examples.
	- Add an automated policy in CI that enforces code analysis and minimum coverage.
	- Add CI/PR checks to validate DB-sharing governance when multiple services use the same schema.
	- Confirm `RATIFICATION_DATE` with governance owners if different from initial adoption date.
-->

# TOESS LMS Constitution

## Core Principles

### 1. .NET-First Library & API Design
All new services and components MUST target .NET 9 (C# 12) and follow a library-first, API-driven design. Public interfaces (APIs and library surface) MUST be:
- Explicitly typed, async-friendly (use `async`/`await`), and designed for DI (Dependency Injection).
- Surface-minimal: expose only what callers need; prefer small, well-documented abstractions over large facades.
- Backed by contract tests for any public API consumed across service boundaries.

Rationale: A .NET-first standard reduces integration friction when converting modules into microservices and ensures consistent runtime behaviour across the system.

### 2. Testing Approach (No TDD)
The project WILL NOT use Test‑Driven Development (TDD) as a mandated practice. Teams will write implementation code first and add tests thereafter using the preferred test framework `NUnit`. While TDD techniques may be used voluntarily by individual developers, the constitutional requirement is an implementation‑first testing workflow with tests added and validated in CI prior to merge.

Testing requirements:
- Unit tests for all logic with dependency inversion and mocking where appropriate.
- Integration tests for data access, third‑party integrations, and inter-service contracts.
- Contract tests (consumer-driven or provider-driven) where APIs are shared between services.
- End-to-end tests for high‑risk user journeys described in the SRS.
- Minimum automated coverage threshold: 80% for business-critical assemblies; non-critical libraries SHOULD aim for 70%.

Rationale: The SRS defines critical educational workflows (assignments, attendance, timetables); mandatory automated tests and coverage thresholds reduce regressions and protect student data. Adopting an implementation-first workflow reflects team preference while preserving quality through CI enforcement and `NUnit`-based testing.

### 3. CI/CD & Automation Gates
Every change MUST pass an automated pipeline before merge. Required CI gates:
- Build on .NET SDK 9.0 and publish artifacts for downstream jobs.
- Static analysis using Roslyn analyzers and StyleCop; treat analyzer warnings as build failures for production branches.
- Unit, integration, and contract test jobs with enforced coverage thresholds.
- Security scanning (Snyk/Dependabot or equivalent) and secrets detection.
- Push artifacts to an image registry or package feed as part of release workflows.

Rationale: Automation ensures consistent quality, reduces manual review load, and catches regressions early.

### 4. Code Quality, Maintainability & Style
Code MUST be idiomatic C# and follow a single project-wide style enforced by `editorconfig`, StyleCop rules, and Roslyn analyzers. Requirements:
- Enable nullable reference types and follow null-safety practices.
- Use meaningful names, limit method length, and prefer composition over inheritance.
- Avoid global mutable state; prefer scoped services and immutable DTOs.
- All public APIs must include XML comments and be included in generated API docs.
- Treat all compiler warnings as errors in CI for `main`/`release` branches.

Rationale: Consistent style and static checks accelerate reviews and reduce cognitive load for maintainers.

### 5. Observability & Diagnostics
Every service MUST provide structured logging, health checks, metrics, and tracing:
- Use structured logging (Serilog or Microsoft.Extensions.Logging with JSON output) and include correlation IDs.
- Integrate OpenTelemetry for distributed tracing; export traces to a supported backend.
- Expose readiness and liveness probes and a `/health` endpoint compatible with orchestration platforms.
- Emit Prometheus-compatible metrics or provide metrics adapters.

Rationale: Observability is required for diagnosing issues in microservice deployments and ensuring SLA adherence for the LMS.

### 6. Security, Privacy & Compliance
Security is mandatory for all components handling user or student data. Requirements:
- Follow OWASP Top 10 mitigations and use parameterized queries or ORM protections to avoid injection.
- Store secrets using secure secret stores (Key Vault, AWS Secrets Manager, or platform equivalent); never commit secrets to source.
- Encrypt sensitive data at rest and in transit (TLS 1.2+ required).
- Apply role-based access control (RBAC) and least privilege for internal APIs.
- Comply with applicable K‑12 privacy standards where relevant (e.g., FERPA/COPPA considerations): document data flows and retention.

Rationale: The SRS emphasizes student and parent data; these protections are non-negotiable.

### 7. Versioning, Deprecation & Backward Compatibility
Services and public libraries MUST follow semantic versioning. API/versioning policy:
- Use SemVer for service packages: MAJOR.MINOR.PATCH.
- Introduce breaking API changes ONLY in a MAJOR version with an explicit deprecation schedule (minimum 90 days) and migration guide.
- Support side-by-side versioned APIs (e.g., `/api/v1/...`), and publish contract tests to prevent regressions.

Rationale: Predictable versioning enables safe evolution of microservices and clear upgrade paths for integrators.

## Technology Constraints & Standards
- Language & Runtime: C# targeting .NET 9.0 (LTS guidance to be confirmed by maintainers).
- Build Tools: `dotnet` CLI for builds, `dotnet test` for tests, `dotnet pack` / container images for releases.
- Recommended libraries: `Serilog`, `OpenTelemetry`, `Polly` for resilience, `FluentValidation` for input validation.
- Containerization: Build reproducible container images with multistage Dockerfiles; include explicit runtime user and minimal surface.
- Data Stores: SQL Server / PostgreSQL for transactional data; the choice per service must be documented in the plan.

- Microservices & Data Ownership
	- Multiple microservices MAY use the same physical database when necessary for operational or migration reasons, but such sharing MUST be governed and documented. Rules when sharing a database:
		- **Owner & Contract**: Each schema/table must have a single owning service; ownership and public contracts (column-level contracts, expected behaviours) MUST be documented in the service spec.
		- **Change Coordination**: Schema changes MUST follow a migration plan coordinated between owners; migrations that affect other services require agreed migration windows and backward-compatible change steps (versioned columns, feature flags, consumer migration order).
		- **Access Controls**: Services must use least-privilege credentials scoped to only the required schema/tables and operations.
		- **Anti-Corruption & API First**: Prefer calling the owning service's API for cross-service access; direct DB access by other services is allowed only when justified and documented, and must include tests and monitoring for coupling risks.
		- **Contracts & Tests**: Shared-schema contract tests or integration tests MUST be added to CI to catch breaking changes early.
		- **Operational Ownership**: Backups, retention, and restore responsibilities must be assigned; incident runbooks must include cross-service impact assessment.

Rationale: Allowing shared physical databases reduces operational complexity for some migration patterns and quicker integrations, but strict ownership, migration coordination, and testing are required to manage coupling and prevent regressions.

Rationale: Standardized stack reduces platform variance and accelerates onboarding.

## Development Workflow & Quality Gates
- Branching: Use `main` as the production branch; feature branches prefixed with `feat/` and hotfixes with `hotfix/`.
- PRs: Require at least one approving reviewer plus an automated green build. PRs must include:
	- Updated/added tests demonstrating behavior
	- Link to spec or SRS sections (from `docs/`)
	- Migration notes when changing persisted schemas
- Reviews: Focus on behavior, tests, readability, and security; reviewers MUST verify CI gates and test coverage changes.
- Releases: Production releases require a documented migration plan for database and contract changes.

Rationale: Clear workflow reduces merging friction and ensures changes are traceable to requirements.

## Governance
Amendments and enforcement:
- This constitution is authoritative for development practices. Changes to this document MUST be proposed as a pull request targeting `.specify/memory/constitution.md`.
- Approval: A proposed amendment requires approval from at least two maintainers, one of whom should be an architect or senior engineer.
- Migration: Material changes (breaking rules or new mandatory requirements) MUST include a migration plan and a timeline for adoption.

Versioning policy for the constitution itself:
- MAJOR: Backward-incompatible governance changes (remove or redefine principles).
- MINOR: New principle or material expansion of guidance.
- PATCH: Editorial fixes, clarifications, and non-substantive wording changes.

Compliance and reviews:
- Periodic review: The constitution SHOULD be reviewed at least once per year or when major architectural changes occur.
- PR enforcement: CI checks and PR templates SHOULD reference the relevant constitution sections for automated verification.

**Version**: 2.0.0 | **Ratified**: 2025-11-23 | **Last Amended**: 2025-11-23
