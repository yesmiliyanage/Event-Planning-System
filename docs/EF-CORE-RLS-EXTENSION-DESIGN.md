# EF Core RLS Extension Design

## Status

- Stage: Draft implementation blueprint
- Scope: Architecture and implementation guidance for a reusable EF Core extension
- Version baseline: TBD (intentionally version-agnostic)

## 1) Objective

Design a reusable EF Core extension that manages PostgreSQL Row-Level Security (RLS) as first-class, code-first schema artifacts.

The extension must:

- Define policies in code using strongly typed constructs.
- Persist policy metadata in the EF model and model snapshot.
- Automatically diff and generate migrations for policy create/alter/drop.
- Emit provider-aware SQL through EF migration SQL generators.
- Preserve deterministic and idempotent behavior across environments.
- Keep policy configuration out of domain models and inside infrastructure configuration.

## 2) Scope and non-goals

In scope:

- Reusable extension library design under the Common module.
- EF metadata, snapshot, migrations, and PostgreSQL execution path.
- Runtime contract for request identity to database session context handoff.
- Testing and quality gates for deterministic migration behavior.

Out of scope in this phase:

- Full policy rollout for all bounded contexts.
- Final auth provider implementation details.
- Final runtime versions for .NET, EF Core, and Npgsql.

## 2.1) Critical analysis summary

### Strengths in the current outline

- Clear code-first intent and strong alignment to EF migration workflows.
- Correct placement direction toward Common module as a cross-cutting concern.
- Good emphasis on deterministic migration diffing and provider-aware execution.
- Proper separation from domain business rules.

### Critical gaps and required hardening

| Gap | Risk if unchanged | Design hardening decision |
|---|---|---|
| No formal failure taxonomy | Inconsistent error handling across model, differ, and provider layers | Introduce extension-specific exception hierarchy with boundary handling rules |
| No observability contract | Low diagnosability of migration and runtime authorization failures | Define structured logging taxonomy, event naming, redaction, and correlation strategy |
| SOLID implied but not explicit | Interface sprawl or tight coupling as extension grows | Add SOLID conformance controls and anti-pattern guardrails |
| Operational rollback semantics not explicit | Difficult incident response during policy regressions | Define explicit rollback expectations and failure containment behavior |
| Configuration drift controls are broad but not scoped to failure modes | Hard to triage policy drift root causes | Add drift classification and diagnostics requirements |

### Design-only rule for this document

- This document defines architecture, contracts, and governance only.
- No implementation code is included by design.

## 2.2) Conformance with repository architecture principles

This design explicitly conforms to the architecture constraints in README:

- Clean Architecture boundary: policy and persistence concerns remain in Common.Infrastructure.
- Domain purity: feature-module Domain and Application business logic stay free of RLS persistence implementation.
- Authorization model: RLS is a database-level enforcement layer that complements endpoint and optional use-case authorization.
- Role consistency: role identifiers are centralized and consumed as contracts; no distributed hardcoded role strings.
- Module ownership: schema-per-module and module-owned migration streams are preserved.
- Logging and security hygiene: structured logs, correlation context, and sensitive-data redaction are mandatory.

## 3) Clean Architecture placement in Common module

The extension should live in the Common module because it is cross-cutting and persistence-focused, while business rules remain in feature modules.

Proposed structure improvements:

```text
API/src/common/
├── Application/
│   └── Authorization/
│       └── Contracts/
│           ├── ICurrentRequestSecurityContext.cs
│           └── SecurityContextValueObjects.cs
├── Domain/
│   └── (no RLS implementation details)
├── Infrastructure/
│   └── Persistence/
│       └── Rls/
│           ├── Abstractions/
│           │   ├── IRlsPolicyDefinition.cs
│           │   ├── IRlsPolicyRegistry.cs
│           │   └── IRlsProviderCapabilities.cs
│           ├── Configuration/
│           │   ├── RlsEntityTypeBuilderExtensions.cs
│           │   ├── RlsModelBuilderExtensions.cs
│           │   └── RlsPolicyBuilder.cs
│           ├── Metadata/
│           │   ├── RlsAnnotationNames.cs
│           │   ├── RlsModelAnnotations.cs
│           │   └── RlsAnnotationCodeGenerator.cs
│           ├── Migrations/
│           │   ├── Operations/
│           │   │   ├── CreateRlsPolicyOperation.cs
│           │   │   ├── AlterRlsPolicyOperation.cs
│           │   │   ├── DropRlsPolicyOperation.cs
│           │   │   ├── EnableRowLevelSecurityOperation.cs
│           │   │   └── DisableRowLevelSecurityOperation.cs
│           │   ├── RlsMigrationBuilderExtensions.cs
│           │   ├── RlsMigrationsModelDiffer.cs
│           │   └── RlsCSharpMigrationOperationGenerator.cs
│           ├── Providers/
│           │   └── PostgreSql/
│           │       ├── PostgreSqlRlsCapabilities.cs
│           │       ├── PostgreSqlRlsSqlFormatter.cs
│           │       └── PostgreSqlRlsMigrationsSqlGenerator.cs
│           └── Runtime/
│               ├── IRlsSessionContextApplier.cs
│               ├── RlsSessionContextApplier.cs
│               └── RlsDbConnectionInterceptor.cs
└── Presentation/
    └── (no RLS persistence implementation)
```

Placement rules:

- Feature module Domain layers do not contain RLS logic.
- Feature module Infrastructure entity configurations call the Common RLS fluent API.
- Common Application contracts expose only request context abstractions.
- Provider-specific SQL generation remains inside Common Infrastructure Providers.

## 4) Requirement-to-design mapping

| Requirement | Design response |
|---|---|
| Code-first purity | Fluent API and migration operations; no hand-written migration SQL in app code |
| Strong typing | Policy definition objects, enums, and typed predicate nodes |
| Model integration | Policies stored as EF annotations tied to entity/table metadata |
| Snapshot awareness | Custom annotation code generation emits policy config into snapshot |
| Automated diffing | Custom model differ computes create/alter/drop operations |
| First-class operations | Custom migration operations and migration builder extensions |
| Provider-aware execution | Provider adapters for SQL generation (PostgreSQL first) |
| Deterministic behavior | Canonical serialization, stable ordering, idempotent SQL generation |
| Separation of concerns | Policy configuration in Infrastructure; domain model remains clean |
| Maintainability/extensibility | Provider abstraction layer and modular extension internals |

## 4.1) SOLID conformance model

This extension follows SOLID using explicit controls:

- Single Responsibility Principle (SRP):
    - Metadata modeling, migration differ, provider SQL translation, and runtime session-context application are separated into distinct components.
    - Exception mapping and logging enrichment are cross-cutting services, not embedded in domain logic.
- Open/Closed Principle (OCP):
    - New providers are introduced through provider capability and SQL generation abstractions without changing core policy metadata contracts.
    - New policy features are added through descriptor extensions and canonicalization rules, avoiding invasive changes to existing consumers.
- Liskov Substitution Principle (LSP):
    - Provider adapters must preserve operation semantics (create, alter, drop, enable, disable) and deterministic behavior guarantees.
- Interface Segregation Principle (ISP):
    - Fine-grained interfaces are preferred (policy registry, provider capability, session context applier) to avoid broad multipurpose abstractions.
- Dependency Inversion Principle (DIP):
    - High-level migration and metadata workflows depend on abstractions, while provider/runtime specifics are injected behind contracts.

Anti-patterns explicitly avoided:

- Putting RLS logic into entity/domain types.
- Embedding provider-specific details into generic metadata models.
- Catch-all exception swallowing inside migration or provider layers.
- Logging sensitive claim/session values in clear text.

## 5) Public configuration API (generic, strongly typed)

The API should feel native to EF configuration patterns.

### 5.1 Configuration contracts (design-level)

- Entity-level configuration contract:
    - Attach RLS configuration to a mapped entity/table.
    - Enable or disable RLS at table scope.
    - Register one or more named policies with command scope, role scope, and predicates.
- Global defaults contract:
    - Configure naming conventions, default policy mode, and deterministic canonicalization options.
- Validation contract:
    - Reject duplicate policy identities and unsupported command/mode combinations during model build.

### 5.2 Configuration object model

The public configuration model should be expressed through typed descriptors (not raw ad hoc strings):

- Policy descriptor identity (schema, table, policy name).
- Command scope descriptor.
- Role-scope descriptor bound to role catalog contracts.
- Predicate descriptor for USING and WITH CHECK semantics.
- Mode descriptor for permissive/restrictive behavior.

## 6) Core policy model

Core strongly typed definitions:

- RlsPolicyId: deterministic identity composed of schema, table, and policy name.
- RlsPolicyDefinition: commands, roles, using predicate, check predicate, mode.
- RlsCommand enum: Select, Insert, Update, Delete, All.
- RlsPolicyMode enum: Permissive, Restrictive.
- RlsRole: role catalog reference, no scattered hardcoded strings.
- RlsPredicate node model: typed predicate graph (column, session value, binary combinators).

Design notes:

- Keep a constrained predicate model for deterministic translation.
- Provide explicit provider extension hooks for advanced provider-specific expressions.
- Canonicalize policy objects before annotation storage.

## 7) EF model metadata integration and snapshot awareness

### 7.1 Metadata storage

Policies are attached to entity/table metadata and persisted via annotations.

- Annotation key pattern: `Common:Rls:*`
- Entity/table annotation includes canonical serialized policy set.
- Table-level RLS enabled/disabled state stored separately.

### 7.2 Snapshot generation

Implement custom annotation code generation to ensure policies appear in model snapshots as fluent calls, not opaque blobs.

Expected snapshot behavior:

- Policy registration is emitted in deterministic order.
- Rebuilding model from snapshot recreates equivalent policy metadata.
- No policy drift between snapshot and runtime model.

### 7.3 Deterministic canonicalization

Canonicalization rules:

- Stable sort: schema, table, policy name.
- Stable sort command sets and role sets.
- Normalized predicate node ordering for commutative operators.
- Normalized identifier casing strategy documented once and applied consistently.

## 8) Migration diffing and first-class operations

### 8.1 Migration operations

Custom migration operations:

- EnableRowLevelSecurityOperation
- DisableRowLevelSecurityOperation
- CreateRlsPolicyOperation
- AlterRlsPolicyOperation
- DropRlsPolicyOperation

### 8.2 Diff algorithm

Model differ compares current model metadata against snapshot metadata.

- Added policy -> Create operation.
- Removed policy -> Drop operation.
- Changed policy definition -> Alter operation.
- RLS toggle changed -> Enable/Disable operation.

Diff rules:

- Only semantic changes produce operations.
- Stable operation ordering by schema/table/policy.
- No duplicate or noisy operations.

### 8.3 Migration code generation

Migrations should show intent as explicit, first-class RLS operations.

Design requirements:

- Generated migration artifacts clearly represent table RLS enable/disable and policy create/alter/drop intent.
- Application migrations remain code-first and readable.
- Developers do not hand-write SQL for policy lifecycle management.
- Generated operations are deterministic and stable across equivalent model states.

## 9) PostgreSQL provider execution strategy

The PostgreSQL provider adapter translates custom operations into SQL commands.

Provider responsibilities:

- Quote identifiers using EF SQL helpers.
- Generate `ALTER TABLE ... ENABLE/DISABLE ROW LEVEL SECURITY`.
- Generate `CREATE POLICY`, `ALTER POLICY`, and `DROP POLICY` statements from typed definitions.
- Emit deterministic SQL text for equivalent models.

Idempotency approach:

- Use safe existence checks where needed for script generation scenarios.
- Keep operation ordering and transaction boundaries deterministic.
- Ensure repeated migration application does not create unintended side effects.

## 10) Runtime identity-to-session context contract

RLS relies on request context values being available in PostgreSQL session scope.

Define a runtime contract, independent from final auth provider choice:

- ICurrentRequestSecurityContext (Application contract): exposes user id, roles, tenant/workspace identifiers when applicable.
- IRlsSessionContextApplier (Infrastructure): applies request context to connection/transaction scope.
- Optional DbConnection interceptor implementation for centralized setup.

Guidelines:

- Apply context at transaction scope when possible.
- Clear or reset context safely on pooled connections.
- Never trust client-sent DB session variables directly.

## 10.1) Exception strategy (extension-specific)

### Goals

- Provide consistent, diagnosable failure semantics across metadata, snapshot, migrations, provider translation, and runtime context application.
- Support deterministic handling and clean propagation at architecture boundaries.

### Exception hierarchy

Base exception:

- RlsExtensionException

Specialized exceptions:

- RlsConfigurationException
    - Raised for invalid policy definitions, duplicate policy identities, or invalid descriptor combinations.
- RlsModelMetadataException
    - Raised when model annotations are missing, malformed, or inconsistent.
- RlsSnapshotConsistencyException
    - Raised when snapshot round-trip cannot restore equivalent policy metadata.
- RlsMigrationDiffException
    - Raised when differ semantics fail determinism or produce invalid operation graphs.
- RlsProviderNotSupportedException
    - Raised when requested features are not supported by the active provider capabilities.
- RlsSqlGenerationException
    - Raised when provider SQL translation fails for valid operations.
- RlsSessionContextException
    - Raised when runtime security context cannot be applied/reset safely.
- RlsIdempotencyException
    - Raised when operation re-application safety guarantees are violated.

### Handling boundaries

- Model build boundary:
    - Fail fast on configuration and metadata exceptions.
- Migration scaffolding boundary:
    - Surface deterministic diff/snapshot exceptions with actionable diagnostics.
- Migration execution boundary:
    - Wrap provider-specific translation/execution failures in extension exceptions while preserving inner context.
- Runtime boundary:
    - Distinguish transient context-application failures from structural misconfiguration failures.

### Exception design constraints

- Every exception includes stable error code/category metadata for logging and diagnostics.
- Exception messages must avoid leaking secrets, raw tokens, or sensitive claim payloads.
- Exceptions must map to documented remediation guidance in operational runbooks.

## 10.2) Logging and observability strategy

### Logging principles

- Use structured logging with stable event names and properties.
- Include correlation identifiers and request/module context for cross-layer tracing.
- Redact sensitive values (tokens, secret keys, raw claim payloads, session variable values).
- Keep logs deterministic and actionable; avoid noisy duplicate events.

### Event categories

- RLS.Configuration
    - Policy registration, validation outcomes, and configuration rejection events.
- RLS.Metadata
    - Annotation serialization/canonicalization diagnostics.
- RLS.Snapshot
    - Snapshot emission and round-trip validation outcomes.
- RLS.MigrationDiff
    - Detected create/alter/drop/enable/disable decisions.
- RLS.Provider.PostgreSql
    - Provider capability checks and SQL generation lifecycle events.
- RLS.RuntimeContext
    - Session context apply/reset success and failure telemetry.

### Severity guidance

- Debug:
    - Canonicalization traces and internal decision diagnostics for local troubleshooting.
- Information:
    - Successful policy operation generation and execution milestones.
- Warning:
    - Recoverable mismatches, capability fallbacks, and non-fatal idempotency guards.
- Error:
    - Operation failures that block migration generation or execution.
- Critical:
    - Security-impacting inconsistencies (for example, failed enforcement context setup with no safe fallback).

### Metrics and tracing

- Counters:
    - Policies added/changed/removed per migration generation cycle.
    - Provider translation failures by category.
- Histograms:
    - Diffing and SQL generation durations.
- Traces:
    - Correlated spans for model build, differ, SQL generation, and execution boundaries.

### Operational diagnostics requirements

- Emit one summary event per migration generation cycle with deterministic counts.
- Emit one summary event per migration execution cycle with success/failure status.
- Ensure failure events include extension error category, provider name, and schema/table/policy identity (non-sensitive only).

## 11) Determinism, transparency, and idempotency guarantees

Guarantee targets:

- Equal models produce equal snapshot output.
- Equal model and snapshot produce no-op diffs.
- Migration files show explicit policy intent.
- Replaying deployment scripts does not produce duplicate policy effects.

Mechanisms:

- Canonical model representation.
- Stable differ ordering.
- Explicit operations and provider SQL formatting rules.
- Automated regression tests for generated migration text and SQL commands.

## 12) Testing strategy and quality gates

### 12.1 Test matrix

- Unit tests: policy builder validation and predicate canonicalization.
- Snapshot tests: model to snapshot to model round-trip equivalence.
- Differ tests: add/modify/remove policy scenarios.
- SQL generator tests: PostgreSQL SQL text for each operation.
- Integration tests: local PostgreSQL RLS enforcement with session context setup.

### 12.2 Quality gates

- Build and analyzer clean (StyleCop and Roslyn expectations).
- No manual SQL in application migrations for RLS lifecycle management.
- Deterministic migration generation checks in CI.
- RLS policy behavior integration checks for protected entities.
- Extension exception taxonomy coverage in tests (classification and boundary mapping).
- Structured logging contract checks (event naming, required properties, redaction rules).

## 13) Implementation phases

1. Foundation: policy model, builder API, annotation model.
2. Snapshot integration: annotation code generation and round-trip tests.
3. Migration operations: operations, differ, migration builder extensions.
4. PostgreSQL provider: SQL generator and provider capability checks.
5. Runtime session context: request context contract and context applier.
6. Failure and observability hardening: extension exceptions and structured logging contracts.
7. Validation: deterministic diff tests, integration RLS tests, and operational diagnostics tests.

## 14) Open decisions (tracked as TBD)

- Final .NET, EF Core, and Npgsql versions.
- Final role catalog naming conventions.
- Final schema naming conventions per module.
- Final claim mapping strategy from auth provider to request security context.
- Final release migration execution model (pipeline/manual/startup gate).
- Final logging backend selection and event retention policy.

## 15) Definition of done for this extension

The extension is considered complete when:

- Policy definitions are authored only through typed code APIs.
- Policies are visible in the EF model snapshot and round-trip correctly.
- Migration scaffolding emits explicit policy operations for create/alter/drop.
- PostgreSQL SQL generation executes policy operations reliably.
- Deterministic and idempotent behavior is demonstrated by automated tests.
- Feature modules can adopt the extension without adding persistence concerns into domain models.
